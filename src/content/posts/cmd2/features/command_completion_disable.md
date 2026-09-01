---
title: Python cmd2：命令、补全与禁用命令
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的命令创建、Tab 补全机制以及命令的禁用、隐藏和移除功能。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 命令

`cmd2` 的设计目标是让你能够轻松创建新命令。这些命令构成了应用程序的核心。如果你最初是使用 [cmd](https://docs.python.org/3/library/cmd.html) 来编写应用，那么当你迁移到 `cmd2` 时，你已构建的所有命令都将正常工作。然而，`cmd2` 提供了更多功能，你可以利用它们为命令添加更强大的特性，使命令更容易编写。在介绍所有精彩功能之前，让我们先简要讨论如何在应用中创建新命令。

## 基本命令

最简单的 `cmd2` 应用如下所示：

```py
#!/usr/bin/env python
"""A simple cmd2 application."""
import cmd2


class App(cmd2.Cmd):
    """A simple cmd2 application."""


if __name__ == '__main__':
    import sys
    c = App()
    sys.exit(c.cmdloop())
```

这个应用继承了 `cmd2.Cmd`，但没有自己的代码，因此所有功能（而且相当丰富）都是继承来的。让我们在这个应用中创建一个名为 `echo` 的简单命令，它输出传入的所有参数。将以下方法添加到类中：

```py
def do_echo(self, line):
    self.poutput(line)
```

当你在 `cmd2` 提示符下输入内容时，第一个以空格分隔的单词会被视为命令名。`cmd2` 会查找名为 `do_命令名` 的方法。如果存在，就调用该方法，并将用户输入的其余部分作为第一个参数传递。如果不存在，`cmd2` 会打印错误消息。由于这种行为，创建新命令只需在类中定义一个具有适当名称的新方法即可。这与使用 Python 标准库中的 [cmd](https://docs.python.org/3/library/cmd.html) 模块创建命令的方式完全一致。

:::note
如果你不熟悉 `poutput()` 方法，请参阅[生成输出](../embedded_output_help/#生成输出)。
:::

## 语句对象

一个命令接收一个参数：包含用户输入其余部分的字符串。然而，在 `cmd2` 中，这个字符串实际上是一个 `cmd2.Statement` 对象，它是 `str` 的子类，以保持与 `cmd` 的向后兼容性。

`cmd2` 比 [cmd](https://docs.python.org/3/library/cmd.html) 模块拥有更复杂的解析引擎。此解析处理包括：

- 带引号的参数
- 输出重定向和管道
- 多行命令
- 快捷方式、别名和宏展开

除了从用户输入中解析所有这些元素外，`cmd2` 还有代码来使这些功能正常工作——对你和你在应用中编写的命令来说几乎是透明的。通过将 `Statement` 对象（而非普通字符串）传递给你的命令，你可以了解 `cmd2` 在命令接收到用户输入之前对其做了什么处理。你还可以避免编写大量解析代码，因为 `cmd2` 让你可以访问它已经解析好的内容。

`Statement` 对象是 `str` 的子类，包含以下属性：

**command**

    调用的命令名称。你已经通过 `cmd2` 调用的方法知道了这一点，
    但有时将其作为字符串会很方便，例如当你想在错误消息中包含命令名称时。

**args**

    包含命令参数的字符串，已移除输出重定向或到 shell 命令的管道部分。
    实际上，`Statement` 对象的"字符串"值也已移除了所有输出重定向和管道子句。引号保留在字符串中。

**command_and_args**

    仅包含命令和参数的字符串，已移除输出重定向或到 shell 命令的管道部分。

**argv**

    类似 `sys.argv` 的参数列表，包括作为 `argv[0]` 的命令和作为列表后续项的参数。
    参数周围的引号将被去除，命令中的输出重定向或管道部分也会被去除。

**raw**

    用户输入的完整内容，原样保留。

**terminator**

    用于结束多行命令的字符。
    你可以配置多个终止字符，此属性会告诉你用户输入的是哪一个。

对于许多简单命令（如上面的 `echo` 命令），你可以忽略 `Statement` 对象及其所有属性，直接将传递的值作为字符串使用。你也可以选择使用 `argv` 属性来进行更复杂的参数处理。在你深入之前，应该先查看 `cmd2` 提供的[参数处理](../argument_processing/)功能。

## 返回值

大多数命令不应返回任何内容（通过省略 `return` 语句或 `return None`）。这表示你的命令已完成（无论是否有错误），`cmd2` 应提示用户输入更多内容。

如果你从命令方法返回 `True` 或任何[真值](https://www.freecodecamp.org/news/truthy-and-falsy-values-in-python/)，这表示 `cmd2` 应停止提示用户输入并正常退出。`cmd2` 已经包含一个 `quit` 命令，但如果你想创建另一个名为 `finish` 的命令，可以这样做：

```py
def do_finish(self, line):
    """Exit the application"""
    return True
```

## 退出码

`cmd2` 具有支持 POSIX shell 退出码的基本基础设施。`cmd2.Cmd` 对象在实例化时将 `exit_code` 属性设置为零。此属性的值由 `cmdloop()` 调用返回。因此，如果你在代码中不对此属性做任何操作，`cmdloop()` 将（几乎）总是返回零。有一些内置的 `cmd2` 命令会在发生错误时将 `exit_code` 设置为 `1`。

你可以使用此功能轻松地向操作系统 shell 返回自定义值：

```py
#!/usr/bin/env python
"""A simple cmd2 application."""
import cmd2


class App(cmd2.Cmd):
    """A simple cmd2 application."""

def do_bail(self, line):
    """Exit the application"""
    self.perror("fatal error, exiting")
    self.exit_code = 2
    return True

if __name__ == '__main__':
    import sys
    c = App()
    sys.exit(c.cmdloop())
```

如果应用从 `bash` 操作系统 shell 运行，你将看到以下交互：

```sh
(Cmd) bail
fatal error, exiting
$ echo $?
2
```

在命令或钩子函数中引发 `SystemExit(code)` 或调用 `sys.exit(code)` 也会设置 `self.exit_code` 并停止程序。

## 异常处理

你可以选择捕获和处理命令方法中发生的任何异常。如果命令方法引发异常，`cmd2` 会捕获它并为你显示。[debug 设置](../settings_plugins/#debug)控制异常的显示方式。如果 `debug` 为 `False`（默认值），`cmd2` 将显示异常名称和消息。如果 `debug` 为 `True`，`cmd2` 将显示回溯信息，然后显示异常名称和消息。

有一些命令可以引发的异常不会按上述方式打印：

- `cmd2.exceptions.SkipPostcommandHooks` - 跳过所有后命令钩子，不打印异常
- `cmd2.exceptions.Cmd2ArgparseError` - 行为类似于 `SkipPostcommandHooks`
- `SystemExit` - `stop` 将被设为 `True`，尝试停止命令循环
- `KeyboardInterrupt` - 如果在文本脚本中运行且 `stop` 尚未为 True 时引发，用于停止脚本

所有其他 `BaseException` 不会被 `cmd2` 捕获，将直接引发。

## 禁用或隐藏命令

详见[禁用命令](#禁用命令)了解如何：

- 移除 `cmd2` 中包含的命令
- 从帮助菜单中隐藏命令
- 在运行时动态禁用和重新启用命令

## 模块化命令与加载/卸载命令

详见[模块化命令](../modular_commands/)了解如何：

- 在单独的 `CommandSet` 模块中定义命令
- 在运行时动态加载或卸载命令

# 补全

`cmd2.Cmd` 为所有适用的内置命令添加了文件系统路径的 Tab 补全，包括：

- [edit](../async_builtin_clipboard/#edit)
- [run_pyscript](../async_builtin_clipboard/#run_pyscript)
- [run_script](../async_builtin_clipboard/#run_script)
- [shell](../async_builtin_clipboard/#shell)

`cmd2.Cmd` 还为 [shell](../async_builtin_clipboard/#shell) 命令添加了 shell 命令的 Tab 补全。

为自定义命令添加相同的文件系统路径补全非常简单。假设你通过实现 `do_foo` 方法定义了一个自定义命令 `foo`。要为 `foo` 命令启用路径补全，请在继承自 `cmd2.Cmd` 的类中添加类似以下的代码行：

```py
complete_foo = cmd2.Cmd.path_complete
```

这将在你的类中定义 `complete_foo` prompt-toolkit 补全方法，并使其使用与内置命令相同的路径补全逻辑。

内置逻辑还提供了一些更高级的路径补全功能，例如你只想匹配目录的情况。假设你有一个通过 `do_bar` 方法实现的自定义命令 `bar`。你可以通过添加类似以下的代码行来仅为此命令启用目录路径补全：

```py
# Make sure you have an "import functools" somewhere at the top
complete_bar = functools.partialmethod(cmd2.Cmd.path_complete, path_filter=os.path.isdir)
```

## 内置 Tab 补全函数

`cmd2.Cmd` 提供以下 Tab 补全函数：

- `basic_complete` - 针对列表进行 Tab 补全的辅助方法

- `path_complete` - 提供灵活的文件系统路径 Tab 补全的辅助方法

    > - 参阅 [paged_output](https://github.com/python-cmd2/cmd2/blob/main/examples/paged_output.py) 示例了解简单用例
    > - 参阅 [python_scripting](https://github.com/python-cmd2/cmd2/blob/main/examples/python_scripting.py) 示例了解更完整的用例

- `delimiter_complete` - 针对列表进行 Tab 补全的辅助方法，但每个匹配项按分隔符拆分

    > - 参阅 [basic_completion](https://github.com/python-cmd2/cmd2/blob/main/examples/basic_completion.py) 示例了解如何使用此功能

## 补全过程中引发异常

有时在 Tab 补全过程中会发生错误，需要向用户报告消息。包括以下示例情况：

- 读取数据库以获取补全数据集失败
- 确定正在补全的数据集的前一个命令行参数无效
- Tab 补全提示

`cmd2` 为此提供了 `CompletionError` 异常类。如果发生错误，显示消息比显示堆栈跟踪更合适，就引发 `CompletionError`。默认情况下，消息以红色（错误样式）显示。然而，`CompletionError` 有一个名为 `apply_style` 的成员。如果不想应用错误样式，将其设为 False。例如，`ArgparseCompleter` 在显示补全提示时将其设为 False。

## 使用 argparse 装饰器进行 Tab 补全

当使用 `cmd2` 的 `@with_argparser` 装饰器时，`cmd2` 提供标志名称的自动 Tab 补全。

参数值的 Tab 补全可以通过 `Cmd2ArgumentParser.add_argument()` 的三个参数之一来配置：

- `choices`
- `choices_provider`
- `completer`

参阅 [argparse_example](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_example.py) 示例了解如何使用 `choices` 参数。参阅 [argparse_completion](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_completion.py) 示例了解如何使用 `choices_provider` 参数。参阅 [argparse_example](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_example.py) 或 [argparse_completion](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_completion.py) 示例了解如何使用 `completer` 参数。

当为使用 `@with_argparser` 装饰器的 `cmd2` 命令进行标志或参数值的 Tab 补全时，`cmd2` 会跟踪状态，一旦某个标志已经被提供，就不会再次尝试补全它。当没有补全结果时，会显示当前参数的提示以帮助用户。

## CompletionItem 提供额外上下文

在补全数据库中的唯一 ID 等内容时，为用户正在补全的项目提供一些额外上下文（如描述）通常很有帮助。为此，`cmd2` 定义了 `CompletionItem` 类，它可以作为三个补全参数（`choices`、`choices_provider` 和 `completer`）中任何一个的返回值。

参阅 [argparse_completion](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_completion.py) 示例或内置 [set](../async_builtin_clipboard/#set) 命令的实现了解如何使用。

## 使用 `read_input()` 进行自定义补全

`cmd2` 提供了 `cmd2.Cmd.read_input` 作为 Python `input()` 函数的替代方案。`read_input` 支持可配置的 Tab 补全和上箭头历史记录。参阅 [read_input](https://github.com/python-cmd2/cmd2/blob/main/examples/read_input.py) 示例了解用法。

## 自定义补全菜单颜色

`cmd2` 提供了自定义补全菜单项及其关联帮助文本的前景色和背景色的功能。详见主题文档中的[自定义补全菜单颜色](../misc_packaging/#自定义补全菜单颜色)。

## 更多信息

参阅 cmd2 的 argparse_utils API 了解关于 argparse 补全的更详细讨论。

# 禁用命令

`cmd2` 允许开发者：

- 移除 `cmd2` 中包含的命令
- 阻止命令出现在帮助菜单中（隐藏命令）
- 在运行时禁用和重新启用命令

参阅 [remove_builtin_commands.py](https://github.com/python-cmd2/cmd2/blob/main/examples/remove_builtin_commands.py) 了解移除或隐藏内置命令的示例。

参阅 [command_sets.py](https://github.com/python-cmd2/cmd2/blob/main/examples/command_sets.py) 了解在运行时动态启用和禁用自定义命令的示例。

## 移除命令

当命令被移除后，命令方法已从对象中删除。该命令不会显示在帮助中，也无法执行。如果你永远不希望某个内置命令成为应用的一部分，这种方法是合适的。在初始化代码中删除命令方法：

```py
    class RemoveBuiltinCommand(cmd2.Cmd):
        """An app which removes a built-in command from cmd2"""

        def __init__(self):
            super().__init__()
            # To remove built-in commands entirely, delete
            # the "do_*" function from the cmd2.Cmd class
            del cmd2.Cmd.do_edit
```

## 隐藏命令

当命令被隐藏后，它不会出现在帮助菜单中，也不会被 Tab 补全，但如果用户知道它的存在并输入该命令，它仍会被执行。你通过将命令添加到 `hidden_commands` 列表来隐藏它：

```py
class HiddenCommands(cmd2.Cmd):
    """An app which demonstrates how to hide a command"""
    def __init__(self):
        super().__init__()
        self.hidden_commands.append('py')
```

如上所示，你通常会在初始化应用时执行此操作。如果你稍后决定取消隐藏某个命令，可以这样做：

```py
self.hidden_commands = [cmd for cmd in self.hidden_commands if cmd != 'py']
```

你可能觉得列表推导式有些多余，更愿意这样做：

```py
self.hidden_commands.remove('py')
```

你也许是对的，但如果 `py` 不在列表中，`remove()` 会引发 `ValueError`，而且如果列表中有多个相同的项，它只会移除第一个。

## 禁用命令

禁用命令的一种方式是在命令方法中添加代码来判断是否应执行该命令。如果命令不应执行，你的代码可以打印适当的错误消息并返回。

`cmd2` 还提供了另一种方式来实现同样的效果。以下是一个简单的应用，在门被锁时禁用 `open` 命令：

```py
class DisabledCommands(cmd2.Cmd):
    """An application which disables and enables commands"""

    def do_lock(self, line):
        self.disable_command('open', "you can't open the door because it is locked")
        self.poutput('the door is locked')

    def do_unlock(self, line):
        self.enable_command('open')
        self.poutput('the door is unlocked')

    def do_open(self, line):
        """open the door"""
        self.poutput('opening the door')
```

这种方法还有一个额外的好处，就是将禁用的命令从帮助菜单中移除。但这种方法只适用于你提前知道命令应被禁用，且重新启用的条件也是提前已知的情况。

## 禁用一类命令

你可以按[命令分类](../embedded_output_help/#命令分类)中所述对命令进行分组或分类。如果这样做，你可以通过一次方法调用来禁用和启用某个分类中的所有命令。假设你创建了一个名为"服务器信息"的命令分类，你可以禁用该分类中的所有命令：

```py
not_connected_msg = 'You must be connected to use this command'
self.disable_category('Server Information', not_connected_msg)
```

类似地，你可以重新启用某个分类中的所有命令：

```py
self.enable_category('Server Information')
```

参阅 [help_categories.py](https://github.com/python-cmd2/cmd2/blob/main/examples/help_categories.py) 了解在运行时动态启用和禁用整个命令分类的示例。
