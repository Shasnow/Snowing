---
title: Python cmd2：应用初始化、多行命令与启动命令
published: 2026-05-28
pinned: false
description: 介绍 cmd2 应用的初始化配置、类变量与实例属性、多行命令的使用，以及启动时执行命令的多种方式。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 应用初始化

以下是一个基本的 `cmd2` 应用示例，演示了你在初始化应用时可能希望利用的许多功能：
```py
#!/usr/bin/env python3
"""A simple example cmd2 application demonstrating many common features.

Features demonstrated include all of the following:
 1) Colorizing/stylizing output
 2) Using multiline commands
 3) Persistent history
 4) How to run an initialization script at startup
 5) How to group and categorize commands when displaying them in help
 6) Opting-in to using the ipy command to run an IPython shell
 7) Allowing access to your application in py and ipy
 8) Displaying an intro banner upon starting your application
 9) Using a custom prompt
10) How to make custom attributes settable at runtime.
11) Shortcuts for commands
12) Persistent bottom toolbar with realtime status updates
"""

import pathlib
import threading
import time

from prompt_toolkit.formatted_text import FormattedText
from rich.style import Style

import cmd2
from cmd2 import (
    Color,
    stylize,
)


class BasicApp(cmd2.Cmd):
    """Cmd2 application to demonstrate many common features."""

    DEFAULT_CATEGORY = "My Custom Commands"

    def __init__(self) -> None:
        """Initialize the cmd2 application."""
        # Startup script that defines a couple aliases for running shell commands
        alias_script = pathlib.Path(__file__).absolute().parent / ".cmd2rc"

        # Create a shortcut for one of our commands
        shortcuts = cmd2.DEFAULT_SHORTCUTS
        shortcuts.update({"&": "intro"})
        super().__init__(
            auto_suggest=True,
            bottom_toolbar=True,
            include_ipy=True,
            multiline_commands=["echo"],
            persistent_history_file="cmd2_history.dat",
            shortcuts=shortcuts,
            startup_script=str(alias_script),
        )

        # Spawn a background thread to refresh the bottom toolbar twice a second.
        # This is necessary because the toolbar contains a timestamp that we want to keep current.
        self._stop_refresh = False
        self._refresh_thread = threading.Thread(target=self._refresh_bottom_toolbar, daemon=True)
        self._refresh_thread.start()

        # Prints an intro banner once upon application startup
        self.intro = (
            stylize(
                "Welcome to cmd2!",
                style=Style(color=Color.GREEN1, bgcolor=Color.GRAY0, bold=True),
            )
            + " Note the full Unicode support:  😇 💩"
            + " and the persistent bottom bar with realtime status updates!"
        )

        # Show this as the prompt when asking for input
        self.prompt = "myapp> "

        # Used as prompt for multiline commands after the first line
        self.continuation_prompt = "... "

        # Allow access to your application in py and ipy via self
        self.self_in_py = True

        # Color to output text in with echo command
        self.foreground_color = Color.CYAN.value

        # Make echo_fg settable at runtime
        fg_colors = [c.value for c in Color]
        self.add_settable(
            cmd2.Settable(
                "foreground_color",
                str,
                "Foreground color to use with echo command",
                self,
                choices=fg_colors,
            )
        )

    def get_rprompt(self) -> str | FormattedText | None:
        current_working_directory = pathlib.Path.cwd()
        style = "bg:ansired fg:ansiwhite"
        text = f"cwd={current_working_directory}"
        return FormattedText([(style, text)])

    def _refresh_bottom_toolbar(self) -> None:
        """Background thread target to refresh the bottom toolbar.

        This is a toy example to show how the bottom toolbar can be used to display
        realtime status updates in an otherwise line-oriented command interpreter.
        """
        import contextlib

        from prompt_toolkit.application.current import get_app

        while not self._stop_refresh:
            with contextlib.suppress(Exception):
                # get_app() will return the currently running prompt-toolkit application
                app = get_app()
                if app:
                    app.invalidate()
            time.sleep(0.5)

    def do_intro(self, _: cmd2.Statement) -> None:
        """Display the intro banner."""
        self.poutput(self.intro)

    def do_echo(self, arg: cmd2.Statement) -> None:
        """Multiline command."""
        self.poutput(
            stylize(
                arg,
                style=Style(color=self.foreground_color),
            )
        )


if __name__ == "__main__":
    import sys

    app = BasicApp()
    sys.exit(app.cmdloop())
```

