# Telnet 协议协商与 checkAccept 放行逻辑

> 归档自问答：Telnet 协商在哪个环节、哪部分代码；以及 `onOpen()` 未发 BINARY 时 `checkAccept()` 是否不等客户端回复就把连接交给 Arthas。
>
> 归档日期：2026-06-17

关联文档：[arthas-termd-connection.md](./arthas-termd-connection.md)、[telnet-charset.md](./telnet-charset.md)、[command-flow-and-termd.md](./command-flow-and-termd.md)

关联项目：`D:\mycode\termd`

---

## 问题一：Telnet 协商在哪个环节、哪部分代码？

Telnet 协商发生在 **TCP 连上之后、Arthas 收到第一个用户按键之前**，全部在 termd 的 Telnet 层完成。

### 1. 整体时间线

```text
TCP 连接建立
  → Netty TelnetChannelHandler.channelActive()
  → TelnetConnection.onInit() → TelnetTtyConnection.onOpen()   【服务端主动发 DO/WILL】
  → 客户端回 WILL/DO / 子协商 SB...SE
  → TelnetConnection 状态机解析 IAC
  → Option.xxx.handleXxx() 回调
  → TelnetTtyConnection.onReceiveBinary / onSendBinary / onTerminalType / onSize
  → checkAccept() 条件满足
  → handler.accept(this) 通知 Arthas：new TermImpl(...)
  → 之后用户数据才稳定进入 Readline
```

协商和用户数据 **共用同一条 `receive(byte[])` 路径**；`0xFF (IAC)` 开头的字节走协议状态机，其余进 `onData` → `BinaryDecoder`。

### 2. 入口：谁收到第一个字节

**Netty 收包**（`termd/.../telnet/netty/TelnetChannelHandler.java`）：

```java
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    this.conn = new NettyTelnetConnection(factory.get(), ctx);
    conn.onInit();
}

public void channelRead(ChannelHandlerContext ctx, Object msg) {
    conn.receive(data);   // 协商字节和用户输入都从这里进
}
```

`onInit()` 会立刻触发服务端发起协商：

```java
// TelnetConnection
public void onInit() {
    handler.onOpen(this);
}
```

### 3. 服务端主动发起：发哪些协商

**`TelnetTtyConnection.onOpen()`**（`termd/.../telnet/TelnetTtyConnection.java`）：

```java
protected void onOpen(TelnetConnection conn) {
    this.conn = conn;

    conn.writeWillOption(Option.ECHO);   // 服务端愿意回显
    conn.writeWillOption(Option.SGA);    // 抑制 Go-Ahead

    if (inBinary) {
        conn.writeDoOption(Option.BINARY);   // 请客户端用 8 位二进制发数据
    }
    if (outBinary) {
        conn.writeWillOption(Option.BINARY); // 服务端用 8 位二进制发数据
    }

    conn.writeDoOption(Option.NAWS);           // 窗口大小
    conn.writeDoOption(Option.TERMINAL_TYPE);  // 终端类型

    checkAccept();
}
```

发出的 Telnet 帧（概念上）：

| 代码调用 | 线上字节（示意） | 含义 |
|----------|------------------|------|
| `writeWillOption(ECHO)` | `FF FB 01` | IAC WILL ECHO |
| `writeWillOption(SGA)` | `FF FB 03` | IAC WILL SGA |
| `writeDoOption(BINARY)` | `FF FD 00` | IAC DO BINARY（仅 `inBinary=true`） |
| `writeWillOption(BINARY)` | `FF FB 00` | IAC WILL BINARY（仅 `outBinary=true`） |
| `writeDoOption(NAWS)` | `FF FD 1F` | IAC DO NAWS |
| `writeDoOption(TERMINAL_TYPE)` | `FF FD 18` | IAC DO TERMINAL_TYPE |

**Arthas 现状**：`TelnetTermServer` / `HttpTelnetTermServer` 创建 bootstrap 时 **没有** 调用 `setInBinary(true)` / `setOutBinary(true)`，默认都是 `false`，所以 **Arthas 默认不会主动发 BINARY 协商**。

