---
title: 一条日志容量指标，方案改了八版：自研平台接监控指标的踩坑实录
date: 2026-11-26 21:30:00
tags:
  - Prometheus
  - 监控
  - 运维
  - Go
  - 踩坑
categories:
  - 技术
  - 监控运维
---

## TL;DR

需求看起来只有一句话：**"journald 日志池容量快满的时候告个警，前端能看一眼水位。"**

结果方案改了 **8 版**（v1.0 → v2.5），每版 review 都被拦下新问题，累计沉淀了几十条"错误写法 vs 正确写法"。事后复盘，坑集中在四个地方：

1. **口径**：统计磁盘占用用了逻辑大小而不是块分配，凭空高估 11%；
2. **语义**：预算解析失败时用推导值兜底，导致 3.8GiB 就误报 critical（真实预算 10G）；无数据输出 0，把"不知道"伪装成"健康"；
3. **规则**：PromQL 里用不存在的模板函数让整组规则挂掉；健康判断用 `max` 聚合被健康副本掩盖；
4. **部署**：宿主机 curl 不到 Prometheus（端口没发布）；升级脚本 `cp -rn` 铺不开新配置；回滚不重启容器、不 `exit 1`。

一句话总结：

> **监控系统的每一个"不知道"，都必须显式地表达为"不知道"，永远不能被伪装成 0、被猜测值兜底、或者被健康副本掩盖。**

<!-- more -->

## 一、需求与链路：先想清楚谁干什么

要接入的指标（journald 日志池容量）：

| 指标 | 类型 | 说明 |
| --- | --- | --- |
| `example_log_journald_used_bytes` | gauge | 池实际占用（**块分配**字节，见第二节） |
| `example_log_journald_budget_bytes` | gauge | 池预算（宿主 `SystemMaxUse=10G`） |
| `example_log_journald_ratio` | gauge | used / budget |
| `example_log_journald_probe_timestamp_seconds` | gauge | 采样心跳 |
| `example_log_capacity_config_ok` | gauge | 1 = 采集链路全成功，0 = 有缺失（带 reason） |

告警：池水位 80% warning / 95% critical、文件系统余量不足、**采样失联**、**配置不完整**、应用日志日产量。

动手前先把链路画出来，并给每一层定死职责：

```
① BFF 埋点/采样 ──► ② 公开 /metrics/<name> 端点
                          │  Prometheus 定期抓取
                          ▼
        ③ scrape job ──► ④ 告警规则（PromQL 判定 + for 防抖）
                          │  推送到 Alertmanager
                          ▼
        ⑤ Alertmanager（聚合/去重/抑制）
                          │  只读查询 /api/v2/alerts
                          ▼
        ⑥ BFF 只读代理/DTO ──► ⑦ 前端（告警中心复用 或 新卡片）
```

关键认知：**Prometheus 只负责判定，Alertmanager 只负责聚合，BFF 只做埋点 + 只读代理，前端只做渲染。** 四层各司其职，一旦混了（比如在 BFF 里写告警判定逻辑，或在规则里做"渲染适配"），每层的语义都会开始互相污染——后面几乎每个坑都和"职责越界"有关。

## 二、口径坑：360 MiB 还是 324 MiB？

BFF 挂载 `/var/log/journal` 只读目录，60s 采样一次。第一版遍历目录、把所有文件大小加起来——结果和权威值对不上：

| 口径 | 结果 |
| --- | --- |
| `journalctl --disk-usage`（权威） | **324 MiB** |
| 文件逻辑大小合计（`Info().Size()`） | 360 MiB（**高估 11%**） |
| 块分配合计（`Stat_t.Blocks × 512`） | 332 MiB |
| 块分配、排除版本目录 | **324 MiB**（与 journalctl 一致） |

两个原因：

1. **journald 预分配 journal 文件**（初始 8MiB），逻辑大小把未写入的预留区也算进去了。要看"实际占了多少磁盘"，必须用块分配：`st.Blocks` 恒以 512B 为单位。
2. journal 目录下有个 `<machine-id>.--version/` 格式版本目录，`journalctl` 不计入。过滤它时还顺手踩了一个 Go 的经典坑：

```go
// ❌ 祖先目录名含 ".--version" 时会误伤
if strings.Contains(path, ".--version") { return filepath.SkipDir }

// ✅ 只判断目录名本身，且是后缀
if strings.HasSuffix(d.Name(), ".--version") { return filepath.SkipDir }
```

