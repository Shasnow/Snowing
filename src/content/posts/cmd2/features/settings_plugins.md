---
title: Python cmd2：设置系统与插件系统详解
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的设置系统，包括内置设置（allow_style、debug、echo 等）、创建新设置和隐藏内置设置。同时，介绍插件系统，包括创建插件、加载插件和使用插件的功能。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
date: 2026-05-28
pubDate: 2026-05-28
---

# 设置

设置为用户提供了控制基于 `cmd2` 应用行为的机制。设置存储在 `cmd2.Cmd` 子类的受保护实例属性中。开发者可以为这些设置设置默认值，用户可以在运行时使用 [set](../async_builtin_clipboard/#set) 命令查看和修改它们。开发者可以[创建新设置](#创建新设置)，也可以对用户[隐藏内置设置](#隐藏设置)。

## 内置设置

`cmd2` 有许多内置设置。这些设置控制某些应用功能和[内置命令](../async_builtin_clipboard/#内置命令)的行为。用户可以使用 [set](../async_builtin_clipboard/#set) 命令显示所有设置并修改任何设置的值。

### allow_style

`cmd2` 程序生成的输出可能包含 ANSI 转义序列，指示终端对输出应用颜色或文本样式（即粗体）。`allow_style` 设置控制以下方法生成的输出中这些转义序列的行为：

- `cmd2.Cmd.perror`
- `cmd2.Cmd.pexcept`
- `cmd2.Cmd.pfeedback`
- `cmd2.Cmd.poutput`
- `cmd2.Cmd.ppaged`
- `cmd2.Cmd.print_to`
- `cmd2.Cmd.psuccess`
- `cmd2.Cmd.pwarning`

此设置可以是三个值之一：

- `Never` - 从输出中去除所有指示终端设置输出样式的 ANSI 转义序列
- `Terminal` - （默认值）当输出发送到终端时传递 ANSI 转义序列，但如果输出被重定向到管道或文件，则去除转义序列
- `Always` - ANSI 转义序列始终传递到输出

### debug

此设置的默认值为 `False`，这导致 `cmd2.Cmd.pexcept` 方法只显示异常的消息。然而，如果 debug 设置为 `True`，则会打印整个堆栈跟踪。

### echo

如果为 `True`，用户发出的每个命令在执行前会被回显到屏幕上。这在运行脚本时特别有用。在提示符处运行命令时不会出现此行为。

### editor

类似于 `EDITOR` shell 变量，此设置包含 [edit](../async_builtin_clipboard/#edit) 命令应运行的程序名称。

### max_completion_table_items

补全表中显示的最大项目数。补全表是一种特殊的补全提示，显示正在补全的项目的详细信息。Tab 补全 `set` 命令可以查看示例。

如果补全建议的数量超过 `max_completion_table_items`，则不会显示表格。

### quiet

如果为 `True`，调用 `cmd2.Cmd.pfeedback` 生成的输出将被抑制。如果为 `False`，输出将发送到 `stdout`。

### scripts_add_to_history

如果为 `True`，脚本和 pyscripts 会将命令添加到历史记录。此设置的默认值为 `True`。

### timing

如果为 `True`，将报告每个执行命令的经过时间。

## 创建新设置

你的应用可以定义代码可以引用的用户可设置参数。在初始化代码中：

1. 创建一个带有默认值的实例属性
2. 创建一个描述你的设置的 [Settable](https://cmd2.readthedocs.io/en/latest/api/utils/#cmd2.utils.Settable) 对象
3. 将 `Settable` 对象传递给 `add_settable` 方法

下面是一个示例，展示了如何创建一个新设置：

```python
#!/usr/bin/env python
"""A sample application for cmd2 demonstrating customized environment parameters."""

import cmd2


class EnvironmentApp(cmd2.Cmd):
    """Example cmd2 application."""

    def __init__(self) -> None:
        super().__init__()
        self.degrees_c = 22
        self.sunny = False
        self.add_settable(
            cmd2.Settable("degrees_c", int, "Temperature in Celsius", self, onchange_cb=self._onchange_degrees_c)
        )
        self.add_settable(cmd2.Settable("sunny", bool, "Is it sunny outside?", self))

    def do_sunbathe(self, _arg) -> None:
        """Attempt to sunbathe."""
        if self.degrees_c < 20:
            result = f"It's {self.degrees_c} C - are you a penguin?"
        elif not self.sunny:
            result = "Too dim."
        else:
            result = "UV is bad for your skin."
        self.poutput(result)

    def _onchange_degrees_c(self, _param_name, _old, new) -> None:
        # if it's over 40C, it's gotta be sunny, right?
        if new > 40:
            self.sunny = True


if __name__ == "__main__":
    import sys

    c = EnvironmentApp()
    sys.exit(c.cmdloop())
```

如果你想在设置更改时收到通知，请确保为 `cmd2.utils.Settable` 的 `onchange_cb` 参数提供一个方法。此方法将在用户更改设置后调用，并接收旧值和新值。

```text
(Cmd) set | grep sunny
sunny                 False                           Is it sunny outside?
(Cmd) set | grep degrees
degrees_c             22                              Temperature in Celsius
(Cmd) sunbathe
Too dim.
(Cmd) set degrees_c 41
degrees_c - was: 22
now: 41
(Cmd) set sunny
sunny: True
(Cmd) sunbathe
UV is bad for your skin.
(Cmd) set degrees_c 13
degrees_c - was: 41
now: 13
(Cmd) sunbathe
It's 13 C - are you a penguin?
```

## 隐藏设置

你可能希望阻止用户修改内置设置。假设你永远不希望程序的最终用户能够在发生错误时启用完整的调试回溯。你可能想要隐藏 [debug](#debug) 设置。`cmd2.Cmd.remove_settable` 方法使这变得简单：

```py
class MyApp(cmd2.Cmd):

    def __init__(self):
        super().__init__()
        self.remove_settable('debug')
```

# 插件

`cmd2` 有一个内置的插件框架，允许开发者创建 `cmd2` 插件来扩展基本 `cmd2` 功能，并可被多个应用使用。

有多种方式可以使用插件向 `cmd2` 添加功能。大多数插件将作为混入（mixin）实现。混入是一个将代码封装并注入到另一个类中的类。在 `cmd2` 项目中使用插件的开发者将把插件的代码注入到他们的 [cmd2.Cmd][] 子类中。

## 混入与初始化

以下简短示例展示了如何混入插件以及插件如何被初始化。

插件代码：

```py
class MyPlugin:
    def __init__(self, *args, **kwargs):
        # code placed here runs before cmd2.Cmd initializes
        super().__init__(*args, **kwargs)
        # code placed here runs after cmd2.Cmd initializes
```

使用插件的示例应用：

```py
import cmd2
import cmd2_myplugin

class Example(cmd2_myplugin.MyPlugin, cmd2.Cmd):
    """An class to show how to use a plugin"""
    def __init__(self, *args, **kwargs):
        # code placed here runs before cmd2.Cmd or
        # any plugins initialize
        super().__init__(*args, **kwargs)
        # code placed here runs after cmd2.Cmd and
        # all plugins have initialized
```

:::warning
插件必须在 `cmd2.Cmd` 之前被继承（或混入）。这有两个原因：

- Python 标准库中的 `cmd.Cmd.__init__` 方法不调用 `super().__init__()`。由于这个疏忽，如果你不先从 `MyPlugin` 继承，`MyPlugin.__init__()` 方法将永远不会被调用。
- 你可能希望你的插件能够重写 `cmd2.Cmd` 的方法。如果你在 `cmd2.Cmd` 之后混入插件，Python 方法解析顺序将先调用 `cmd2.Cmd` 的方法，然后才调用插件中的方法。
:::

## 添加命令

你的插件可以添加用户可见的命令。在插件中的操作方式与在 `cmd2.Cmd` 应用中相同：

```py
class MyPlugin:
    def do_say(self, statement):
        """Simple say command"""
        self.poutput(statement)
```

你在插件中拥有与在 `cmd2.Cmd` 应用中相同的所有能力，包括通过装饰器进行参数解析和自定义帮助方法。

## 添加（或隐藏）设置

插件可以向应用添加用户可控的设置。示例：

```py
class MyPlugin:
    def __init__(self, *args, **kwargs):
        # code placed here runs before cmd2.Cmd initializes
        super().__init__(*args, **kwargs)
        # code placed here runs after cmd2.Cmd initializes
        self.mysetting = 'somevalue'
        self.add_settable(cmd2.Settable('mysetting', str, 'short help message for mysetting', self))
```

你可以通过调用 `cmd2.Cmd.remove_settable` 来对用户隐藏设置。详见[设置](../settings_plugins/#设置)。

## 装饰器

你的插件可以提供装饰器，插件用户可以使用它来将自己的功能包装在他们自己的命令周围。

## 重写方法

你的插件可以重写核心 `cmd2.Cmd` 方法，改变它们的行为。这种方法应该谨慎使用，因为它非常脆弱。如果开发者选择在应用中使用多个插件，并且其中几个插件重写了同一个方法，只有第一个被混入的插件的重写方法会被调用。

钩子是更好的方法。

## 钩子

插件可以注册钩子方法，这些方法由 `cmd2.Cmd` 在应用和命令处理生命周期的各个点调用。插件不应重写任何 `cmd` 基类钩子方法，而应按照[钩子](../hooks/)部分所述注册它们的钩子。

你应该以插件名称开头来命名你的钩子。钩子方法会被混入 `cmd2` 应用中，这种命名约定有助于避免意外的方法重写。

简单示例：

```py
class MyPlugin:
    def __init__(self, *args, **kwargs):
        # code placed here runs before cmd2 initializes
        super().__init__(*args, **kwargs)
        # code placed here runs after cmd2 initializes
        # this is where you register any hook functions
        self.register_postparsing_hook(self.cmd2_myplugin_postparsing_hook)

    def cmd2_myplugin_postparsing_hook(self, data: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
        """Method to be called after parsing user input, but before running the command"""
        self.poutput('in postparsing_hook')
        return data
```

注册允许多个插件（甚至应用本身）各自注入在应用或命令处理生命周期中被调用的代码。

参阅[钩子](../hooks/)文档了解应用和命令生命周期的完整细节，包括所有可用钩子和钩子影响生命周期的方式。

## 类和函数

你的插件还可以提供 `cmd2` 应用开发者可以使用的类和函数。在文档中描述这些类和函数，以便插件用户知道有哪些可用。

## 示例

参阅 [cmd2 插件模板](https://github.com/python-cmd2/cmd2-plugin-template)了解更多信息。