```java
// TelnetTermServer.listen()
bootstrap = new NettyTelnetTtyBootstrap().setHost(hostIp).setPort(port);
// 没有 setInBinary(true) / setOutBinary(true)
```

### 4. 核心：IAC 状态机在哪解析

**`TelnetConnection.receive()` + 内部枚举 `Status`**（`termd/.../telnet/TelnetConnection.java`）：

```java
public void receive(byte[] data) {
    for (byte b : data) {
        status.handle(this, b);   // 逐字节状态机
    }
    flushDataIfNecessary();
}
```

状态流转要点：

| 状态 | 遇到什么 | 做什么 |
|------|----------|--------|
| `DATA` | 普通字节 | `appendData` → 稍后 `onData` |
| `DATA` | `0xFF` (IAC) | 进入 `IAC`（BINARY 模式下可能先进 `ESC`） |
| `IAC` | `DO/DONT/WILL/WONT` | 进入对应子状态，下一字节是 option code |
| `WILL` | option code | `onOptionWill(code)` |
| `SB` | 子协商内容 | 收齐后 `onOptionParameters`（NAWS、TERMINAL_TYPE） |

`WILL` 分支：

```java
protected void onOptionWill(byte optionCode) {
    for (Option option : Option.values()) {
        if (option.code == optionCode) {
            option.handleWill(this);
            return;
        }
    }
    send(new byte[]{BYTE_IAC, BYTE_DONT, optionCode});  // 不认识的选项回 DON'T
}
```

### 5. 各选项的具体处理：`Option.java`

**BINARY（切 charset 的关键）**：

```java
BINARY((byte) 0) {
    void handleDo(TelnetConnection session) {
        session.sendBinary = true;
        session.handler.onSendBinary(true);      // 出站改 UTF-8 编码
    }
    void handleWill(TelnetConnection session) {
        session.receiveBinary = true;
        session.handler.onReceiveBinary(true);   // 入站改 UTF-8 解码
    }
},
```

**TERMINAL_TYPE**（客户端上报 xterm 等）：

```java
TERMINAL_TYPE((byte) 24) {
    void handleWill(TelnetConnection session) {
        // 客户端同意后，服务端发 SB SEND，要求客户端报类型
        session.send(new byte[]{IAC, SB, 24, SEND, IAC, SE});
    }
    void handleParameters(TelnetConnection session, byte[] parameters) {
        if (parameters[0] == BYTE_IS) {
            String terminalType = new String(parameters, 1, parameters.length - 1);
            session.handler.onTerminalType(terminalType);
        }
    }
},
```

**NAWS**（窗口宽高）：

```java
NAWS((byte) 31) {
    void handleParameters(...) {
        int width = ...; int height = ...;
        session.handler.onSize(width, height);
    }
}
```

### 6. BINARY 协商成功后：切换 `TelnetCharset` → UTF-8

```java
protected void onSendBinary(boolean binary) {
    sendingBinary = binary;
    if (binary) {
        encoder.setCharset(charset);   // 默认 UTF-8
    }
    checkAccept();
}

protected void onReceiveBinary(boolean binary) {
    receivingBinary = binary;
    if (binary) {
        decoder.setCharset(charset);   // 从 TelnetCharset 换成 UTF-8
    }
    checkAccept();
}
```

构造时 decoder 默认是 `TelnetCharset`：

```java
this.decoder = new BinaryDecoder(512, TelnetCharset.INSTANCE, readBuffer);
```

### 7. 何时算「协商完成」，通知 Arthas

**`checkAccept()`** 是闸门：满足条件才把 `TtyConnection` 交给 Arthas：

```java
private void checkAccept() {
    if (!accepted) {
        if (!outBinary | (outBinary && sendingBinary)) {
            if (!inBinary | (inBinary && receivingBinary)) {
                accepted = true;
                readBuffer.setReadHandler(eventDecoder);
                handler.accept(this);   // → TermImpl 创建
            }
        }
    }
}
```

