---
title: 服务明明活着，压测却报 ConnectException：一次 JDK HttpClient 并发首连竞态排查
date: 2026-09-23 02:51:47
tags:
  - Java
  - HttpClient
  - 并发
  - 问题排查
categories:
  - 技术
  - Java
---

## TL;DR

本地做一次 200 并发的下单压测，服务端 Tomcat 正常启动、健康检查 200、日志里甚至能看到请求已经打到 `DispatcherServlet`，但测试仍偶发抛 `java.net.ConnectException`，根因是 `java.nio.channels.ClosedChannelException`。

排查下来，问题不在服务端，而在**压测客户端 JDK `HttpClient` 的连接建立/重试路径**：200 个线程同时"首连"时，`PlainHttpConnection` 的「失败重试」会复用同一个 `SocketChannel`，并发下存在竞态窗口，`connect` 在已关闭的通道上执行就抛 `ClosedChannelException`。叠加 `localhost` 的 IPv4/IPv6 双栈回退后更容易触发。

修复两步：请求地址改用 `127.0.0.1` 消除地址族歧义；在接口幂等的前提下，对传输层 `IOException` 做有限重试。

<!-- more -->

## 一、背景

项目是一个交易下单服务（个人作品集），核心链路是「幂等 + Redis/DB 库存防超卖 + Outbox 最终一致」。为了验证并发正确性，写了一个本地集成测试 `OrderConcurrencyLocalIT`：

- `@SpringBootTest(webEnvironment = RANDOM_PORT)`，连真实 MySQL 3307 与 Redis 6379；
- 200 个线程用 `CountDownLatch` 同时发令，共用一个 `HttpClient` 打 `POST /api/orders`；
- 两个用例：不同幂等键（验证不超卖，成功数 == 库存）、同一幂等键（验证只落 1 单）。

环境：Windows 11、JDK 17.0.12。

## 二、现象：服务活着，请求却"连不上"

压测偶发失败，异常链如下（已精简）：

```text
java.util.concurrent.ExecutionException: java.net.ConnectException
    at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:205)
    at ...OrderConcurrencyLocalIT.fire(OrderConcurrencyLocalIT.java:157)
    at ...OrderConcurrencyLocalIT.shouldCreateOnlyOneOrderForSameKey(OrderConcurrencyLocalIT.java:116)
Caused by: java.net.ConnectException
    at java.net.http/jdk.internal.net.http.HttpClientImpl.send(HttpClientImpl.java:574)
    at ...OrderConcurrencyLocalIT.call(OrderConcurrencyLocalIT.java:175)
Caused by: java.net.ConnectException
    at java.net.http/jdk.internal.net.http.common.Utils.toConnectException(Utils.java:1055)
    at java.net.http/jdk.internal.net.http.PlainHttpConnection.connectAsync(PlainHttpConnection.java:198)
    at java.net.http/jdk.internal.net.http.PlainHttpConnection.checkRetryConnect(PlainHttpConnection.java:230)
Caused by: java.nio.channels.ClosedChannelException
    at java.base/sun.nio.ch.SocketChannelImpl.ensureOpen(SocketChannelImpl.java:195)
    at java.base/sun.nio.ch.SocketChannelImpl.beginConnect(SocketChannelImpl.java:760)
    at java.net.http/jdk.internal.net.http.PlainHttpConnection.lambda$connectAsync$0(PlainHttpConnection.java:183)
```

三个关键点：

1. 栈里出现 `PlainHttpConnection.connectAsync → checkRetryConnect`，说明失败发生在 **TCP 连接建立阶段**，请求还没进入 HTTP 语义；
2. 最底层是 `ClosedChannelException`，语义是「在已关闭的 channel 上做 I/O」；
3. 反直觉的是：服务端日志里 `DispatcherServlet` 已经被 worker 线程初始化（`Initializing Spring DispatcherServlet 'dispatcherServlet'`），也就是说**有请求真的到达了服务端**，服务本身没挂。

## 三、排查过程

