---
title: Python cmd2：构建强大的命令行解释器
published: 2026-05-28
pinned: false
description: 介绍 Python cmd2 包的使用方法，用于构建强大的命令行解释器（CLI）程序，扩展 Python 标准库的 cmd 模块。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---
本系列文档基于 [cmd2 官方文档](https://cmd2.readthedocs.io/en/latest/) 翻译和修改，适用于 cmd2>=4.0.0

::github{repo="python-cmd2/cmd2"}

# cmd2：构建强大的命令行解释器

cmd2 是一个 Python 包，用于构建强大的命令行解释器（CLI）程序。它扩展了 Python 标准库的 [cmd](https://docs.python.org/3/library/cmd.html) 模块。

`cmd2` 的基本用法与 `cmd` 模块完全相同。

## 快速开始

1. 创建一个 `cmd2.Cmd` 的子类。定义属性和 `do_*` 方法来控制其行为。
   在本文档中，我们假设你将子类命名为 `App`：

```py title="创建继承自 cmd2.Cmd 的类"
from cmd2 import Cmd

class App(Cmd):
    # 自定义属性和方法
    pass
```

2. 实例化 `App` 并启动命令循环：

```py title="实例化并启动 cmd2 应用"
from cmd2 import Cmd

class App(Cmd):
    # 自定义属性和方法
    pass

app = App()
app.cmdloop()
```

# cmd2 入门指南

正在构建一个新的 [REPL（读取-求值-打印循环）](https://en.wikipedia.org/wiki/Read–eval–print_loop) 或 [命令行界面（CLI）](https://en.wikipedia.org/wiki/Command-line_interface) 应用程序吗？

已经构建了一个使用 Python 标准库中 [cmd](https://docs.python.org/3/library/cmd.html) 模块的应用程序，并希望通过很少的工作添加更多功能吗？

`cmd2` 是一个用于构建命令行应用程序的强大 Python 库。从这里开始，了解这个库是否适合你的需求。

## 为什么选择 cmd2？

`cmd2` 提供了许多开箱即用的功能，使构建命令行应用程序变得简单而强大：

1. **自动补全** - 为命令和参数提供智能自动补全
2. **历史记录** - 自动保存和检索命令历史
3. **帮助系统** - 自动生成帮助文档
4. **参数解析** - 强大的参数解析和验证
5. **输出重定向** - 支持将输出重定向到文件
6. **别名系统** - 创建命令别名以提高效率
7. **多命令支持** - 在一行中执行多个命令

## 安装说明

`cmd2` 可在 Linux、macOS 和 Windows 上运行。它需要 Python 3.11 或更高版本、[pip](https://pypi.org/project/pip) 和 [setuptools](https://pypi.org/project/setuptools)。如果你已经具备这些条件，那么可以直接运行：

```shell
$ pip install cmd2
```

:::note
根据你在系统上安装 Python 的方式、位置以及使用的操作系统，你可能需要管理员或 root 权限才能安装 Python 包。如果是这种情况，请采取必要的步骤以 root/管理员身份运行本节中的命令，例如：在大多数 Linux 或 Mac 系统上，你可以在命令前加上 `sudo`：

```shell
$ sudo pip install <package_name>
```
:::

:::tip
你也可以使用其他 Python 包管理器，如 [uv](https://github.com/astral-sh/uv)，但这超出了本安装指南的范围。`cmd2` 开发者非常喜欢并强烈推荐 `uv`，并且 `cmd2` 本身的开发也使用它。但如果你是一位足够资深的 Python 开发者，已经在使用 `uv`，那么你就不需要我们来告诉你如何使用它了
:::
### 先决条件

如果你从 [python.org](https://www.python.org) 安装了 Python >=3.11，你将已经拥有 [pip](https://pypi.org/project/pip) 和 [setuptools](https://pypi.org/project/setuptools)，但可能需要升级到最新版本：

在 Linux 或 OS X 上：

```shell
$ pip install -U pip setuptools
```

在 Windows 上：

```shell
C:\> python -m pip install -U pip setuptools
```

### 从 PyPI 安装

[pip](https://pypi.org/project/pip) 是推荐的安装器。使用 pip 从 [PyPI](https://pypi.org) 安装包非常简单：

```shell
$ pip install cmd2
```

这将安装所需的第三方依赖项（如果需要）。

### 从 GitHub 安装

最新版本的 `cmd2` 可以直接从 GitHub 的 main 分支使用 [pip](https://pypi.org/project/pip) 安装：

```shell
$ pip install -U git+git://github.com/python-cmd2/cmd2.git
```

### 从 Debian 或 Ubuntu 仓库安装

我们建议从 [pip](https://pypi.org/project/pip) 安装，但如果你希望从 Debian 或 Ubuntu 仓库安装，可以使用 apt-get。

对于 Python 3：

    $ sudo apt-get install python3-cmd2

这也将安装所需的第三方依赖项。

:::warning
2.0.0 之前的 `cmd2` 版本应被视为不稳定的 "beta" 质量，不应依赖于生产环境使用。如果你无法从操作系统仓库获取 >= 2.0.0 的版本，那么我们建议从 PyPI 或 GitHub 安装 - 参见 [Pip 安装](#从 PyPI 安装) 或 [从 GitHub 安装](#从 GitHub 安装)。
:::

### 升级 cmd2

将已安装的 `cmd2` 升级到 [PyPI](https://pypi.org) 的最新版本：

    pip install -U cmd2

这将升级到 `cmd2` 的最新稳定版本，并在必要时升级任何依赖项。

### 卸载 cmd2

如果你希望永久卸载 `cmd2`，也可以使用 [pip](https://pypi.org/project/pip) 轻松完成：

    $ pip uninstall cmd2

## 将 cmd2 集成到你的项目中

安装完成后，你需要确保项目的依赖项中包含 `cmd2`。请确保你的 `pyproject.toml` 或 `setup.py` 包含以下依赖项：

    'cmd2>=3,<4'

`cmd2` 项目使用 [语义化版本控制](https://semver.org)，这意味着任何不兼容的 API 更改都将以新的主版本号发布。公共 API 在 [API 参考](https://cmd2.readthedocs.io/en/latest/api/) 中有文档记录。

我们建议你遵循 Python 打包用户指南中关于 [install_requires](https://packaging.python.org/discussions/install-requires-vs-requirements/) 的建议。通过设置允许版本的上限，你可以确保你的项目不会无意中安装了不兼容的未来版本 `cmd2`。

## 快速示例

以下是一个简单的 `cmd2` 应用程序示例：

```python
from cmd2 import Cmd

class MyApp(Cmd):
    """一个简单的 cmd2 应用示例"""
    
    def do_greet(self, args):
        """问候用户"""
        if args:
            self.poutput(f"你好，{args}！")
        else:
            self.poutput("你好！")
    
    def do_quit(self, args):
        """退出应用程序"""
        return True

if __name__ == '__main__':
    app = MyApp()
    app.cmdloop()
```

## 替代方案

对于不与用户进行连续循环交互的程序——那些只是简单地从命令行接受一组参数、返回结果，而不将用户保持在程序环境中的程序——你只需要 [sys](https://docs.python.org/3/library/sys.html).argv（命令行参数）和 [argparse](https://docs.python.org/3/library/argparse.html)（用于解析 UNIX 风格的选项和标志）。尽管有些人可能更喜欢 [docopt](https://pypi.org/project/docopt) 或 [click](https://click.palletsprojects.com) 而不是 [argparse](https://docs.python.org/3/library/argparse.html)。

[textual](https://textual.textualize.io/) 模块能够构建复杂的全屏终端用户界面（TUI），这些界面不仅限于简单的文本输入和输出；它们可以用选项绘制屏幕，这些选项可以通过光标键甚至鼠标点击来选择。然而，编写 `textual` 应用程序并不像使用 `cmd2` 那样简单直接。

存在几个 Python 包用于构建交互式命令行应用程序，其概念与 [cmd](https://docs.python.org/3/library/cmd.html) 应用程序大致相似。它们都没有 `cmd2` 与 [cmd](https://docs.python.org/3/library/cmd.html) 的紧密联系，但它们可能值得研究。

[Click](https://click.palletsprojects.com) 是一个 Python 包，用于以可组合的方式创建美观的命令行界面，只需尽可能少的代码。它更倾向于命令行实用程序而不是命令行解释器，但两者都可以使用。

要获得一个基于 [Click](https://click.palletsprojects.com) 的工作命令解释器应用程序，需要比 `cmd2` 更多的努力和样板代码。`cmd2` 专注于提供出色的开箱即用体验，尽可能多地内置有用功能，同时要求开发者尽可能少的工作量。我们相信 `cmd2` 为开发者提供了编写命令行解释器的最简单方式，同时为最终用户提供良好的体验。

历史上，[Python Prompt Toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit) 被认为是 `cmd2` 的一个强大但更复杂的替代品。然而，从 4.0.0 版本开始，`cmd2` 在内部使用 `prompt-toolkit` 作为其 REPL 引擎。这意味着你可以获得 `prompt-toolkit` 的强大功能和跨平台兼容性，以及 `cmd2` 易于使用的 API。

如果你正在寻求更丰富的视觉最终用户体验，并且不介意投入更多开发时间，我们建议查看 [Textual](https://github.com/Textualize/textual)，因为它可以用来在终端中构建非常复杂的用户界面，这些界面更类似于功能丰富的 Web GUI。

## 资源

项目相关链接和其他资源：

- [cmd](https://docs.python.org/3/library/cmd.html)
- [cmd2 项目页面](https://github.com/python-cmd2/cmd2)
- [项目错误跟踪器](https://github.com/python-cmd2/cmd2/issues)
- [API 参考](https://cmd2.readthedocs.io/en/latest/api/)
- [入门示例应用](#入门示例)

## 下一步

现在你已经了解了 `cmd2` 的基本概念，可以：

1. 按照上述安装说明安装 `cmd2`
2. 查看[入门示例应用](#入门示例)了解更完整的示例

# 从 cmd 迁移到 cmd2

如果你正在考虑将你的 [cmd](https://docs.python.org/3/library/cmd.html) 应用程序迁移到 `cmd2`，本节将帮助你决定是否适合你的应用程序，并展示如何进行迁移。

## 为什么选择 cmd2

### cmd

[cmd](https://docs.python.org/3/library/cmd.html) 是 Python 标准库中用于创建简单交互式命令行应用程序的模块。[cmd](https://docs.python.org/3/library/cmd.html) 是一个极其简单的框架，有很多不足之处。它甚至不包含内置的退出应用程序的方式！

由于 [cmd](https://docs.python.org/3/library/cmd.html) 提供的 API 是 `cmd2` 的基础，理解 [cmd](https://docs.python.org/3/library/cmd.html) 的使用是学习 `cmd2` 的第一步。阅读完 [cmd](https://docs.python.org/3/library/cmd.html) 文档后，请返回此处了解 `cmd2` 与 [cmd](https://docs.python.org/3/library/cmd.html) 的不同之处。

### cmd2

`cmd2` 是 [cmd](https://docs.python.org/3/library/cmd.html) 的功能齐全的扩展，提供了丰富的功能，使开发者能够更快、更轻松地创建功能丰富、令用户满意的交互式命令行应用程序。

`cmd2` 可以作为 [cmd](https://docs.python.org/3/library/cmd.html) 的直接替代品使用，但有一些小的差异，详见[不兼容性](#不兼容性)部分。只需将 `cmd2` 导入替代 [cmd](https://docs.python.org/3/library/cmd.html) 即可为应用程序添加许多功能，无需进一步修改。迁移到 `cmd2` 还将为开发者提供更多可能性，为用户提供一流的交互式命令行体验。

:::warning
从版本 4.0.0 开始，`cmd2` 不再实际依赖于 `cmd`。`cmd2` 与 `cmd` 大部分 API 兼容。
请参阅[不兼容性](#不兼容性)了解少数已记录的不兼容性。
:::

### 自动功能

从 [cmd](https://docs.python.org/3/library/cmd.html) 切换到 `cmd2` 后，你的应用程序将自动获得以下新功能和能力，无需任何额外工作：

- 更强大的[历史记录](features/history/)。[cmd](https://docs.python.org/3/library/cmd.html) 和 `cmd2` 都提供熟悉的 readline 风格历史记录，但 `cmd2` 使用强大的 [prompt-toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit) 库，提供纯 Python、完全跨平台的体验。此外，`cmd2` 有一个强大的 `history` 命令，允许你在选择的文本编辑器中编辑先前的命令、一次重新运行多个命令、将先前的命令保存为脚本以供稍后执行等等。
- 用户可以将输出重定向到文件或通过管道传递给其他操作系统命令。你记得在所有打印函数中使用 `self.stdout` 而不是 `sys.stdout` 吗？如果是这样，这将开箱即用。如果不是，你需要回去修复它们。在修复之前，你应该考虑 `cmd2` 提供的各种[生成输出](features/embedded_output_help/#生成输出)方式。
- 用户可以加载[脚本文件](features/scripting/)，其中包含要执行的一系列命令。
- 用户可以创建[快捷方式、别名和宏](features/shortcuts_aliases_macros/)来减少重复命令所需的输入。
- [嵌入式 Python 和/或 IPython shell](features/embedded_output_help/#嵌入式-python-shell) 允许用户在你的 `cmd2` 应用程序中执行 Python 代码。
- [剪贴板集成](features/clipboard/) 允许你将命令输出保存到操作系统剪贴板。
- 内置的[计时器](features/misc_packaging/#计时器) 可以显示命令执行所需的时间。

## 不兼容性

`cmd2` 努力与 [cmd](https://docs.python.org/3/library/cmd.html) 保持直接兼容，但也有一些不兼容之处。

### Cmd.emptyline()

[Cmd.emptyline()](https://docs.python.org/3/library/cmd.html#cmd.Cmd.emptyline) 函数在响应提示输入空行时被调用。默认情况下，在 `cmd` 中，如果未覆盖此方法，它会重复并执行最后输入的非空命令。然而，我们遇到的最终用户都不认为这是期望或可取的默认行为。`cmd2` 完全忽略空行，基类 `cmd.emptyline()` 方法永远不会被调用，因此无法覆盖空行行为。

### Cmd.identchars

在 [cmd](https://docs.python.org/3/library/cmd.html) 中，[Cmd.identchars](https://docs.python.org/3/library/cmd.html#cmd.Cmd.identchars) 属性包含命令名称接受的字符字符串。[cmd](https://docs.python.org/3/library/cmd.html) 使用这些字符来分割输入的第一个"单词"，无需用户输入空格。例如，如果 `identchars` 包含所有字母字符的字符串，用户可以输入像 `L20` 这样的命令，它将被解释为命令 `L`，第一个参数为 `20`。

从版本 0.9.0 开始，`cmd2` 忽略了 `identchars`；`cmd2` 中的解析逻辑在空白处分割命令和参数。我们选择这个破坏性更改是因为虽然 [cmd](https://docs.python.org/3/library/cmd.html) 支持 unicode，但在命令名称中使用非 ASCII unicode 字符同时使用 `identchars` 功能可能会有些痛苦。要求使用空格分隔参数也确保了 `cmd2` 许多其他有用功能的可靠运行，包括[制表符补全](features/command_completion_disable/#补全)和[快捷方式、别名和宏](features/shortcuts_aliases_macros/)。

如果你确实需要在应用程序中使用此功能，可以通过编写[解析后钩子](features/hooks/#解析后钩子)来添加回来。

### Cmd.cmdqueue

在 [cmd](https://docs.python.org/3/library/cmd.html) 中，[Cmd.cmdqueue](https://docs.python.org/3/library/cmd.html#cmd.Cmd.cmdqueue) 属性包含排队的输入行列表。当 `cmdloop()` 需要新输入时会检查 cmdqueue 列表；如果非空，其元素将按顺序处理，就像在提示符下输入一样。

从版本 0.9.13 开始，`cmd2` 移除了对 `Cmd.cmdqueue` 的支持。由于 `cmd2` 支持通过主 `cmdloop()`、文本脚本、Python 脚本和历史记录重放运行命令，要在这些方法之间保持一致行为的唯一方法是消除命令队列。此外，没有这个队列的存在，推理应用程序行为要容易得多。

## 最低要求更改

`cmd2.Cmd` 提供了标准库中 [cmd.Cmd](https://docs.python.org/3/library/cmd.html#cmd.Cmd) 的所有相同公共方法和字段，除了 `Cmd.emptyline`（`cmd2` 从不调用此方法）。大多数基于标准库的应用程序只需几分钟即可迁移到 `cmd2`。

### 导入和继承

你需要将导入从：

```py
import cmd
```

改为：

```py
import cmd2
```

然后将类定义从：

```py
class CmdLineApp(cmd.Cmd):
```

改为：

```py
class CmdLineApp(cmd2.Cmd):
```

### 退出

查看你创建的用于退出应用程序的命令。你可能有一个名为 `exit` 的命令，也许还有一个类似的名为 `quit` 的命令。你可能还实现了 `do_EOF()` 方法，以便你的程序像许多操作系统 shell 一样退出。如果所有这些命令都只是退出应用程序，你可以删除它们。请参阅[退出](features/misc_packaging/#退出方式)。

### 分发

如果你要分发应用程序，还需要确保正确安装了 `cmd2`。你需要在 `pyproject.toml` 或 `setup.py` 中添加以下依赖项：

    'cmd2>=3,<4'

## 后续步骤

一旦你当前的应用程序开始使用 `cmd2`，你可以通过利用其他 `cmd2` 功能来扩展功能。以下三个想法可以帮助你入门。

### 参数解析

对于除最简单命令之外的所有命令，使用 [argparse](https://docs.python.org/3/library/argparse.html) 解析用户输入可能比为每个命令手动解析更容易。`cmd2` 提供了一个 `@with_argparser()` 装饰器，它将 `Cmd2ArgumentParser` 对象与你的命令关联。使用此方法将：

1. 将包含参数的 [Namespace](https://docs.python.org/3/library/argparse.html#argparse.Namespace) 传递给你的命令，而不是文本字符串
2. 正确处理来自用户的带引号的字符串输入
3. 基于 `Cmd2ArgumentParser` 为你创建帮助消息
4. 为你的应用程序添加[制表符补全](features/command_completion_disable/#补全)提供巨大帮助
5. 使实现子命令变得更加容易（例如 `git` 有许多子命令，如 `git pull`、`git diff` 等）

如果你想进一步了解，可以查看更多关于[参数处理](features/argument_processing/)的内容。

### 帮助

如果你的应用程序中有许多命令，`cmd2` 可以使用一行装饰器 `@with_category()` 对这些命令进行分类。当用户输入 `help` 时，可用命令将按你指定的类别组织。

如果你已经在使用 `argparse` 或决定切换到它，你可以轻松地[标准化所有帮助消息](features/argument_processing/#帮助消息)，使其由参数解析器生成并由 `cmd2` 显示。不再有与代码实际功能不匹配的帮助消息。

### 生成输出

如果你的程序通过直接打印到 `sys.stdout` 生成输出，你应该考虑切换到 `cmd2.Cmd.poutput`、`cmd2.Cmd.perror` 和 `cmd2.Cmd.pfeedback`。这些方法与几个内置的[设置](features/settings_plugins/)配合使用，允许用户查看或抑制反馈（即进度或状态输出）。它们还根据用户偏好正确处理 ANSI 彩色输出。`cmd2` 对 :simple-rich: [rich](https://github.com/Textualize/rich) 的依赖使你可以轻松地为输出添加颜色和样式。请参阅[彩色输出](features/embedded_output_help/#彩色输出)部分了解更多详情。这些及其他相关主题在[生成输出](features/embedded_output_help/#生成输出)中有介绍。

## 总结

迁移到 `cmd2` 是一个相对简单的过程，可以为你的应用程序带来许多强大的功能。通过遵循本指南中的步骤，你可以快速将现有的 `cmd` 应用程序升级为功能更强大的 `cmd2` 应用程序，为用户提供更好的交互体验。

# 入门示例

以下是简单的 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例应用程序的快速演练，它展示了 `cmd2` 的许多功能：

- [设置](features/settings_plugins/)
- [命令](features/command_completion_disable/)
- [参数处理](features/argument_processing/)
- [生成输出](features/embedded_output_help/#生成输出)
- [帮助](features/embedded_output_help/#帮助)
- [快捷方式](features/shortcuts_aliases_macros/#快捷方式)
- [多行命令](features/app_setup/#多行命令)
- [历史记录](features/history/)
- [底部工具栏](features/prompt_redirection/#底部工具栏)

如果你不想边看边输入，这里是完整的源代码（你可以点击展开，然后点击右上角的 **Copy** 按钮）：
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

## 基本应用程序

首先，我们需要创建一个新的 `cmd2` 应用程序。创建一个新文件 `getting_started.py`，内容如下：

```py
#!/usr/bin/env python
"""一个基本的 cmd2 应用程序。"""
import cmd2


class BasicApp(cmd2.Cmd):
    """展示许多常见功能的 Cmd2 应用程序。"""


if __name__ == '__main__':
    import sys
    app = BasicApp()
    sys.exit(app.cmdloop())
```

我们有一个新类 `BasicApp`，它是 `cmd2.Cmd` 的子类。当我们告诉 Python 运行我们的文件时：

```shell
$ python getting_started.py
```

应用程序会创建我们类的一个实例，并调用 `cmd2.Cmd.cmdloop` 方法。此方法接受用户输入并根据该输入运行命令。由于我们继承了 `cmd2.Cmd`，我们的新应用程序已经具有许多内置功能。

恭喜，你有了一个可以工作的 `cmd2` 应用程序。你可以运行它，然后输入 `quit` 退出。

## 创建新设置

在创建第一个命令之前，我们将向此应用程序添加一个新设置。`cmd2` 对[设置](features/settings_plugins/#设置)有强大的支持。你在对象初始化期间配置设置，因此我们需要为类添加一个初始化器：

```py
def __init__(self):
    super().__init__()

    # 使 maxrepeats 可在运行时设置
    self.maxrepeats = 3
    self.add_settable(cmd2.Settable('maxrepeats', int, 'speak 命令的最大重复次数', self))
```

在初始化器中，首先要确保我们初始化了 `cmd2`。这就是 `super().__init__()` 这行代码的作用。接下来创建一个属性来保存设置。最后，使用 `cmd2.utils.Settable` 类的新实例调用 `cmd2.Cmd.add_settable` 方法。现在如果你运行脚本，并输入 `set` 命令查看设置：

```shell
$ python getting_started.py
(Cmd) set
```

你会看到我们的 `maxrepeats` 设置显示出来，默认值为 `3`。

## 创建命令

现在我们将创建第一个命令，称为 `speak`，它将回显我们告诉它说的内容。我们将使用[参数处理器](features/argument_processing/)，以便 `speak` 命令可以大喊和说 Pig Latin。我们还将使用一些内置方法来[生成输出](features/embedded_output_help/#生成输出)。将此代码添加到 `getting_started.py`，使 `speak_parser` 属性和 `do_speak()` 方法成为 `BasicApp()` 类的一部分：

```py
speak_parser = cmd2.Cmd2ArgumentParser()
speak_parser.add_argument('-p', '--piglatin', action='store_true', help='atinLay')
speak_parser.add_argument('-s', '--shout', action='store_true', help='N00B EMULATION MODE')
speak_parser.add_argument('-r', '--repeat', type=int, help='输出 [n] 次')
speak_parser.add_argument('words', nargs='+', help='要说的话')

@cmd2.with_argparser(speak_parser)
def do_speak(self, args):
    """重复你告诉我的内容。"""
    words = []
    for word in args.words:
        if args.piglatin:
            word = '%s%say' % (word[1:], word[0])
        if args.shout:
            word = word.upper()
        words.append(word)
    repetitions = args.repeat or 1
    for _ in range(min(repetitions, self.maxrepeats)):
        # .poutput 处理换行符，并适应输出重定向
        self.poutput(' '.join(words))
```

在脚本顶部，你还需要添加：

```py
import argparse
```

这里有很多内容需要解释，让我们逐步分析。我们创建了 `speak_parser`，它使用 Python 标准库中的 [argparse](https://docs.python.org/3/library/argparse.html) 模块来解析用户的命令行输入。到目前为止，还没有任何特定于 `cmd2` 的内容。

还有一个新方法叫做 `do_speak()`。在 [cmd](https://docs.python.org/3/library/cmd.html) 和 `cmd2` 中，以 `do_` 开头的方法会成为新命令，因此通过定义此方法，我们创建了一个名为 `speak` 的命令。

注意 `do_speak()` 方法上的 `cmd2.decorators.with_argparser` 装饰器。此装饰器为我们做了 3 件有用的事情：

1. 它告诉 `cmd2` 使用我们定义的 argparser 处理 `speak` 命令的所有输入。如果用户输入不符合 argparser 定义的要求，将向用户显示错误。
2. 它修改了我们的 `do_speak` 方法，使我们接收来自参数解析器的命名空间，而不是接收原始用户输入作为参数。
3. 它基于 argparser 为我们创建帮助消息。

你可以在方法体中看到我们如何使用来自 argparser 的命名空间（作为变量 `args` 传入）。我们构建了一个要输出的单词列表，同时支持 `--piglatin` 和 `--shout` 选项。

在方法末尾，我们使用 `maxrepeats` 设置作为打印输出次数的上限。

你会注意到的最后一件事是我们使用 `self.poutput()` 方法来显示输出。`poutput()` 是 `cmd2` 提供的方法，我强烈建议你在想要[生成输出](features/embedded_output_help/#生成输出)时随时使用它。它提供以下好处：

1. 允许用户将输出重定向到文本文件或通过管道传递给 shell 进程
2. 为重定向的输出优雅地处理 `BrokenPipeError` 异常
3. 根据设置[剥离嵌入的 ANSI 序列](features/settings_plugins/#allow_style)（通常用于背景和前景颜色）

再次运行脚本，尝试 `speak` 命令。尝试输入 `help speak`，你会看到一个漂亮的消息，描述了该命令的各种选项。

通过这几行代码，我们创建了一个[命令](features/command_completion_disable/#命令)，使用了[参数处理器](features/argument_processing/)，为用户添加了漂亮的[帮助消息](features/embedded_output_help/#帮助)，并[生成了一些输出](features/embedded_output_help/#生成输出)。

## 快捷方式

`cmd2` 有多种功能来简化重复的用户输入：[快捷方式、别名和宏](features/shortcuts_aliases_macros/)。让我们为应用程序添加一个快捷方式。快捷方式是可以替代命令名称使用的字符串。例如，`cmd2` 支持快捷方式 `!`，它运行 `shell` 命令。所以不用输入：

```shell
(Cmd) shell ls -al
```

你可以输入：

```shell
(Cmd) !ls -al
```

让我们为 `speak` 命令添加一个快捷方式。修改 `__init__()` 方法，使其如下所示：

```py
def __init__(self):
    shortcuts = cmd2.DEFAULT_SHORTCUTS
    shortcuts.update({'&': 'speak'})
    super().__init__(shortcuts=shortcuts)

    # 使 maxrepeats 可在运行时设置
    self.maxrepeats = 3
    self.add_settable(cmd2.Settable('maxrepeats', int, 'speak 命令的最大重复次数', self))
```

快捷方式传递给 `cmd2` 初始化器，如果你想要 `cmd2` 的内置快捷方式，你必须传递它们。这些快捷方式定义为字典，键是快捷方式，值包含命令。使用默认快捷方式并添加自己的快捷方式时，最好使用 `.update()` 方法修改字典。这样，如果你添加的快捷方式恰好已经在默认集中，你的将覆盖默认的，并且在运行时不会出现任何错误。

再次运行你的应用程序，输入：

```shell
(Cmd) shortcuts
```

查看所有快捷方式的列表，包括我们刚刚创建的 speak 的快捷方式。

## 多行命令

一些用例受益于跨越多行的命令。例如，你可能希望用户能够输入 SQL 命令，这些命令通常跨越多行并以分号结尾。让我们为应用程序添加一个多行命令。首先我们将创建一个名为 `orate` 的新命令。此代码同时显示了 `speak` 命令和 `orate` 命令的定义：

```py
@cmd2.with_argparser(speak_parser)
def do_speak(self, args):
    """重复你告诉我的内容。"""
    words = []
    for word in args.words:
        if args.piglatin:
            word = '%s%say' % (word[1:], word[0])
        if args.shout:
            word = word.upper()
        words.append(word)
    repetitions = args.repeat or 1
    for _ in range(min(repetitions, self.maxrepeats)):
        # .poutput 处理换行符，并适应输出重定向
        self.poutput(' '.join(words))

# orate 是 speak 的同义词，接受多行输入
do_orate = do_speak
```

创建新命令后，我们需要告诉 `cmd2` 将该命令视为多行命令。修改 super 初始化行，使其如下所示：

```py
super().__init__(multiline_commands=['orate'], shortcuts=shortcuts)
```

现在当你运行示例时，你可以输入类似这样的内容：

```text
(Cmd) orate O for a Muse of fire, that would ascend
> The brightest heaven of invention,
> A kingdom for a stage, princes to act
> And monarchs to behold the swelling scene! ;
```

注意提示符会更改以指示输入仍在进行中。`cmd2` 将继续提示输入，直到看到未加引号的分号（默认的多行命令终止字符）。

## 历史记录

`cmd2` 跟踪用户输入的命令历史。作为开发者，你不需要做任何事情来启用此功能，你可以免费获得它。如果你希望命令历史在应用程序调用之间持久存在，你需要做一些工作。[历史记录](features/history/)页面有所有详细信息。

用户可以使用两种方法访问命令历史：

- [prompt-toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit) 库，它提供了 [GNU readline 库](https://en.wikipedia.org/wiki/GNU_Readline)的纯 Python 替代品，完全跨平台兼容
- `cmd2` 内置的 `history` 命令

在基于 `cmd2` 的应用程序的提示符中，你可以按 `Control-p` 移动到先前输入的命令，按 `Control-n` 移动到下一个命令。你还可以使用 `Control-r` 搜索命令历史。

默认情况下，`prompt-toolkit` 提供 Emacs 风格的键绑定，这对 GNU Readline 库的用户来说很熟悉。你可以参考 [readline 速查表](http://readline.kablamo.org/emacs.html)或深入了解 [Prompt Toolkit 用户手册](https://python-prompt-toolkit.readthedocs.io/en/stable/pages/advanced_topics/key_bindings.html)了解所有详细信息，包括自定义键绑定的说明。

`history` 命令允许用户查看命令历史，并按编号、范围、字符串搜索或正则表达式从历史中选择命令。使用选定的命令，用户可以：

- 重新运行命令
- 在文本编辑器中编辑选定的命令，并在文本编辑器退出后运行它们
- 将命令保存到文件
- 运行命令，同时将命令及其输出保存到文件

在任何 `cmd2` 输入提示符中输入 `history -h` 了解有关 `history` 命令的更多信息，或探索[用户的命令历史](features/history/#用户指南)。

## 总结

你刚刚创建了一个简单但功能齐全的命令行应用程序。通过最少的工作，应用程序利用了 `cmd2` 的许多强大功能。要了解更多信息，你可以：

- 深入了解 `cmd2` 提供的所有[功能特性](/archive/?tag=cmd2)
- 浏览 [API 参考](https://cmd2.readthedocs.io/en/latest/api/)

# 替代事件循环

在本文档中，我们专注于 90% 的用例，我们认为这适用于超过 90% 的用户群。这侧重于易用性和最佳开箱即用体验，开发者以最少的工作量获得最多功能。我们说的是使用 `cmdloop()` 方法运行 `cmd2` 应用程序：

```py
from cmd2 import Cmd

class App(Cmd):
    # 自定义属性和方法
    pass

app = App()
app.cmdloop()
```

然而，这种使用 `cmd2` 的方式有一些限制，主要是 `cmd2` 拥有程序的内部循环。这可能会不必要地限制使用依赖于控制自己事件循环的库。

许多 Python 并发库涉及或需要它们控制的事件循环，例如 [asyncio](https://docs.python.org/3/library/asyncio.html)、[gevent](http://www.gevent.org/)、[Twisted](https://twistedmatrix.com) 等。

:::warning
从版本 **4.0** 开始，`cmd2` 依赖于 `prompt-toolkit`，而后者又[原生使用 asyncio](https://python-prompt-toolkit.readthedocs.io/en/stable/pages/advanced_topics/asyncio.html)并启动自己的 `asyncio` 事件循环。

[async_call.py](https://github.com/python-cmd2/cmd2/blob/main/examples/async_call.py) 示例展示了如何从 cmd2 命令调用异步函数。
:::

`cmd2` 应用程序可以通过使用以下代码以 `cmd2` 不拥有程序主循环的方式执行：

```py
import cmd2

class Cmd2EventBased(cmd2.Cmd):
    def __init__(self):
        cmd2.Cmd.__init__(self)

    # ... 你的类代码 ...

if __name__ == '__main__':
    app = Cmd2EventBased()
    app.preloop()

    # 在你希望运行单个命令的任何事件循环机制中执行此操作
    cmd_line_text = "help history"
    app.runcmds_plus_hooks([cmd_line_text])

    app.postloop()
```

`cmd2.Cmd.runcmds_plus_hooks` 方法运行多个命令，其中每个单独的命令通过 `cmd2.Cmd.onecmd_plus_hooks` 执行。

`cmd2.Cmd.onecmd_plus_hooks` 方法将执行以下操作来以正常方式执行单个命令：

1. 将用户输入解析为 `cmd2.Statement` 对象
2. 调用使用 `cmd2.Cmd.register_postparsing_hook` 注册的方法
3. 如果用户请求且允许，重定向输出
4. 启动计时器
5. 调用使用 `cmd2.Cmd.register_precmd_hook` 注册的方法
6. 调用 `cmd2.Cmd.precmd` - 为了与 `cmd` 向后兼容
7. 将语句添加到[历史记录](features/history/)
8. 调用 `do_command` 方法
9. 调用使用 `cmd2.Cmd.register_postcmd_hook` 注册的方法
10. 调用 `cmd2.Cmd.postcmd` - 为了与 `cmd` 向后兼容
11. 停止计时器并显示经过的时间
12. 如果输出被重定向，停止重定向输出
13. 调用使用 `cmd2.Cmd.register_cmdfinalization_hook` 注册的方法

以这种方式运行可以实现与外部事件循环的集成。然而，如何与任何特定事件循环集成超出了本文档的范围。请注意，以这种方式运行有几个缺点，包括：

- 要求开发者编写更多代码
- 不允许通过命令行参数在调用时运行命令
