# Arthas 源码阅读笔记

本目录归档 Arthas 及底层 termd 相关的技术问答与源码分析。

| 文档 | 说明 |
|------|------|
| [command-flow-and-termd.md](./command-flow-and-termd.md) | 命令行输入 → 后端执行的完整链路与 termd 数据流转 |
| [term-echo-and-readline.md](./term-echo-and-readline.md) | 「回显（echo）」含义，以及 DefaultTermStdinHandler 与 Readline 的分工 |
| [readline-interaction-handle.md](./readline-interaction-handle.md) | `Readline.Interaction#handle` 深度解析：按键分发、ANSI diff、ACCEPT_LINE、Keymap |

关联项目：`D:\mycode\termd`（Arthas 终端底层网络交互库）