**假设一：服务没起来？** 否。Tomcat 正常 `Started ... on port xxxx`，`/api/health` 返回 200，且请求已触发 `DispatcherServlet` 懒加载初始化。

**假设二：服务端过载、accept 队列溢出？** 存疑。同样 200 并发的另一个用例（不同幂等键）稳定通过；如果真是服务端接纳能力问题，两个用例应该一起挂，而不是只挂其中一个。

**缩小到客户端。** 异常发生在 `connect` 阶段，与服务端业务无关。先把 `localhost` 换成 `127.0.0.1`，现象明显减轻但仍偶发——说明还叠加了并发下的连接竞态。

**读 OpenJDK 源码。** `PlainHttpConnection.connectAsync` 在连接失败时会走到 `checkRetryConnect`：

- `canRetryConnect` 判定「可重试」的条件是：异常是 `ConnectException`、`unsuccessfulAttempts == 0`、且连接超时定时器尚未到期；
- 一旦判定重试，`checkRetryConnect` 会**用同一个 `SocketChannel` 再调一次 `connectAsync`**；
- 而连接失败分支会调用 `close()` 关闭通道。

也就是说，存在这样一条路径：第一次连接失败 → 关闭通道 → 触发重试 → 在**已关闭的通道**上再次 `connect` → `ClosedChannelException`。源码注释还专门解释了这段重试的来由：

> On some platforms, a ConnectEvent may be raised and a ConnectionException may occur with the message "Connection timed out: no further information" before our actual connection timeout has expired. In this case ... we will retry once again.

这是为某些平台（含 Windows）的"连接超时提示不准"做的补偿逻辑。单请求时它无害；但 200 个线程同时首连时，这条「重试 + 关闭」路径就有了竞态窗口。

## 四、为什么会是 ClosedChannelException

把机制拼起来看：

- **默认协议是 HTTP/2。** `HttpClient.version()` 的默认值是 `HttpClient.Version.HTTP_2`（官方 javadoc 明确写了）。明文（`http://`）场景下客户端会尝试 h2c 升级，连接建立流程比纯 HTTP/1.1 更复杂。
- **连接池默认无上限。** `jdk.httpclient.connectionPoolSize` 默认为 `0`，含义是「不限制 keep-alive 缓存中的连接数」。200 并发会同时建立大量连接，放大竞态窗口。
- **`localhost` 双栈。** 本机 `localhost` 解析顺序是 `::1` 在前、`127.0.0.1` 在后，而 Tomcat 只监听 IPv4 通配地址。地址族不匹配会触发地址回退/重试路径。社区已有几乎一模一样的案例：同样的 `ConnectException → ClosedChannelException → SocketChannelImpl.beginConnect`，根因就是 `localhost` 的 IPv4/IPv6 解析问题（见参考链接）。

需要诚实标注的是：**「重试复用同一 channel」「默认 HTTP/2」「池默认无上限」是源码/官方文档确证的；而"具体是哪一次竞态触发了关闭"是基于机制的推断。** 我没有把它归因到某一个官方 bug，但可以给一个旁证：JDK 自身的 `java/net/httpclient/CancelRequestTest` 就曾因间歇性 `ClosedChannelException` 失败（OpenJDK PR #7776 / 8254786），说明 HttpClient 连接建立/关闭路径上的偶发竞态是已知现象。

## 五、修复

### 方案 A：地址改用 `127.0.0.1`

消除 `localhost` 的地址族歧义，让客户端不必在 `::1` 与 `127.0.0.1` 之间回退：

```java
HttpRequest request = HttpRequest.newBuilder(
        URI.create("http://127.0.0.1:" + port + "/api/orders"))
        // ...
        .build();
```

### 方案 B：对传输层异常做有限重试

这一步才是真正让用例稳定下来的关键。前提是**接口幂等**——本项目的下单接口通过请求头 `x-idempotency-key` 保证幂等，因此重试是安全的：

