---
title: 深入 JDK HttpClient：连接建立、连接池与关闭路径上的竞态
date: 2026-09-23 03:05:00
tags:
  - Java
  - HttpClient
  - 并发
  - 源码
categories:
  - 技术
  - Java
---

## 引子

上一篇 [服务明明活着，压测却报 ConnectException]({% post_link jdk-httpclient-concurrent-connect-race 点这里 %}) 记录了一次压测偶发 `ClosedChannelException` 的排查，最后给出的修复是「改用 `127.0.0.1` + 幂等重试」。但当时有个问题没讲透：**JDK HttpClient 的连接建立与关闭路径上，到底为什么会存在竞态？**

这篇文章是一次源码级的深挖。目标不是背 API，而是把 `java.net.http` 内部「一次请求如何拿到连接、连接如何被复用、如何被关闭」讲清楚，然后指出竞态究竟藏在哪几条边界上。

> 版本范围：本文基于 **OpenJDK 17（jdk17u）** 的 `jdk.internal.net.http` 实现。JDK 21 之后部分内部类有重构，细节可能不同，但核心模型一致。

<!-- more -->

## 一、全景：先建立心智模型

JDK 的 `HttpClient` 是一个抽象类，真正的实现是 `jdk.internal.net.http.HttpClientImpl`。一次 `send` 背后涉及这些角色：

- **`HttpClientImpl`**：客户端本体，持有连接池、`SelectorManager`、线程池（executor）。
- **`MultiExchange`**：一次「用户请求」的封装，负责重定向、鉴权过滤、以及**请求级重试**。
- **`Exchange` / `ExchangeImpl`**：一次「请求-响应交互」。`ExchangeImpl` 的两种实现是 `Http1Exchange`（HTTP/1.1）和 `Stream`（HTTP/2）。
- **`HttpConnection`**：连接抽象。子类有 `PlainHttpConnection`（明文直连）、`PlainProxyConnection`、`AsyncSSLConnection` 等。
- **`SelectorManager`**：**单线程**的事件循环。所有非阻塞 I/O 事件（连接完成、可读、可写）都在这个线程上处理。
- **`ConnectionPool`**：HTTP/1.1 的 keep-alive 连接池。
- **`FlowTube` / `SequentialScheduler`**：读写数据的响应式管道与串行调度器。

{% mermaid %}
graph TD
    User["调用方<br/>client.send"] --> ME["MultiExchange<br/>请求级重试/重定向"]
    ME --> EX["Exchange"]
    EX --> EI["ExchangeImpl<br/>Http1Exchange / Stream"]
    EI --> HC["HttpConnection<br/>PlainHttpConnection"]
    HC --> SM["SelectorManager<br/>单线程事件循环"]
    HC --> CP["ConnectionPool<br/>HTTP/1.1 keep-alive"]
    EI --> FT["FlowTube / SequentialScheduler"]
    CLI["HttpClientImpl"] --> SM
    CLI --> CP
    CLI --> EXE["Executor 线程池"]
{% endmermaid %}

线程模型很关键，因为**竞态往往就发生在「业务线程」与「SelectorManager 单线程」的交接处**：

- 业务线程：调用 `send`，触发连接建立、写入请求；
- SelectorManager 线程：处理 `OP_CONNECT`、`OP_READ`、`OP_WRITE`；
- Executor 线程：执行异步回调、`CompletableFuture` 的依赖任务。

## 二、冷启动：一次 connect 的完整流程

当池中没有可用连接时，会新建一条 `PlainHttpConnection`，然后调用 `connectAsync`。核心逻辑（`PlainHttpConnection.connectAsync`）：

```java
boolean finished = AccessController.doPrivileged(
        () -> chan.connect(Utils.resolveAddress(address)));
if (finished) {
    client().connectionOpened(this);
    cf.complete(ConnectState.SUCCESS);
} else {
    client().registerEvent(new ConnectEvent(cf, exchange));
}
```

翻译成人话：

1. 对一个**非阻塞** `SocketChannel` 调用 `connect`；
2. 如果 TCP 握手瞬间完成（本机/局域网常见），直接标记成功；
3. 否则把 `ConnectEvent` 注册到 `SelectorManager`，等 selector 收到 `OP_CONNECT` 后调用 `finishConnect()`；
4. 连接成功后调用 `client().connectionOpened(this)`，把连接登记进「活动连接」集合。