这次还立了一条验收纪律：**有物理量对照时必须写口径校验脚本**，api 值 vs `journalctl --disk-usage`，容差 1MiB 或 2%，不通过不交付。没有这个对照，11% 的高估会一直藏在告警阈值里。

顺带一提，验收脚本本身也踩了两个环境坑：systemd 255 的 `--disk-usage` **不接受参数**（`--disk-usage=short` 直接报错）；`numfmt` 不认 `"332.0MB"`（**不能带 B**）。工具命令永远以目标机实测为准。

## 三、语义坑一：fail-closed，不要用猜测值兜底

预算（`SystemMaxUse=10G`）通过部署脚本解析后经环境变量注入 BFF。早期版本想"更健壮"：预算缺失时推导一个 `min(10% 文件系统, 4GiB)` 兜底。

后果：宿主机明明配了 10G，如果注入链路断了，BFF 按 4GiB 算——日志用到 3.8GiB 就误报 critical。**与真实配置不一致的猜测值，比没有值更危险**：没有值顶多不出告警，猜测值会出**错误的告警**。

最终语义（fail-closed）：

| 场景 | 发布的序列 | `config_ok` | API |
| --- | --- | --- | --- |
| 全部成功 | used/budget/ratio/probe 全出 | 1 | 正常 |
| 预算缺失/非法 | **只出 used/probe，不出 budget/ratio** | 0 | `reason=budget_unavailable`，`level=unknown` |
| 目录不可读 | 不出任何池序列 | 0 | `pool` 省略（`null`，不是 0） |
| 功能关闭 | 什么都不出（含 `config_ok`） | 不发布 | `reason=capacity_disabled` |

配套还有一条 Go/Viper 的坑：**`viper.AutomaticEnv()` 对 `Unmarshal` 到结构体不可见**。配置文件里没有 `log.journald` 段时，`EXAMPLE_LOG_JOURNALD_*` 环境变量根本进不了结构体。必须显式 `SetDefault` + `BindEnv`（且放在 `AutomaticEnv()` 之后），并补一条"配置文件无该段、只给 env"的回归测试。

## 四、语义坑二：无数据 ≠ 0

这是整个方案里被 review 揪得最狠的一条，前后犯了三个变体：

**变体 1：采样失败时输出 0。**

```go
fmt.Fprintf(b, "example_log_journald_ratio 0\n")  // 读成"容量归零，健康"
```

正确做法是**无数据不出序列**，调用方按"序列缺失"判定。0 是一个合法的观测值，把"不知道"写成 0，等于给监控系统喂假数据。

**变体 2：一个总开关吞掉所有字段。** 预算解析失败时，把 `used`（明明采集到了）也一起藏起来。改成**逐字段独立成功位**：

```go
type PoolSample struct {
    BaseOK   bool // 目录读取成功 → used + probe_timestamp
    BudgetOK bool // 预算解析成功 → budget + ratio
    FSOK     bool // Statfs 成功   → filesystem_avail
}
if smp.BaseOK { s.publishUsed(...) }
if smp.BaseOK && smp.BudgetOK { s.publishRatio(...) }  // ratio 两个前提缺一不可
```

**变体 3：API 把空值返回成 0。** 查 Prometheus 日产量时，`err != nil` 或**返回空向量**都返回 `null + reason="no_data"`——空向量不等于 0 字节/天。

同一条纪律延伸到前端渲染契约：`level` 枚举必须有 **`unknown` 档**（预算缺失判不了档时，禁止落成 `ok` 渲染成绿色）；`stale=true` 时 `level` 不可信，必须灰显"数据已过期"，不得按历史档位上色。

## 五、规则坑：四条让告警"静默失效"的写法

规则文件是自研平台渲染 Prometheus 配置的一部分，写错轻则单条失效，重则整组加载失败。

**坑 1：用不存在的模板函数。**

```yaml
# ❌ Prometheus 模板函数表里没有 default，整组规则解析失败
summary: "采样失联（{{ $labels.instance | default \"全部实例\" }}）"

# ✅ Go 模板 if/else
summary: '采样失联{{ if $labels.instance }}（{{ $labels.instance }}）{{ else }}（全部实例）{{ end }}'
```