```java
private Response call(String idempotencyKey, String userId, int quantity) throws Exception {
    String body = String.format("{\"userId\":\"%s\",\"productId\":\"%s\",\"quantity\":%d}",
            userId, PRODUCT, quantity);
    HttpRequest request = HttpRequest.newBuilder(URI.create("http://127.0.0.1:" + port + "/api/orders"))
            .header("Content-Type", "application/json; charset=utf-8")
            .header("x-idempotency-key", idempotencyKey)
            .timeout(Duration.ofSeconds(20))
            .POST(HttpRequest.BodyPublishers.ofString(body, StandardCharsets.UTF_8))
            .build();

    // 200 并发首连时 JDK HttpClient 连接池存在瞬时竞态（ClosedChannelException），
    // 下单接口幂等，故对传输层 IOException 做有限重试，避免偶发网络错误污染正确性断言。
    IOException lastError = null;
    for (int attempt = 1; attempt <= 3; attempt++) {
        long begin = System.nanoTime();
        try {
            HttpResponse<String> response =
                    http.send(request, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
            long costMs = (System.nanoTime() - begin) / 1_000_000;
            Matcher matcher = ORDER_NO.matcher(response.body());
            String orderNo = matcher.find() ? matcher.group(1) : null;
            return new Response(response.statusCode(), orderNo, costMs);
        } catch (IOException ex) {
            lastError = ex;
            Thread.sleep(50L * attempt);
        }
    }
    throw lastError;
}
```

### 方案 C（可选）：显式 HTTP/1.1

如果不需要 HTTP/2，可以显式指定版本，绕开明文 h2c 升级，减少连接建立分支：

```java
HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_1_1)
        .connectTimeout(Duration.ofSeconds(5))
        .build();
```

### 取舍对比

| 方案 | 作用 | 代价 | 适用 |
| --- | --- | --- | --- |
| A `127.0.0.1` | 消除双栈回退 | 几乎无 | 本地/单机测试 |
| B 有限重试 | 兜住瞬时连接竞态 | 需要接口幂等 | 压测/客户端调用 |
| C 强制 HTTP/1.1 | 减少连接建立分支 | 放弃 HTTP/2 特性 | 明确不需要 h2 时 |

**为什么不去调服务端？** 服务端已能正常处理请求（`DispatcherServlet` 已被触发），把 `server.tomcat.accept-count`、`max-connections` 调大并不能解决客户端侧的连接建立竞态——方向错了。

## 六、复盘

1. **先分清「服务端不可用」和「客户端连接问题」。** 异常阶段（connect vs 请求处理）+ 服务端日志时间线（是否有请求真正到达）是两个最快的判据。
2. **压测客户端自身也要健壮。** 客户端把偶发的连接抖动抛成 `ExecutionException`，如果测试不区分「传输错误」和「业务断言失败」，就会把网络抖动误报成正确性问题。
3. **幂等是重试的前提。** 同一个 `x-idempotency-key` 重试不会多下单，才敢放心重试。
4. **高并发的"首连"最容易踩 JDK HttpClient 的坑。** 连接复用起来之后反而稳定，问题往往集中在冷启动那一瞬间。

## 参考

- [HttpClient (Java SE 17) — 默认版本 HTTP/2](https://docs.oracle.com/en/java/javase/17/docs/api/java.net.http/java/net/http/HttpClient.html)
- [HttpClient.Version (Java SE 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.net.http/java/net/http/HttpClient.Version.html)
- [java.net.http module summary — `jdk.httpclient.connectionPoolSize` 默认 0](https://docs.oracle.com/en/java/javase/17/docs/api/java.net.http/module-summary.html)
- [OpenJDK `PlainHttpConnection.java`（jdk17u）](https://github.com/openjdk/jdk17u/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/PlainHttpConnection.java)
- [Java's HttpClient doesn't resolve localhost to IPv6 — tanin](https://tanin.nanakorn.com/javas-httpclient-doesnt-resolve-localhost-to-ipv6/)
- [OpenJDK PR #7776（8254786：CancelRequestTest 间歇性 ClosedChannelException）](https://github.com/openjdk/jdk/pull/7776)