{% mermaid %}
sequenceDiagram
    participant T as 业务线程
    participant HC as PlainHttpConnection
    participant SM as SelectorManager
    participant CLI as HttpClientImpl
    T->>HC: connectAsync
    HC->>HC: chan.connect 非阻塞
    alt 立即完成
        HC->>CLI: connectionOpened
    else 未完成
        HC->>SM: registerEvent(ConnectEvent)
        SM->>SM: selector 等待 OP_CONNECT
        SM->>HC: finishConnect
        HC->>CLI: connectionOpened
    end
    HC-->>T: complete SUCCESS
{% endmermaid %}

注意一个细节：连接成功时调用的是 `connectionOpened`；**连接失败时会调用 `close()`**。这两条分支的存在，正是后面 R1 竞态的土壤。

## 三、两层重试：别把它们搞混

JDK HttpClient 里其实有**两层**重试，很多人只知道其中一层。

### 3.1 连接级重试：`checkRetryConnect`

连接建立失败时，`PlainHttpConnection` 会先问一句「要不要重试这个连接」：

```java
private boolean canRetryConnect(Throwable e) {
    if (!MultiExchange.RETRY_CONNECT) return false;
    if (!(e instanceof ConnectException)) return false;
    if (unsuccessfulAttempts > 0) return false;
    ConnectTimerEvent timer = connectTimerEvent;
    if (timer == null) return true;
    return timer.deadline().isAfter(Instant.now());
}
```

如果判定可重试，`checkRetryConnect` 会**用同一个 `SocketChannel` 再调一次 `connectAsync`**：

```java
if (connect == ConnectState.RETRY) {
    int attempts = unsuccessfulAttempts;
    assert attempts <= 1;
    return connectAsync(exchange);
}
```

源码注释解释了它的来由——为某些平台的「连接超时提示不准」做补偿：

> On some platforms, a ConnectEvent may be raised and a ConnectionException may occur with the message "Connection timed out: no further information" before our actual connection timeout has expired. In this case ... we will retry once again.

这就是上一篇里 `ClosedChannelException` 出现的直接路径：**重试复用同一 channel，而失败分支可能已经把它关掉了。**

### 3.2 请求级重试：`MultiExchange.retryOnFailure`

另一层在更上层。`MultiExchange` 判断整个请求要不要重发：

```java
static final boolean RETRY_CONNECT = !disableRetryConnect();

private boolean retryOnFailure(Throwable t) {
    if (requestCancelled()) return false;
    return t instanceof ConnectionExpiredException
            || (RETRY_CONNECT && (t instanceof ConnectException));
}
```

几个要点：

- `RETRY_CONNECT` **默认开启**，即 `ConnectException` 会触发一次请求级重试；可用系统属性 `jdk.httpclient.disableRetryConnect` 关闭。
- 默认只重试**一次**（`retriedOnce`）。
- 只对**幂等方法**（GET/HEAD）自动重试；POST 等默认不重试，除非设置 `jdk.httpclient.enableAllMethodRetry=true`。
- 连接超时用 `ConnectTimeoutTracker` 跨重试共享，避免重试把超时时间翻倍。

### 3.3 两层重试的差异

| | 连接级 `checkRetryConnect` | 请求级 `MultiExchange` |
| --- | --- | --- |
| 触发 | 单次 connect 失败 | 整个请求失败 |
| 复用 | 复用同一 `SocketChannel` | 可能新建连接 |
| 次数 | 1 次 | 1 次 |
| 开关 | `MultiExchange.RETRY_CONNECT` | `disableRetryConnect` / `enableAllMethodRetry` |

**理解这两层，才能理解为什么「偶发连接失败」有时能自愈、有时却直接抛给调用方。**

## 四、热路径：连接池与复用

连接建立之后，HTTP/1.1 的连接会被放回池中复用。`ConnectionPool` 的关键常量：

```java
static final long KEEP_ALIVE = Utils.getIntegerNetProperty(
        "jdk.httpclient.keepalive.timeout", 1200); // 秒
static final long MAX_POOL_SIZE = Utils.getIntegerNetProperty(
        "jdk.httpclient.connectionPoolSize", 0);   // 0 = 无上限
```

池按 `CacheKey`（目标地址 + 代理 + 是否加密）分桶。取连接时从链表头摘除：

