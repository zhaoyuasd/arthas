# TelnetCharset 入站解码器

> 归档自问答：`io.termd.core.io.TelnetCharset` 这个类是干嘛的。
>
> 归档日期：2026-06-17

关联文档：[telnet-negotiation-and-accept.md](./telnet-negotiation-and-accept.md)、[command-flow-and-termd.md](./command-flow-and-termd.md)

关联项目：`D:\mycode\termd`

---

## 一句话

`TelnetCharset` 是 termd 里为 **Telnet 入站字节流** 定制的一个 `Charset`（解码器），用来在 `BinaryDecoder` 里把 `byte[]` 转成字符，再往上变成 `int[]` 码点。它实现的是 **Telnet NVT（网络虚拟终端）的 ASCII 行规程**：按字节解码，并把 Telnet 规定的 **`\r\n` / `\r\0` 折成单个 `\r`**，避免上层把「回车」当成两个字符。

---

## 在哪里用

`TelnetTtyConnection` 创建 `BinaryDecoder` 时 **默认** 用它：

```java
// TelnetTtyConnection 构造
this.decoder = new BinaryDecoder(512, TelnetCharset.INSTANCE, readBuffer);
```

协商好 **BINARY 入站** 之后会换成真正的 charset（Arthas 里一般是 UTF-8）：

```java
protected void onReceiveBinary(boolean binary) {
    receivingBinary = binary;
    if (binary) {
        decoder.setCharset(charset);  // 例如 UTF-8，不再用 TelnetCharset
    }
    checkAccept();
}
```

所以它的生命周期是：

| 阶段 | 入站解码用的 Charset |
|------|---------------------|
| 连接初期 / 未开 BINARY 入站 | `TelnetCharset.INSTANCE` |
| BINARY 入站协商成功后 | 连接配置的 `charset`（如 UTF-8） |

出站编码 **不用** `TelnetCharset`（`newEncoder()` 直接抛异常），出站走 `BinaryEncoder` + 普通 charset。

---

## 它具体做什么

类注释：

```java
/**
 * Ascii based telnet charset.
 *
 * The decoder transforms {@code \r\n} sequence and {@code \r0} to {@code \r}.
 */
```

### 1. 按字节 → 字符（偏 ASCII / Latin-1）

```java
if (b >= 0) {
    c = (char) b;           // 0~127：直接当 ASCII
} else {
    c = (char)(256 + b);    // 128~255：按 Latin-1 扩展
}
```

不是 UTF-8 多字节解码；**一个 byte 对应一个 char**（Telnet 非 BINARY 时代的 NVT 语义）。

### 2. Telnet 换行归一化（核心逻辑）

```java
if (prevCR && (b == '\n' || b == 0)) {
    pos++;
    prevCR = false;
    continue;   // 吞掉 \n 或 \0，不输出
}
c = (char) b;
prevCR = (b == '\r');
```

| 网络收到的字节 | 解码结果 |
|--------------|----------|
| `\r` `\n` | 只产出 **一个** `\r` |
| `\r` `\0` | 只产出 **一个** `\r` |
| 其他 | 按单字节映射 |

这是 Telnet 协议里的惯例：发送端用 `CR LF` 表示行结束，接收端应把这一对 **规范化** 成一个行结束符，而不是让上层看到 `\r` 和 `\n` 两个事件。

单元测试验证了这一点（`TelnetCharsetTest.testDecodeCRLF` / `testDecodeCRNULL`）。

---

## 和 UTF-8 的关系

- **`TelnetCharset`**：Telnet 传统 NVT 模式，单字节 + CR/LF 规整；**不负责** UTF-8 中文等多字节。
- **BINARY + UTF-8**：协商成功后 `BinaryDecoder.setCharset(UTF-8)`，中文等按 UTF-8 正常解码；此时 **不再走** `TelnetCharset` 的 CR/LF 特殊规则（由 UTF-8 标准解码器处理）。

Arthas 现代连接通常会协商 BINARY，所以大部分时间实际用的是 UTF-8；`TelnetCharset` 更多是 **连接早期或未开 BINARY 时的默认入站解码器**，以及协议兼容兜底。

**注意**：Arthas 默认 `inBinary=false`，不会主动发 BINARY 协商，因此多数 Arthas Telnet 连接 **全程使用 `TelnetCharset`** 做入站解码（除非客户端主动发 `WILL BINARY`）。详见 [telnet-negotiation-and-accept.md](./telnet-negotiation-and-accept.md)。

---

## 数据流中的位置

```text
客户端按键/输入
  → TCP 字节
  → TelnetConnection（剥 IAC）
  → BinaryDecoder.write(byte[])
      → CharsetDecoder.decode()
           默认: TelnetCharset（NVT + \r\n→\r）
           BINARY 后: UTF-8 等
      → char → int[] 码点
  → ReadBuffer → TtyEventDecoder → stdinHandler → Readline ...
```

---

## 小结

| 问题 | 答案 |
|------|------|
| 干什么的？ | Telnet **入站**专用的单字节 Charset 解码器 |
| 特殊行为？ | `\r\n`、`\r\0` 合并成一个 `\r` |
| 区分 Win/Linux/Xshell？ | **不区分**；只管字节流规程 |
| 和 UTF-8？ | 默认用它；BINARY 协商后切到 UTF-8 |
| 能编码吗？ | 不能，`newEncoder()` 不支持 |

---

## 相关源码

| 文件 | 职责 |
|------|------|
| `termd/.../io/TelnetCharset.java` | 自定义 Charset 解码 |
| `termd/.../io/TelnetCharsetTest.java` | CR/LF、CR/NUL 归一化测试 |
| `termd/.../io/BinaryDecoder.java` | 使用 Charset 做 byte[]→int[] |
| `termd/.../telnet/TelnetTtyConnection.java` | 默认 TelnetCharset，BINARY 后切换 |