**坑 2：健康判断用 `max` 聚合，被健康副本掩盖。** 多副本部署时，只要有一个副本的采样新鲜，`max` 就取到新鲜值，另一个副本**冻结了也不会告警**：

```yaml
# ❌
expr: time() - max by (scope) (example_log_journald_probe_timestamp_seconds) > 300

# ✅ 健康类逐实例判断，不带聚合
expr: (time() - example_log_journald_probe_timestamp_seconds) > 300
```

反过来，**共享池水位才用 `max by (scope)`**——各副本读到的是同一个池，`max` 只是去重。两者语义相反：健康要看最差实例，水位才做去重。

**坑 3：没有 `absent()` 兜底。** 上面两条都依赖"序列存在"。如果 scrape job 被误删或端点 404，序列整体 stale，`up==0` 和 `time()-probe>300` **都没有序列可评估，永远不会触发**。失联告警必须三条件并列：

```yaml
expr: |
  up{job="bff_log"} == 0
  or
  absent(example_log_journald_probe_timestamp_seconds)
  or
  (time() - example_log_journald_probe_timestamp_seconds) > 300
```

**坑 4：规则读 gauge 当阈值。** `daily_bytes > daily_budget_bytes` 看着优雅，但阈值 gauge 缺失时比较结果不确定。阈值写死字面量（`> 536870912`），调阈值走发布——监控规则要的是确定性，不是灵活性。

另外两条通用纪律：容量类 gauge 全部加 `probe_timestamp` **新鲜度守卫**（否则采样冻结时按旧值"永久告警"或"永久漏报"）；counter 告警只用 `increase()/rate()`，多副本 `sum by (job)`——**不要**在进程里自己维护"24 小时滚动桶"，重启归零、多副本统计不全，两个 bug 直接焊死在代码里。

## 六、部署坑：环境永远和你想的不一样

**坑 1：宿主机 curl 不到 Prometheus。** 生成的容器编排只发布 BFF 的 8000 端口，Prometheus(9090) / Alertmanager(9093) 都没有宿主端口——所有 `curl http://127.0.0.1:9090/...` 必定失败，而早期升级脚本里全是它。校验一律经 BFF 容器内的 BusyBox `wget` 发起：

```bash
compose=(docker compose --env-file "$ROOT/.env" -f "$ROOT/output/docker-compose.yml")
"${compose[@]}" exec -T example wget -qO- --post-data='' http://prometheus:9090/-/reload
"${compose[@]}" exec -T example wget -qO- 'http://prometheus:9090/api/v1/rules?type=alert' | jq ...
```

**坑 2：镜像里没有 promtool。** 想当然地 `docker compose exec prometheus promtool check rules`——运行镜像是无 shell 的精简镜像，`sh` 都没有。改用 reload 指标 + API 校验：`prometheus_config_last_reload_successful == 1` 且新规则组存在。教训：**不要假设镜像里有二进制**；规则静态校验应该前移到 CI（单独的工具镜像），部署后的 reload 指标只是兜底。

**坑 3：升级脚本 `cp -rn` 铺不开新配置。** `cp -rn` 只补缺不覆盖，老部署拿不到新 scrape job 和新规则文件，而且没人报错。`upgrade.sh` 必须受控下发，回滚要完整：

```bash
rollback() {
  # 1) 恢复备份的 prometheus.yml + 每个被覆盖的规则文件
  # 2) 删除"本次新增"的规则文件（记录 NEW_RULES；只恢复旧文件，坏的新文件还在目录里）
  # 3) 重启 prometheus 容器——只 reload 不够，坏配置可能已让进程进入失败重启循环
  "${compose[@]}" restart prometheus
  exit 1   # ❌ 常见错误：err "升级失败" 不退出，脚本继续打印"升级完成"
}
```

细节里还有一枪：**函数定义必须写在第一次调用之前**。bash 的函数是运行到才解析的，`reload_prom || rollback` 写在 `rollback()` 定义前面，回滚路径就是个未定义命令。

**坑 4：`${VAR:?}` 强制插值拦住卸载脚本。** compose 里写 `${LOG_JOURNALD_BUDGET_BYTES:?}`，所有**没带 `--env-file`** 的 compose 调用（ps/uninstall/backup）都会直接失败——卸载不应该被一个日志预算拦住。改成 `:-0`，把拦截放到部署前置检查和应用侧 fail-closed（第三节）。