逻辑表（`|` 是逻辑或）：

| inBinary | outBinary | 何时 `accepted` |
|----------|-----------|-----------------|
| false | false | **onOpen 后立刻**（Arthas 默认就是这种） |
| true | false | 客户端 `WILL BINARY` 之后 |
| false | true | 客户端 `DO BINARY` 之后 |
| true | true | 两个都协商完 |

### 8. 一次 BINARY 协商的报文往返（若开启）

假设 `setInBinary(true)`（服务端要客户端发 8 位数据）：

```text
服务端 onOpen:  IAC DO BINARY        (FF FD 00)
客户端:         IAC WILL BINARY      (FF FB 00)
  → onOptionWill(0)
  → Option.BINARY.handleWill
  → onReceiveBinary(true)
  → decoder.setCharset(UTF-8)
  → checkAccept()
```

出站 BINARY（`setOutBinary(true)`）：

```text
服务端 onOpen:  IAC WILL BINARY      (FF FB 00)
客户端:         IAC DO BINARY        (FF FD 00)
  → onOptionDo(0)
  → onSendBinary(true)
  → encoder.setCharset(UTF-8)
```

### 9. TERMINAL_TYPE 协商往返

```text
服务端 onOpen:     IAC DO TERMINAL_TYPE     (FF FD 18)
客户端:            IAC WILL TERMINAL_TYPE    (FF FB 18)
  → handleWill → IAC SB 18 SEND IAC SE
客户端:            IAC SB 18 IS "xterm" IAC SE
  → SB 状态机 → handleParameters → onTerminalType("xterm")
```

`TermImpl.type()` 读的就是这个值，但 **keymap 加载不用它**。

### 10. 协商之后用户输入怎么走

```text
channelRead
  → TelnetConnection.receive(byte[])
      IAC 序列 → Option 处理（或丢弃）
      普通数据 → flushData → TelnetTtyConnection.onData(byte[])
  → BinaryDecoder.write(byte[])
      TelnetCharset 或 UTF-8 解码
  → ReadBuffer → TtyEventDecoder → stdinHandler → Readline
```

### 11. 代码地图（按文件）

| 环节 | 文件 | 职责 |
|------|------|------|
| TCP/Netty 入口 | `termd/.../netty/TelnetChannelHandler.java` | `channelActive` / `channelRead` |
| 连接对象 | `termd/.../netty/NettyTelnetConnection.java` | `send()` 写回客户端 |
| 协议状态机 | `termd/.../telnet/TelnetConnection.java` | IAC 解析、DO/WILL/SB |
| 选项语义 | `termd/.../telnet/Option.java` | BINARY/NAWS/TERMINAL_TYPE 等 |
| TTY 桥接 | `termd/.../telnet/TelnetTtyConnection.java` | `onOpen` 发协商、`checkAccept`、切 charset |
| Arthas 启动 | `arthas/.../TelnetTermServer.java` | 创建 bootstrap（默认不开 BINARY） |
| charset 开关 | `termd/.../io/TelnetCharset.java` | 非 BINARY 入站解码 |

---

## 问题二：`onOpen()` 未发 BINARY 时，是否不等客户端回复就 `checkAccept` 交给 Arthas？

**结论：在 Arthas 默认配置下（`inBinary = false`、`outBinary = false`），`onOpen()` 末尾那次 `checkAccept()` 会立刻通过，连接马上交给 Arthas，不会等客户端对 ECHO / SGA / NAWS / TERMINAL_TYPE 的回复。**

### 为什么 `checkAccept()` 会立刻过？

```java
private void checkAccept() {
    if (!accepted) {
        if (!outBinary | (outBinary && sendingBinary)) {
            if (!inBinary | (inBinary && receivingBinary)) {
                accepted = true;
                readBuffer.setReadHandler(eventDecoder);
                handler.accept(this);   // → new TermImpl(...)
            }
        }
    }
}
```