```java
synchronized HttpConnection getConnection(...) {
    if (stopped) return null;
    CacheKey key = new CacheKey(secure, addr, proxy);
    HttpConnection c = secure ? findConnection(key, sslPool)
                              : findConnection(key, plainPool);
    return c;
}
```

**连接池里一个非常容易被忽略的机制：`CleanupTrigger`。**

连接放回池时（`returnToPool`），会先在连接的 `FlowTube` 上注册一个 `CleanupTrigger`，然后再把连接加入池：

```java
CleanupTrigger cleanup = registerCleanupTrigger(conn);
synchronized (this) {
    if (cleanup.isDone()) {
        return;              // 注册期间已经被清理
    } else if (stopped) {
        conn.close();
        return;
    }
    ...
    putConnection(conn, plainPool);
    expiryList.add(conn, now, keepAlive);
}
```

`CleanupTrigger` 订阅了这条连接：**只要连接在池中期间对端关闭了连接、或发来了任何数据、或发生错误，它就立刻把连接移出池并关闭。**

```java
@Override public void onError(Throwable error) { triggerCleanup(error); }
@Override public void onComplete() { triggerCleanup(null); }
@Override public void onNext(List<ByteBuffer> item) {
    triggerCleanup(new IOException("Data received while in pool"));
}
```

这是一个很优雅的设计：池里的连接不是「死」的，而是有人盯着它的死活。

## 五、复用时的自检：`checkOpen`

从池里取出的连接，并不一定还活着——对端可能刚关闭，而 `CleanupTrigger` 的清理还没被 selector 线程执行到。为此，`HttpConnection.getConnection` 对明文连接会先做一次 `checkOpen()`：

```java
if (!secure) {
    c = pool.getConnection(false, addr, proxy);
    if (c != null && c.checkOpen()) {   // 可能已被对端关闭
        return c;
    } else {
        return getPlainConnection(addr, proxy, request, client);
    }
}
```

`checkOpen()` 的做法是**尝试从 channel 读 1 个字节**（连接在池中时本应无数据可读）：

```java
final boolean checkOpen() {
    if (isOpen()) {
        try {
            int read = channel().read(ByteBuffer.allocate(1));
            if (read == 0) return true;   // 没数据，看起来还活着
            close();                       // 读到数据是协议错误
        } catch (IOException x) {
            return false;                  // EOF 或异常，已关闭
        }
    }
    return false;
}
```

而 `checkOpen()` 的 Javadoc 直接承认了这里存在竞态，并说明它就是为此而生的：

> It helps minimizing race conditions where the selector manager thread hasn't woken up - or hasn't raised the event, before the connection was retrieved from the pool. It helps reduce the occurrence of "HTTP/1.1 parser received no bytes" exception, when the server closes the connection while it's being taken out of the pool.

**这是 JDK 自己在源码注释里承认的竞态，并给出了缓解手段。** 但「缓解」不等于「消除」——它只能降低窗口，不能保证对端不会在 `checkOpen` 通过之后、真正写入请求之前关闭连接。

## 六、关闭路径

`PlainHttpConnection.close()` 是幂等的，靠一个 `closed` 标志位保护：

```java
@Override
public void close() {
    synchronized (this) {
        if (closed) {
            return;
        }
        closed = true;
    }
    try {
        if (connectTimerEvent != null)
            client().cancelTimer(connectTimerEvent);
        chan.close();
        tube.signalClosed();
    } finally {
        client().connectionClosed(this);
    }
}
```

一次关闭做了四件事：

1. 取消连接超时定时器（否则定时器还会对着一条已关连接发 `ConnectException`）；
2. `chan.close()` 关闭底层 socket；
3. `tube.signalClosed()` 通知读写管道结束；
4. `client().connectionClosed(this)` 把连接从活动集合移除。

什么时候会关闭？

- 响应头里 `Connection: close`，或 `keepAlive` 判定失败（`closeOrReturnToCache`）；
- `checkOpen()` 发现连接已死；
- 池满淘汰最旧连接（`MAX_POOL_SIZE > 0` 时）或 keep-alive 过期被清理；
- 客户端 `stop()`（`ConnectionPool.stop()` 关闭所有连接）；
- 请求被取消（`cancel`）。

