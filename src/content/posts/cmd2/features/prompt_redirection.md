---
title: Python cmd2：提示符定制、输出重定向与管道
published: 2026-05-28
pinned: false
description: 介绍 cmd2 的提示符定制功能，包括自定义提示符、续行提示符、异步反馈和底部工具栏。介绍 cmd2 的输出重定向和管道功能，包括重定向到文件、剪贴板、管道传递给 shell 命令，以及禁用和限制。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 提示符

`cmd2` 在请求用户输入之前会发出一个可配置的提示符。

## 自定义提示符

此提示符可以通过设置 `cmd2.Cmd.prompt` 实例属性来配置。它包含应作为用户输入提示符打印的字符串。参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例了解静态设置提示符的简单用例。

## 续行提示符

当用户输入[多行命令](../app_setup/#多行命令)时，它可能跨越多行输入。第一行输入的提示符由 `cmd2.Cmd.prompt` 实例属性指定。后续行输入的提示符由 `cmd2.Cmd.continuation_prompt` 属性定义。参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例了解自定义续行提示符的演示。

## 更新提示符

如果你希望在命令之间更新提示符，可以使用[应用生命周期钩子](../hooks/#应用生命周期钩子)之一，例如[命令后钩子](../hooks/#命令后钩子)。参阅 [python_scripting.py](https://github.com/python-cmd2/cmd2/blob/main/examples/python_scripting.py) 了解动态更新提示符的示例。

## 异步反馈

`cmd2` 提供了一个函数来向用户提供异步反馈，而不会干扰命令行。这允许在用户仍在提示符处输入文本时提供反馈。

- `cmd2.Cmd.add_alert`

### 异步反馈机制

警报可以通过两种方式与 CLI 交互：

1. **消息打印**：可以在当前提示行上方直接打印消息
2. **提示符更新**：可以动态替换活动提示符的文本以反映变化的状态

:::note
为确保用户界面保持准确，如果警报是在当前提示符渲染之前创建的，则提示符更新将被忽略。这可以防止旧警报覆盖较新的提示符，尽管警报的消息仍会被打印。
:::

### 终端窗口管理

`cmd2` 还提供了一个函数来更改终端窗口的标题。

- `cmd2.Cmd.set_window_title`

理解这些函数的最简单方式是参阅 [async_printing.py](https://github.com/python-cmd2/cmd2/blob/main/examples/async_printing.py) 示例了解演示。

## 底部工具栏

`cmd2` 支持一个可选的、持久的底部工具栏，在应用空闲等待输入时始终显示在终端窗口底部。

### 启用工具栏

要启用工具栏，在 `cmd2.Cmd.__init__` 构造函数中设置 `bottom_toolbar=True`：

```py
class App(cmd2.Cmd):
    def __init__(self):
        super().__init__(bottom_toolbar=True)
```

### 自定义工具栏内容

你可以通过重写 `cmd2.Cmd.get_bottom_toolbar` 方法来自定义工具栏的内容。此方法应返回一个字符串或 `(style, text)` 元组列表用于格式化文本。

```py
    def get_bottom_toolbar(self) -> list[str | tuple[str, str]] | None:
        return [
            ('ansigreen', 'My Application Name'),
            ('', ' - '),
            ('ansiyellow', 'Current Status: Idle'),
        ]
```

### 刷新工具栏

由于工具栏由 `prompt-toolkit` 作为提示符的一部分渲染，它会在提示符刷新时自然重绘。如果你希望工具栏自动更新（例如显示时钟），可以使用后台线程定期调用 `app.invalidate()`。

参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例了解此技术的演示。

# 输出重定向与管道

与 POSIX shell 类似，命令的输出可以被重定向和/或管道传递。此功能完全跨平台，在 Windows、macOS 和 Linux 上的工作方式相同。

## 输出重定向

### 重定向到文件

将 `cmd2` 命令的输出重定向到文件的方式与 POSIX shell 中相同：

- 使用 `>` 发送到文件，如 `mycommand args > filename.txt`
- 使用 `>>` 追加到文件，如 `mycommand args >> filename.txt`

如果你需要在命令中包含任何这些重定向字符，可以将它们用引号括起来：`mycommand 'with > in the argument'`。

### 重定向到剪贴板

`cmd2` 输出重定向支持大多数 shell 中没有的额外功能——如果 `>` 或 `>>` 后面的文件名留空，则输出被重定向到操作系统剪贴板，以便可以粘贴到另一个程序中。

- 使用 `mycommand args >` 覆盖剪贴板
- 使用 `mycommand args >>` 追加到剪贴板

## 管道

将 `cmd2` 命令的输出管道传递给 shell 命令的方式与 POSIX shell 中相同：

- 使用 `|` 作为 shell 命令的输入管道传递，如 `mycommand args | wc`

## 多管道和重定向

支持多个管道，后面可选择跟随重定向。因此可以执行类似以下的操作：

    (Cmd) help | grep py | wc > output.txt

上述运行 **help** 命令，将其输出管道传递给搜索包含 _py_ 的行的 **grep**，然后将 grep 的输出管道传递给 **wc** "单词计数"命令，最后将该输出重定向到名为 _output.txt_ 的文件。

## 禁用重定向

:::note
如果你希望禁用 cmd2 的输出重定向和管道功能，可以通过将 `cmd2.Cmd` 类实例的 `allow_redirection` 属性设为 `False` 来实现。例如，如果你出于安全原因希望限制最终用户写入磁盘或与 shell 命令交互的能力时，这会很有用：

```py
from cmd2 import Cmd
class App(Cmd):
    def __init__(self):
        super().__init__(allow_redirection=False)
```

cmd2 的解析器仍然会将 `>`、`>>` 和 `|` 符号视为输出重定向和管道符号，并相应地从命令行参数中去除它们之后的参数。但命令的输出不会被重定向到文件或管道传递给 shell 命令。
:::

## 重定向的限制

:::warning
`cmd2` 应用中的重定向和管道有一些限制：

- 只能管道传递给 shell 命令，不能传递给其他 `cmd2` 应用命令
- **stdout** 会被重定向/管道传递，**stderr** 不会
:::