`inBinary`、`outBinary` 都是 `false` 时：

| 条件 | 计算 | 结果 |
|------|------|------|
| `!outBinary \| (outBinary && sendingBinary)` | `true \| false` | **true** |
| `!inBinary \| (inBinary && receivingBinary)` | `true \| false` | **true** |

两个都成立 → 第一次 `checkAccept()` 就 `accepted = true`，并调用 `handler.accept(this)`。

Arthas 侧没有开 BINARY：

```java
bootstrap = new NettyTelnetTtyBootstrap().setHost(hostIp).setPort(port);
// 没有 setInBinary(true) / setOutBinary(true)
```

### 实际时序（默认 Arthas）

```text
channelActive
  → onInit() → onOpen()
       发出 IAC WILL ECHO
       发出 IAC WILL SGA
       发出 IAC DO NAWS
       发出 IAC DO TERMINAL_TYPE
       checkAccept()  ← 同步执行，不等网络往返
         → handler.accept(conn)
         → TermImpl 创建
         → Shell 启动、readline() ...

（之后异步）客户端陆续回复 WILL/DO、SB 子协商...
  → terminalType、窗口 size 稍后才有值
```

也就是说：

- **把连接交给 Arthas**：`onOpen()` 里同步完成，**不依赖**客户端回复。
- **ECHO / SGA / NAWS / TERMINAL_TYPE**：照样发出去，但在后台慢慢协商；termd **没有**为它们设「等齐了再 accept」的逻辑。

### `checkAccept` 到底在等什么？

**只在开启 BINARY 时才会阻塞：**

| 配置 | 要等什么 |
|------|----------|
| `inBinary=true` | 客户端 `IAC WILL BINARY` → `receivingBinary=true` |
| `outBinary=true` | 客户端 `IAC DO BINARY` → `sendingBinary=true` |
| 两者都 `false`（Arthas 默认） | **什么都不等** |

`checkAccept()` 还会在 `onReceiveBinary` / `onSendBinary` 里再调一次，那是给「后来才协商完 BINARY」用的；默认路径下第一次就已经 accept 了。

### 协商还没完，用户输入会丢吗？

不会。`BinaryDecoder` 的输出先进 `ReadBuffer`：

- `checkAccept()` 之前：`readHandler == null`，数据 **入队**；
- `checkAccept()` 里 `setReadHandler(eventDecoder)` 后：`drainQueue()` 把队列里的数据补发给上层。

所以即使用户极快敲键，也只是短暂缓冲，不会丢。

### 小结

1. **默认不发 BINARY 协商**（`inBinary`/`outBinary` 为 false）。
2. **`onOpen()` 末尾 `checkAccept()` 立即通过**，同步 `handler.accept(this)`，**TermImpl / Shell 马上起来**，不等客户端。
3. **ECHO、SGA、NAWS、TERMINAL_TYPE** 是「发了就继续」的异步协商；`terminalType()`、`size()` 可能稍后才有效，但不影响连接先交给 Arthas。

若把 `setInBinary(true)` / `setOutBinary(true)` 打开，才会变成「等 BINARY 握手完成再 accept」（或至少在 `onReceiveBinary`/`onSendBinary` 回调里第二次 `checkAccept` 才真正放行——取决于第一次是否已过）。Arthas 当前没开，所以一直是「连上就交给 Arthas」。

---

## 相关源码索引

| 主题 | 文件 |
|------|------|
| `onOpen` 发协商 | `termd/.../telnet/TelnetTtyConnection.java` |
| IAC 状态机 | `termd/.../telnet/TelnetConnection.java` |
| 选项处理 | `termd/.../telnet/Option.java` |
| `checkAccept` | `termd/.../telnet/TelnetTtyConnection.java` |
| ReadBuffer 缓冲 | `termd/.../tty/ReadBuffer.java` |
| Arthas bootstrap 默认配置 | `arthas/.../TelnetTermServer.java`、`HttpTelnetTermServer.java` |
