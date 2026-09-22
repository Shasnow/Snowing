---
title: Python cmd2：嵌入式 Python Shell、生成输出与帮助系统
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的嵌入式 Python/IPython Shell、输出生成方法（包括分页、着色、对齐）以及帮助系统和命令分类功能。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 嵌入式 Python Shell

## Python（可选）

如果 `cmd2.Cmd` 类在实例化时传入 `include_py=True`，则可选的 `py` 命令将可用，并启动一个交互式 Python shell：

```py
from cmd2 import Cmd
class App(Cmd):
    def __init__(self):
        Cmd.__init__(self, include_py=True)
```

Python shell 可以使用 `self.pyscript_name` 中命名的对象（默认为 `app`）来运行你应用中的 CLI 命令。这个封装提供了在你的 `cmd2` 应用中执行命令的能力，同时与完整的 `Cmd` 实例保持隔离。例如，可以使用 `app("command ...")` 来运行任何应用命令。

你可以选择通过将 `self.self_in_py` 设为 `True` 来启用对应用的完全访问。启用此标志会将 `self` 添加到 Python 会话中，它是对你的 `cmd2` 应用的引用。这对调试应用非常有用。

在 Python 会话中创建的任何局部或全局变量不会在 CLI 环境中持久存在。

`self.py_locals` 中的所有内容在 Python 环境中始终可用。

所有这些参数也可通过 `run_pyscript` 命令运行的 Python 脚本使用：

- 支持文件系统路径的 Tab 补全
- 能够向调用的脚本传递命令行参数