{% mermaid %}
flowchart TD
    A["close()"] --> B{"closed 已置位?"}
    B -- 是 --> C["直接返回（幂等）"]
    B -- 否 --> D["cancelTimer"]
    D --> E["chan.close"]
    E --> F["tube.signalClosed"]
    F --> G["connectionClosed 移出活动集合"]
{% endmermaid %}

## 七、竞态地图：到底哪些地方会出问题

把上面所有机制拼起来，竞态集中在**连接生命周期与连接池的几条边界**上。下面逐条给出触发条件、源码位置、机制与现象。

| 编号 | 触发条件 | 源码位置 | 机制 | 现象 | JDK 的缓解 |
| --- | --- | --- | --- | --- | --- |
| R1 | 连接失败后重试 | `PlainHttpConnection.checkRetryConnect` | 重试复用同一 `SocketChannel`，而失败分支已 `close()` | `ClosedChannelException` | 重试前检查 `unsuccessfulAttempts`、超时 |
| R2 | 请求取消/中断与 connect 并发 | `ConnectEvent.handle` / `Exchange.cancel` | `abort`/`close` 与 `finishConnect` 竞争 | `CancellationException`、连接被提前关闭 | `requestCancelled` 判定、幂等 close |
| R3 | 池中连接被对端关闭 vs 被取回复用 | `ConnectionPool.CleanupTrigger` + `HttpConnection.checkOpen` | selector 尚未处理关闭事件时，连接被另一线程取走 | `HTTP/1.1 parser received no bytes`、`IOException` | `checkOpen()` 读 1 字节探测 |
| R4 | 池过期/淘汰清理 vs 取用 | `ConnectionPool.purgeExpiredConnections...` / `stop` | 清理线程 `close` 与取用线程并发 | 使用到已关闭连接 | `synchronized`、`ExpiryList` |
| R5 | 连接超时定时器 vs connect 完成 | `ConnectTimerEvent.handle` | 定时器触发与 `finishConnect` 竞争 | 已建立的连接被判超时 | `cancelTimer`、`deadline` 判定 |

### R1：连接失败重试复用已关 channel

这是上一篇的直接成因。`ConnectEvent.handle` 失败时，要么进入 `RETRY`（**不关闭** channel），要么 `toConnectException` + `close()`。但 `connectAsync` 的异常分支本身也会 `close()`。当「重试」与「关闭」交错，第二次 `chan.connect` 就会在已关闭的 channel 上执行：

```
java.nio.channels.ClosedChannelException
    at sun.nio.ch.SocketChannelImpl.ensureOpen(SocketChannelImpl.java)
    at sun.nio.ch.SocketChannelImpl.beginConnect(SocketChannelImpl.java)
    at jdk.internal.net.http.PlainHttpConnection.lambda$connectAsync$0(PlainHttpConnection.java)
```

高并发同时首连会放大这个窗口。

### R2：取消与连接建立并发

`sendAsync` 返回的 `CompletableFuture` 是可取消的。取消会走到 `MultiExchange.cancel` → `Exchange.cancel` → 连接的 `abort`/`close`。如果此刻 selector 线程正好在 `finishConnect`，就会出现「一边要完成连接、一边要关掉连接」的竞争。JDK 用 `requestCancelled()` 做判断，并保证 `close()` 幂等，把影响收敛为取消语义。

### R3：池中连接被对端关闭 vs 被取回复用

这是**最隐蔽**的一类，也是 `checkOpen` 存在的原因。时序：

{% mermaid %}
sequenceDiagram
    participant App as 业务线程
    participant Pool as ConnectionPool
    participant Clean as CleanupTrigger
    participant Srv as 对端/服务器
    Note over Pool: 空闲连接在池中
    Srv-->>Clean: 对端关闭连接 / 发来数据
    Clean->>Pool: cleanup 移出池并 close
    App->>Pool: getConnection
    Pool-->>App: 返回同一条连接
    App->>Srv: 写入请求
    Srv-->>App: 连接已关闭 → IOException
{% endmermaid %}

关键在于：`CleanupTrigger` 的清理动作是在 **selector 线程**上被驱动的；而 `getConnection` 是在**业务线程**上执行的。两个线程之间没有全局锁，窗口就出现了。`checkOpen()` 把窗口缩小，但无法消除「自检通过之后、写入之前对端关闭」的极窄窗口。

### R4：池清理与取用并发