## Cmd 类初始化器

`cmd2.Cmd` 实例或子类实例是一个交互式 CLI 应用框架。没有充分的理由直接实例化 `Cmd`；相反，它作为你自定义类的超类来继承 `Cmd` 的方法并封装操作方法。

某些内容必须在你从 `cmd2.Cmd` 派生的类的 `__init__()` 方法中初始化（`__init__()` 的所有参数都是可选的）。

## Cmd 类变量

`cmd2.Cmd` 类提供了几个类级别的变量，可以在子类中被重写以更改该类所有实例的默认行为。

- **DEFAULT_CATEGORY**：尚未显式分类的命令的默认帮助分类。（默认：`"Cmd2 Commands"`）
- **DEFAULT_EDITOR**：`edit` 命令使用的默认编辑器程序。
- **DEFAULT_PROMPT**：默认提示符字符串。（默认：`"(Cmd) "`）
- **MISC_HEADER**：列出杂项帮助主题的帮助部分标题。（默认：`"Miscellaneous Help Topics"`）

## Cmd 实例属性

`cmd2.Cmd` 类提供了大量公共实例属性，允许开发者在 `__init__()` 方法提供的选项之外进一步自定义 `cmd2` 应用。

### 公共实例属性

以下是开发者可能希望重写的 `cmd2.Cmd` 实例属性：

- **bottom_toolbar**：如果为 `True`，将显示底部工具栏（默认：`False`）
- **broken_pipe_warning**：如果非空，当发生管道断裂错误时将显示此字符串
- **continuation_prompt**：多行命令在第 2 行及后续行输入时使用的提示符
- **debug**：如果为 `True`，在错误时显示完整堆栈跟踪（默认：`False`）
- **default_error**：运行不存在的命令时打印的错误信息
- **disabled_commands**：已被禁用的命令。用于支持仅在应用特定状态下可用的命令。此字典的键是命令名称，值是 DisabledCommand 对象
- **echo**：如果为 `True`，用户发出的每个命令在执行前会被回显到屏幕上。这在运行脚本时特别有用。在提示符处运行命令时不会出现此行为。（默认：`False`）
- **editor**：与 _edit_ 命令一起使用的文本编辑器程序（如 `vim`）
- **exclude_from_history**：从 _history_ 命令中排除的命令
- **exit_code**：决定退出应用时 `cmdloop()` 返回的值
- **help_error**：找不到帮助信息时打印的错误信息
- **hidden_commands**：从帮助菜单和 Tab 补全中排除的命令
- **last_result**：存储上一个命令运行的结果，以便在 Python 脚本或交互式控制台中使用结果。内置命令不使用此属性，它纯粹为用户定义的命令和便利而设
- **macros**：宏名称及其值的字典
- **max_column_completion_results**：单列中显示的最大补全结果数（默认：7）
- **max_completion_table_items**：显示补全表允许的最大补全结果数（默认：50）
- **pager**：设置 `Cmd.ppaged()` 方法用于使用分页器显示换行输出的分页器命令
- **pager_chop**：设置 `Cmd.ppaged()` 方法用于使用分页器显示截断输出的分页器命令
- **py_bridge_name**：嵌入式 Python 环境和脚本用来调用命令的 `cmd2` 应用名称（默认：`app`）
- **py_locals**：定义 Python shell 和脚本中可用的特定变量/函数的字典（比使用 **self_in_py** 使所有内容可用提供更精细的控制）
- **quiet**：如果为 `True`，完全抑制非必要输出（默认：`False`）
- **scripts_add_to_history**：如果为 `True`，脚本和 pyscripts 会将命令添加到历史记录（默认：`True`）
- **self_in_py**：如果为 `True`，允许在 _py_ 命令中通过 `self` 访问你的应用（默认：`False`）
- **settable**：控制哪些实例属性可以在运行时使用 _set_ 命令设置的字典
- **timing**：如果为 `True`，显示每个命令的执行时间（默认：`False`）

