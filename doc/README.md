# Arthas 源码阅读笔记

本目录归档 Arthas 及底层 termd 相关的技术问答与源码分析。

| 文档 | 说明 |
|------|------|
| [command-flow-and-termd.md](./command-flow-and-termd.md) | 命令行输入 → 后端执行的完整链路与 termd 数据流转 |
| [term-echo-and-readline.md](./term-echo-and-readline.md) | 「回显（echo）」含义，以及 DefaultTermStdinHandler 与 Readline 的分工 |
| [readline-interaction-handle.md](./readline-interaction-handle.md) | `Readline.Interaction#handle` 深度解析：按键分发、ANSI diff、ACCEPT_LINE、Keymap |
| [inputrc-and-key-flow.md](./inputrc-and-key-flow.md) | Arthas `inputrc` 键盘绑定、控制台按键触发、输入在 termd 中的流转；功能键序列与 OS/终端兼容性 |
| [telnet-charset.md](./telnet-charset.md) | `TelnetCharset` 入站解码器：NVT 语义、`\r\n` 归一化、与 UTF-8/BINARY 的关系 |
| [telnet-negotiation-and-accept.md](./telnet-negotiation-and-accept.md) | Telnet 协议协商代码路径（`onOpen`、IAC 状态机、`Option`）；`checkAccept` 默认立即放行逻辑 |
| [arthas-termd-connection.md](./arthas-termd-connection.md) | Arthas 与 termd 建立连接的关键代码：`listen()`、`checkAccept()`、三阶段调用链（bind / 会话 / 按键） |

关联项目：`D:\mycode\termd`（Arthas 终端底层网络交互库）