此命令提供了比简单文本文件脚本更复杂和强大的脚本能力。Python 脚本可以包含条件控制流逻辑。参阅 [python_scripting.py](https://github.com/python-cmd2/cmd2/blob/main/examples/python_scripting.py) 和 [conditional.py](https://github.com/python-cmd2/cmd2/blob/main/examples/scripts/conditional.py) 脚本了解如何在自己的应用中实现这一点。参阅[脚本](../scripting/)了解 `cmd2` 应用中两种脚本方法的说明。

以下是使用 `run_pyscript` 的简单示例以及 [arg_printer](https://github.com/python-cmd2/cmd2/blob/main/examples/scripts/arg_printer.py) 脚本：

```sh
(Cmd) run_pyscript examples/scripts/arg_printer.py foo bar baz
Running Python script 'arg_printer.py' which was called with 3 arguments
arg 1: 'foo'
arg 2: 'bar'
arg 3: 'baz'
```

## IPython（可选）

**如果**系统中安装了 [IPython](http://ipython.readthedocs.io)，**并且** `cmd2.Cmd` 类在实例化时传入 `include_ipy=True`，则可选的 `ipy` 命令将启动一个交互式 IPython shell：

```py
from cmd2 import Cmd
class App(Cmd):
    def __init__(self):
        Cmd.__init__(self, include_ipy=True)
```

`ipy` 命令进入一个交互式 [IPython](http://ipython.readthedocs.io) 会话。与交互式 Python 会话类似，如果 `self.self_in_py` 为 `True`，此 shell 可以通过 `self` 访问你的应用实例，并且通过 `self` 对应用所做的任何更改都将持久存在。但是，在 `ipy` shell 中创建的任何局部或全局变量不会在 CLI 环境中持久存在。

同样，与交互式 Python 会话一样，`ipy` shell 可以访问 `self.py_locals` 的内容，并可以使用 `app` 对象（或你的自定义名称）回调应用。

[IPython](http://ipython.readthedocs.io) 提供了许多优势，包括：

> - 全面的对象自省
> - 使用 `?` 获取对象帮助
> - 可扩展的 Tab 补全，默认支持 Python 变量和关键字的补全
> - 优秀的内置 [ipdb](https://pypi.org/project/ipdb/) 调试器

对象自省和 Tab 补全使 IPython 在调试以及交互式实验和数据分析方面特别高效。

# 生成输出

标准 `cmd` 应用可以使用以下任一方法产生输出：

```py
print("Greetings, Professor Falken.", file=self.stdout)
self.stdout.write("Shall we play a game?\n")
```

虽然你可以直接将输出发送到 `sys.stdout`，但 `cmd2.Cmd` 可以使用 `stdin` 和 `stdout` 变量进行初始化，它们会存储为 `self.stdin` 和 `self.stdout`。通过每次产生输出时使用这些变量，你可以通过更改类的初始化方式来轻松改变所有输出的目标位置。

`cmd2.Cmd` 以多种便捷方式扩展了这种方法。参阅[输出重定向和管道](../prompt_redirection/#输出重定向与管道)了解用户如何更改命令输出的发送位置。为了让这些功能正常工作，你生成的输出必须发送到 `self.stdout`。你可以使用上述方法，一切都会正常工作。`cmd2.Cmd` 还包含许多输出相关的方法，你可以用来增强应用产生的输出。

由于 `cmd2` 依赖于 [rich](https://github.com/Textualize/rich) 库，以下 `cmd2.Cmd` 输出方法可以原生渲染 rich 的[可渲染对象](https://rich.readthedocs.io/en/latest/protocol.html)，实现美观复杂的输出：

- `poutput`
- `perror`
- `psuccess`
- `pwarning`
- `pfeedback`
- `ppaged`

:::tip
**高级输出自定义**
上述每个方法都接受额外的可选参数来帮助控制输出格式：

- `sep`：在打印文本之间写入的字符串，默认为 " "
- `end`：在打印文本末尾写入的字符串，默认为换行符
- `style`：应用于输出的可选样式
- `soft_wrap`：启用软换行模式。如果为 True，文本行不会被自动换行或裁剪以适应终端宽度。默认为 True
- `justify`：对齐方式（"left"、"center"、"right"、"full"）。默认为 None
- `emoji`：如果为 True，Rich 会将 emoji 代码（如 😃）替换为对应的 Unicode 字符。默认为 False
- `markup`：如果为 True，Rich 会将带有标签的字符串（如 `[bold]hello[/bold]`）解释为带样式的输出。默认为 False
- `highlight`：如果为 True，Rich 会自动对字符串中的元素应用高亮，如常见的 Python 数据类型（数字、布尔值或 None）
- `rich_print_kwargs`：传递给 `console.print()` 的可选额外关键字参数
- `kwargs`：用于支持扩展 print 方法的任意关键字参数，不会传递给 `console.print()`
:::

## 普通输出

`poutput` 方法类似于 Python 内置的 [print](https://docs.python.org/3/library/functions.html#print) 函数。`poutput` 添加了一些便利功能：

1. 由于用户可以将输出管道传递给 shell 命令，它会捕获 `BrokenPipeError` 并将 `self.broken_pipe_warning` 的内容输出到 `stderr`。`self.broken_pipe_warning` 默认为空字符串，所以此方法只会吞掉异常。如果你想显示错误消息，请在初始化 `cmd2.Cmd` 时将其放入 `self.broken_pipe_warning` 中。
2. 它会检查并遵守 [allow_style](../settings_plugins/#allow_style) 设置。详见下方[彩色输出](#彩色输出)。
3. 它允许打印任意 rich 可渲染对象，可以实现视觉上非常复杂的效果。

以下是一个展示此方法用法的简单命令：

```py
def do_echo(self, args):
    """A simple command showing how poutput() works"""
    self.poutput(args)
```

## 错误消息

当程序中发生错误时，你可以通过调用 `perror` 方法在 `sys.stderr` 上显示它。默认情况下，此方法会将 `Cmd2Style.ERROR` 应用于输出。

## 警告消息

`pwarning` 与 `cmd2.Cmd.perror` 类似，但会将 `Cmd2Style.WARNING` 应用于输出。

## 反馈信息

你可能需要向用户显示不属于生成输出的信息。这可能是调试信息或长时间运行命令的进度信息。它不是输出，不是错误消息，而是状态信息。如果你使用 [Timing](../settings_plugins/#timing) 设置，命令运行耗时的输出将以无样式的文本输出到 `stderr`。你可以通过将 `style=None` 传递给 `perror` 方法来产生此类输出。

如果 [quiet](../settings_plugins/#quiet) 设置为 `True`，则调用 `cmd2.Cmd.pfeedback` 不会产生任何输出。如果 `quiet` 为 `False`，`pfeedback` 方法将输出发送到 `stdout`。因此，`pfeedback` 适用于你希望在 `quiet` 为 `True` 时能够静音的非必要输出。

## 异常

如果你的应用捕获了异常并希望向用户显示异常信息，`pexcept` 方法可以提供帮助。默认行为是只显示异常中包含的消息。但是，如果 [debug](../settings_plugins/#debug) 设置为 `True`，则会显示整个堆栈跟踪。

## 分页输出

如果你知道将要生成大量输出，你可能希望以用户可以前后滚动的方式显示它。如果你将所有要显示的输出传递给 `ppaged` 的单次调用，它将被管道传递给操作系统适当的 shell 命令来分页输出。在 Windows 上，输出被管道传递给 [more](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/more)；在 macOS 和 Linux 等类 Unix 操作系统上，输出被管道传递给 [less](https://man7.org/linux/man-pages/man1/less.1.html)。

## 彩色输出

你可以在输出中添加自己的 [ANSI 转义序列](https://en.wikipedia.org/wiki/ANSI_escape_code#Colors)来指示终端更改前景色和背景色。

`cmd2` 提供了许多便捷函数和类来为文本添加颜色和其他样式。这些都基于 [rich](https://github.com/Textualize/rich)，文档在以下章节中：

- `cmd2.colors`
- `cmd2.rich_utils`
- `cmd2.string_utils`

[color.py](https://github.com/python-cmd2/cmd2/blob/main/examples/color.py) 示例演示了 `cmd2` 应用中所有可用的颜色。

### 自定义主题

`cmd2` 使用 rich 的 [Theme](https://rich.readthedocs.io/en/stable/reference/theme.html) 对象来定义各种 UI 元素的样式。你可以使用 `cmd2.theme.update_theme` 定义自己的自定义主题。参阅 [rich_theme.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_theme.py) 示例了解更多信息。

在向输出添加所需的转义序列后，你应该使用以下方法之一来向用户呈现输出：

- `cmd2.Cmd.poutput`
- `cmd2.Cmd.perror`
- `cmd2.Cmd.pwarning`
- `cmd2.Cmd.pexcept`
- `cmd2.Cmd.pfeedback`
- `cmd2.Cmd.ppaged`

这些方法都遵守 [allow_style](../settings_plugins/#allow_style) 设置，用户可以修改该设置来控制这些转义码是否传递到终端。

## 文本对齐

如果你想生成在指定宽度或终端宽度内左对齐、居中或右对齐的输出，以下函数可以提供帮助：

- `cmd2.string_utils.align_left`
- `cmd2.string_utils.align_center`
- `cmd2.string_utils.align_right`

这些函数与 Python 的字符串对齐函数不同，它们支持显示宽度大于 1 的字符。此外，ANSI 样式序列会被安全忽略，不计入显示宽度。这意味着支持彩色文本。如果文本包含换行符，则每行独立对齐。

:::tip
**高级对齐自定义**
你也可以在调用 `cmd2` 的 print 方法时使用 `justify` 参数来控制输出对齐方式。
:::

## 多列输出

在生成多列输出时，你通常需要计算每个项目的宽度以便用空格适当填充。然而，有些 Unicode 字符占用 2 个单元格，而另一些占用 0 个。更复杂的是，你可能在输出中包含了 ANSI 转义序列来在终端上生成颜色。

`cmd2.string_utils.str_width` 函数解决了这两个问题。给它传递一个字符串，无论它包含哪些 Unicode 字符和 ANSI 文本样式转义序列，它都会告诉你该字符串在屏幕上打印时会占用多少个字符。

# 帮助

根据我们的经验，终端用户很少阅读文档，无论文档多么高质量或有用。因此，在应用中提供良好的内置帮助非常重要。幸运的是，`cmd2` 使这变得简单。

## 获取帮助

`cmd2` 使终端用户可以通过内置的 `help` 命令轻松获取帮助。单独的 `help` 命令显示可用命令的列表：

```text
(Cmd) help

Documented Commands
 alias  help     ipy    py    run_pyscript  set    shortcuts
 edit   history  macro  quit  run_script    shell
```

`help` 命令也可用于提供特定命令的详细帮助：

```text
(Cmd) help quit
Usage: quit [-h]

Exit this application.

Optional Arguments:
  -h, --help  show this help message and exit
```

## 提供帮助

`cmd2` 使 `cmd2` 应用的开发者可以轻松提供帮助。默认情况下，命令的帮助是定义命令的 `do_*` 方法的文档字符串——例如，对于命令 **foo**，该命令通过定义 `do_foo` 方法实现，该方法的文档字符串就是帮助内容。

对于使用 `@with_argparser` 装饰器解析参数的命令，帮助由 `argparse` 提供。详见[帮助消息](../argument_processing/#help-messages)。

偶尔可能会出现不寻常的情况，提供静态帮助文本不够好，你希望在命令的帮助文本中提供动态信息。为满足此需求，如果定义了与 `do_foo` 方法匹配的 `help_foo` 方法，则该方法将用于提供命令 **foo** 的帮助。此动态帮助仅适用于不使用 `argparse` 装饰器的命令，因为我们不希望 `help cmd` 和 `cmd -h` 的输出不同。

## 命令分类

在 `cmd2` 中，`help` 命令将其输出组织为分类。每个命令属于一个分类，显示由 `DEFAULT_CATEGORY` 类变量驱动。

有 3 种指定命令分类的方法：

1. 使用 `DEFAULT_CATEGORY` 类变量（自动）
2. 使用 `@with_category` 装饰器（手动）
3. 使用 `categorize()` 函数（手动）

### 自动分类

对命令进行分类最有效的方式是在 `Cmd` 或 `CommandSet` 类中定义 `DEFAULT_CATEGORY` 类变量。在该类中定义的任何没有显式分类覆盖的命令将自动归入此分类。

默认情况下，`cmd2.Cmd` 将其 `DEFAULT_CATEGORY` 定义为 `"Cmd2 Commands"`。

```py
class MyApp(cmd2.Cmd):
    # All commands defined in this class will be grouped here
    DEFAULT_CATEGORY = 'Application Commands'

    def do_echo(self, arg):
        """Echo command"""
        self.poutput(arg)
```

这也适用于[命令集](../modular_commands/#命令集)：

```py
class Plugin(cmd2.CommandSet):
    DEFAULT_CATEGORY = 'Plugin Commands'

    def do_plugin_cmd(self, _):
        """Plugin command"""
        self._cmd.poutput('Plugin')
```

使用继承时，`cmd2` 使用命令实际定义所在类的 `DEFAULT_CATEGORY`。这意味着内置命令（如 `help`、`history` 和 `quit`）保留在 `"Cmd2 Commands"` 分类中，而你的命令移到你的自定义分类中。

如果你想重命名内置分类本身，可以在 `Cmd` 子类中通过在类级别重新赋值 `cmd2.Cmd.DEFAULT_CATEGORY` 来实现：

```py
class MyApp(cmd2.Cmd):
    # Rename the framework's built-in category
    cmd2.Cmd.DEFAULT_CATEGORY = 'Shell Commands'

    # Set the category for your own commands
    DEFAULT_CATEGORY = 'Application Commands'
```

有关此功能的完整演示，请参阅 [default_categories.py](https://github.com/python-cmd2/cmd2/blob/main/examples/default_categories.py) 示例。

### 手动分类

如果需要将单个命令移到与类默认不同的分类中，可以使用 `@with_category` 装饰器或 `categorize()` 函数。这些手动设置始终优先于 `DEFAULT_CATEGORY`。

使用 `@with_category` 装饰器：

```py
@with_category('Connecting')
def do_which(self, _):
    """Which command"""
    self.poutput('Which')
```

使用 `categorize()` 函数：

可以对单个函数调用：

```py
def do_connect(self, _):
    """Connect command"""
    self.poutput('Connect')

# Tag the above command functions under the category Connecting
categorize(do_connect, CMD_CAT_CONNECTING)
```

或者对函数的可迭代容器调用：

```py
def do_undeploy(self, _):
    """Undeploy command"""
    self.poutput('Undeploy')

def do_stop(self, _):
    """Stop command"""
    self.poutput('Stop')

def do_findleakers(self, _):
    """Find Leakers command"""
    self.poutput('Find Leakers')

# Tag the above command functions under the category Application Management
categorize((do_undeploy,
            do_stop,
            do_findleakers), CMD_CAT_APP_MGMT)
```

`help` 命令还有一个详细选项（`help -v` 或 `help --verbose`），它将帮助分类与每个命令的帮助消息结合在一起：

    Application Management
    ─────────────────────────────────────
    Name          Description
    ─────────────────────────────────────
    deploy        Deploy command.
    expire        Expire command.
    findleakers   Find Leakers command.
    list          List command.
    redeploy      Redeploy command.
    restart       Restart command.
    sessions      Sessions command.
    start         Start command.
    stop          Stop command.
    undeploy      Undeploy command.


    Command Management
    ─────────────────────────────────────────────────────────────────
    Name               Description
    ─────────────────────────────────────────────────────────────────
    disable_commands   Disable the Application Management commands.
    enable_commands    Enable the Application Management commands.


    Connecting
    ────────────────────────────
    Name      Description
    ────────────────────────────
    connect   Connect command.
    which     Which command.


    Server Information
    ─────────────────────────────────────────────────────────────────────────────────────────────────
    Name                  Description
    ─────────────────────────────────────────────────────────────────────────────────────────────────
    resources             Resources command.
    serverinfo            Server Info command.
    sslconnectorciphers   SSL Connector Ciphers command is an example of a command that contains
                        multiple lines of help information for the user. Each line of help in a
                        contiguous set of lines will be printed and aligned in the verbose output
                        provided with 'help --verbose'.
    status                Status command.
    thread_dump           Thread Dump command.
    vminfo                VM Info command.


    Other
    ─────────────────────────────────────────────────────────────────────────────────────────
    Name           Description
    ─────────────────────────────────────────────────────────────────────────────────────────
    alias          Manage aliases.
    config         Config command.
    edit           Run a text editor and optionally open a file with it.
    help           List available commands or provide detailed help for a specific command.
    history        View, run, edit, save, or clear previously entered commands.
    macro          Manage macros.
    quit           Exit this application.
    run_pyscript   Run Python script within this application's environment.
    run_script     Run text script.
    set            Set a settable parameter or show current settings of parameters.
    shell          Execute a command as if at the OS prompt.
    shortcuts      List available shortcuts.
    version        Version command.

When called with the `-v` flag for verbose help, the one-line description for each command is
provided by the first line of the docstring for that command's associated `do_*` method.