keep-alive 过期清理、池满淘汰、`stop()` 都会 `close` 连接。这些动作与 `getConnection` 并发时，可能出现「刚取出的连接正被关闭」。`ConnectionPool` 用 `synchronized` 保护了池结构的增删，`returnToPool` 还特意先注册 `CleanupTrigger` 再入池（避免入池后才注册、注册前就被清理的漏洞），但连接本身的关闭与使用仍是跨组件的。

### R5：超时定时器与连接完成竞争

`ConnectTimerEvent` 到点后会构造 `ConnectException("HTTP connect timed out")` 并 `cancel` 整个 exchange。如果 `finishConnect` 与之几乎同时发生，就取决于谁先到达。JDK 通过 `canRetryConnect` 里的 `timer.deadline().isAfter(Instant.now())` 和成功路径的 `cancelTimer` 来收敛。

## 八、我们能做什么

理解了机制，就能把「偶发」变成「可控」。

**客户端侧：**

- **幂等 + 重试**：这是应对 R1/R2/R5 最实用的手段。接口幂等（如通过业务幂等键）时，对传输层异常做有限退避重试；非幂等接口要谨慎。
- **谨慎使用开关**：`jdk.httpclient.disableRetryConnect` 会关闭连接级/请求级重试，通常不要动；`jdk.httpclient.enableAllMethodRetry=true` 会让 POST 也可自动重试，务必确认幂等。
- **连接池参数**：`jdk.httpclient.connectionPoolSize`（默认 0 无上限）、`jdk.httpclient.keepalive.timeout`（默认 1200s）。压测时按需限制池大小，避免连接数失控。
- **地址明确**：本地压测用 `127.0.0.1` 而非 `localhost`，避免 IPv4/IPv6 双栈回退叠加竞态（见上一篇）。
- **显式 HTTP/1.1**：不需要 HTTP/2 时用 `.version(HTTP_1_1)`，绕开明文 h2c 升级，减少连接建立分支。

**服务端侧：**

- 合理配置 keep-alive 与空闲超时，避免服务端先于客户端关闭空闲连接（可显著减少 R3）。
- 不要在负载高峰期主动、批量地关闭连接。

**测试侧：**

- 区分「传输错误」与「业务断言失败」：压测客户端必须容忍瞬时网络抖动，否则会把客户端问题误报成业务正确性问题。
- 记录重试次数、连接建立耗时，便于定位是「客户端连接问题」还是「服务端处理问题」。

## 九、小结

JDK HttpClient 的连接生命周期可以概括为三句话：

1. **建立**：非阻塞 `connect` + `SelectorManager` 事件驱动，失败有连接级与请求级两层重试；
2. **复用**：HTTP/1.1 keep-alive 连接池 + `CleanupTrigger` 盯守 + `checkOpen` 自检；
3. **关闭**：幂等 `close()`，取消定时器、关闭 channel、通知管道、移出活动集合。

竞态恰恰藏在「业务线程」与「selector 单线程」的交接处：**建立失败后的重试、池中连接的关闭与取用、超时与完成的竞争**。JDK 用 `checkOpen`、`CleanupTrigger`、`requestCancelled`、幂等 `close` 等手段做了大量缓解——源码注释也坦承这是「缓解（minimizing）」而非「消除」。作为使用者，我们无法修改 JDK，但可以通过**幂等重试、地址明确、参数调优、测试健壮性**把这些偶发风险控制在可接受范围。

## 参考

- [OpenJDK 17u `PlainHttpConnection.java`](https://github.com/openjdk/jdk17u/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/PlainHttpConnection.java)
- [OpenJDK 17u `MultiExchange.java`](https://github.com/openjdk/jdk17u/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/MultiExchange.java)
- [OpenJDK 17u `ConnectionPool.java`](https://github.com/openjdk/jdk17u/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/ConnectionPool.java)
- [OpenJDK 17u `HttpConnection.java`](https://github.com/openjdk/jdk17u/blob/master/src/java.net.http/share/classes/jdk/internal/net/http/HttpConnection.java)
- [HttpClient (Java SE 17) — 默认版本 HTTP/2](https://docs.oracle.com/en/java/javase/17/docs/api/java.net.http/java/net/http/HttpClient.html)
- [java.net.http module summary — `jdk.httpclient.connectionPoolSize` 默认 0](https://docs.oracle.com/en/java/javase/17/docs/api/java.net.http/module-summary.html)
- [JEP 321: HTTP Client API](https://openjdk.org/jeps/321)