## 七、权限坑：看起来有守卫，实际谁都能进

容量结构化 API 挂权限时，用了平台既有的 `RequirePermission("telemetry.overview")` 中间件。review 时发现它是 **fail-open** 的：守卫按 code 查权限表，查不到 key 时只 warn 放行——键名写错（或没注册）时，"守卫"静默变成摆设。

修复：测试里显式断言权限键已注册：

```go
if LookupCode("telemetry.overview.view") == "" {
    t.Fatal("权限键未注册，守卫会 fail-open")
}
```

这类陷阱的共性：**它永远不会在功能测试里失败**（正常用户照样 200），只有"无权限用户也能 200"时才暴露。安全相关的测试用例（401 / 403 / 200 / superadmin）必须像功能用例一样显式写出来。

## 八、沉淀：把 8 版踩坑变成一页 runbook

方案定稿后，把这些经验整理成一份通用的《指标接入开发流程与改动清单》，把"新增一条指标 + 告警 + 前端展示"拆成 **Step 0–10**：

```
Step 0 定类型与命名（counter/gauge、后缀、低基数标签、冻结契约）
Step 1 埋点/采样（ctx 超时、部分成功位、超限降频）
Step 2 暴露 scrape 端点（公开区、不挂 JWT、手写文本）
Step 3 配置注册（SetDefault + BindEnv、fail-closed）
Step 4 scrape job（HA 用 dns_sd，静态单目标会跳 instance）
Step 5 告警规则（模板白名单、absent 兜底、最差实例、新鲜度守卫、字面量阈值）
Step 6 Alertmanager（通常不改）
Step 7 BFF 代理/结构化 API（空值契约、unknown 档、权限守卫）
Step 8 契约（openapi + 模块文档）
Step 9 前端（告警中心零改动 或 新卡片）
Step 10 部署接线 + 验收（受控下发、容器内校验、HA 反差演练）
```

外加一条**红线 checklist**（节选）：

- [ ] 无数据不出 0 序列（`omitempty` + `null`，`level` 有 `unknown` 档）
- [ ] 失联规则三条件：`up==0 or absent(...) or probe>300`
- [ ] 健康用最差实例（不聚合），共享池水位才 `max by (scope)`
- [ ] 预算/阈值 fail-closed，不猜测兜底
- [ ] 规则模板不用 `default`，用 `{{ if }}…{{ else }}…{{ end }}`
- [ ] 升级脚本：备份规则、删新增、重启 prometheus、`exit 1`
- [ ] 校验走容器、用 reload 指标，别信 promtool 和宿主端口

runbook 最有价值的部分不是流程，而是每个坑都留了"❌ 错误写法 / ✅ 正确写法"的对照——下次再接新指标，先过一遍 checklist，八成的问题在写代码前就被拦下了。

## 九、复盘与面试口径

| 主题 | 一句话口径 |
| --- | --- |
| 监控数据语义 | "不知道"必须显式表达：无数据不出 0 序列、判不了档给 `unknown`、预算缺失 fail-closed，永远不用猜测值伪装成观测值。 |
| 指标口径 | 磁盘占用用块分配（`Stat_t.Blocks × 512`），不用逻辑大小；有权威物理量时写对照验收脚本（容差 1MiB/2%）。 |
| 健康告警 | 健康类规则用最差实例、逐实例判断，禁止 `max` 聚合（会被健康副本掩盖）；序列整体消失只能靠 `absent()` 兜底。 |
| 告警阈值 | 阈值写死字面量，不读 gauge（缺失时比较结果不确定）；counter 只用 `increase()/rate()`。 |
| 部署 | 升级必须受控下发（`cp -rn` 铺不开新配置）；回滚要删新增文件、重启容器、`exit 1`；校验走容器网络，不假设镜像里有工具。 |
| 部分成功 | 逐字段独立成功位，不要一个总开关吞掉可用字段；`ratio` 的两个前提缺一不可。 |
| fail-open | 权限守卫查不到键只 warn 放行是静默失效；测试必须断言权限键已注册 + 401/403 用例。 |

## 参考

- [Prometheus — ALERTING RULES（模板函数表）](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus — Recording rules 与告警设计](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Alertmanager — Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [systemd — journald `SystemMaxUse`](https://www.freedesktop.org/software/systemd/man/latest/systemd-journald.service.html)