# 多行命令

对于名称列在 `cmd2.Cmd.__init__` 的 `multiline_commands` 参数中的命令，命令输入可以跨越多行。这些命令只有在用户输入了_终止符_后才会被执行。默认情况下，命令终止符是 `;`。通过指定 `cmd2.Cmd.__init__()` 的 `terminators` 可选参数可以设置不同的终止符。空行_始终_被视为命令终止符（不可被重写）。

在多行命令中，输出重定向字符如 `>` 和 `|` 是命令参数的一部分，除非它们出现在终止符之后。

## 续行提示符

当用户输入**多行命令**时，它可能跨越多行输入。第一行输入的提示符由 `cmd2.Cmd.prompt` 实例属性指定——详见[自定义提示符](../prompt_redirection/#自定义提示符)。后续行输入的提示符由 `cmd2.Cmd.continuation_prompt` 属性定义。

## 使用场景

多行命令应该谨慎使用，以保持基于 `cmd2` 的面向行的命令解释器应用的良好用户体验。

然而，某些使用场景从跨越多行的命令能力中受益匪浅。例如，你可能希望用户能够输入 SQL 命令，这些命令通常跨越多行并以分号终止。

我们估计不到 5% 的 `cmd2` 应用使用此功能。但它在这里为那些能从中获得价值的使用场景而准备。

# 启动命令

`cmd2` 提供了几种在应用启动后立即运行命令的方式：

1. 调用时传入命令
2. 启动脚本

作为启动脚本一部分运行的命令总是在应用完成初始化后立即运行，因此它们保证在任何_调用时命令_之前运行。

## 调用时传入命令

你可以在调用应用时通过将命令作为额外参数传递给程序来发送命令。`cmd2` 将每个参数解释为一个单独的命令，因此如果命令超过一个单词，你应该将每个命令用引号括起来。你可以使用单引号或双引号。

    $ python examples/cmd_as_argument.py "say hello" "say Gracie" quit
    hello
    Gracie

你可以在命令末尾加上 **quit** 命令，这样你的 `cmd2` 应用就会像非交互式命令行工具（CLU）一样运行。这意味着它可以被外部应用程序脚本化，并轻松用于自动化。

:::note
如果你希望禁用 cmd2 对命令行参数的消费，可以通过将 `cmd2.Cmd` 类实例的 `allow_cli_args` 参数设为 `False` 来实现。例如，如果你想使用 [argparse](https://docs.python.org/3/library/argparse.html) 来解析应用的整体命令行参数时，这会很有用：

```py
from cmd2 import Cmd
class App(Cmd):
    def __init__(self):
        super().__init__(allow_cli_args=False)
```
:::

## 启动脚本

你可以通过将文件路径传递给 `cmd2.Cmd.__init__()` 方法的 `startup_script` 参数来从初始化脚本执行命令：

```py
class StartupApp(cmd2.Cmd):
    def __init__(self):
        cmd2.Cmd.__init__(self, startup_script='.cmd2rc')
```

此文本文件应包含一个[命令脚本](../scripting/#命令脚本)。参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例了解演示。

你可以通过将 `silence_startup_script` 设为 True 来静音启动脚本的输出：

```py
cmd2.Cmd.__init__(self, startup_script='.cmd2rc', silence_startup_script=True)
```

:::warning
写入 `stderr` 的内容仍会为"静音"的启动脚本打印。此外，如果 `allow_redirection` 为 False，则启动脚本无法被静音，因为静音功能通过将脚本输出重定向到 `os.devnull` 来实现。
:::
