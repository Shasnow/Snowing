---
title: Python cmd2：模块化命令详解
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的模块化命令系统，包括 CommandSet 的定义、自动加载、动态加载/卸载、事件处理器和子命令注入。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 模块化命令

## 概述

Cmd2 还允许开发者将其命令定义模块化为 `CommandSet` 对象。CommandSet 代表 `cmd2` 应用中命令的逻辑分组。默认情况下，`CommandSet` 对象需要手动注册。然而，通过在实例化 `cmd2.Cmd` 类时设置 `auto_load_commands=True`，可以自动发现和加载所有 `CommandSet` 对象。这还允许开发者动态地从 `cmd2` 应用中添加/移除命令。这对于添加额外功能的可加载插件非常有用。此外，它允许面向对象的封装和特定于 CommandSet 的状态的垃圾回收。

### 功能特性

- **模块化命令集** - 命令可以被拆分为单独的模块，而不是在一个包含所有命令的"上帝类"中
- **自动命令发现** - 在应用中，只需定义和导入 CommandSet 就足以让 `cmd2` 发现和加载你的命令（如果设置了 `auto_load_commands=True`）。无需手动注册
- **动态可加载/可卸载命令** - 命令函数和 CommandSet 都可以在应用执行期间动态加载和卸载。这可以实现动态加载模块以添加额外命令等功能
- **事件处理器** - `CommandSet` 类中提供了四个事件处理器用于自定义初始化和清理步骤。详见[事件处理器](#事件处理器)
- **子命令注入** - 子命令可以独立于基础命令定义。这允许以操作为中心（而非以对象为中心）的命令系统，同时仍然围绕被管理的对象组织代码和处理器

参阅 [cmd2.CommandSet][] 的 API 文档。

参阅[示例](https://github.com/python-cmd2/cmd2/tree/main/examples/modular_commands)了解更多细节。

## 定义命令

### 命令集

CommandSet 将多个命令组合在一起。插件将使用与在 `cmd2.Cmd` 中定义时相同的规则来检查 `CommandSet` 中的函数。命令必须以 `do_` 为前缀，帮助函数以 `help_` 为前缀，补全函数以 `complete_` 为前缀。

CommandSet 命令方法期望的参数与在 `cmd2.Cmd` 子类中定义时相同，只是 `self` 现在将引用 `CommandSet` 而不是 cmd2 实例。cmd2 实例可以通过在 `CommandSet` 注册时填充的 `self._cmd` 来访问。

只有在初始化器不接受参数时，CommandSet 才会被自动加载。如果需要提供初始化器参数，请参阅[手动 CommandSet 构造](#手动-commandset-构造)。

```py
import cmd2
from cmd2 import CommandSet

class ExampleApp(cmd2.Cmd):
    """
    CommandSets are automatically loaded. Nothing needs to be done.
    """
    def __init__(self, *args, **kwargs):
        super().__init__(*args, auto_load_commands=True, **kwargs)

    def do_something(self, arg):
        """Something Command."""
        self.poutput('this is the something command')

class AutoLoadCommandSet(CommandSet[ExampleApp]):
    DEFAULT_CATEGORY = 'My Category'

    def __init__(self):
        super().__init__()

    def do_hello(self, _: cmd2.Statement):
        """Hello Command."""
        self._cmd.poutput('Hello')

    def do_world(self, _: cmd2.Statement):
        """World Command."""
        self._cmd.poutput('World')
```

### 手动 CommandSet 构造

如果 CommandSet 类需要向初始化器提供参数，你可以手动构造 CommandSet 并将其传递给 Cmd2。

```py
import cmd2
from cmd2 import CommandSet

class ExampleApp(cmd2.Cmd):
    """
    CommandSets with initializer parameters are provided in the initializer
    """
    def __init__(self, *args, **kwargs):
        super().__init__(*args, auto_load_commands=True, **kwargs)

    def do_something(self, arg):
        """Something Command."""
        self.last_result = 5
        self.poutput('this is the something command')

class CustomInitCommandSet(CommandSet[ExampleApp]):
    DEFAULT_CATEGORY = 'My Category'

    def __init__(self, arg1, arg2):
        super().__init__()
        self._arg1 = arg1
        self._arg2 = arg2

    def do_show_arg1(self, _: cmd2.Statement):
        """Show Arg 1."""
        self._cmd.poutput(f'Arg1: {self._arg1}')

    def do_show_arg2(self, _: cmd2.Statement):
        """Show Arg 2."""
        self._cmd.poutput(f'Arg2: {self._arg2}')


def main():
    my_commands = CustomInitCommandSet(1, 2)
    app = ExampleApp(command_sets=[my_commands])
    app.cmdloop()
```

### 类型提示与 self._cmd

当 `CommandSet` 被注册时，其 `_cmd` 属性会被填充为对 `cmd2.Cmd` 实例的引用。`CommandSet` 是一个[泛型](https://docs.python.org/3/library/typing.html#typing.Generic)类，允许你指定它期望被加载到的特定 `cmd2.Cmd` 子类。

通过使用你的应用类参数化继承，你的 IDE 和静态分析工具（如 Mypy）将知道 `self._cmd` 的确切类型。这在访问主应用实例的自定义属性或方法时提供了完整的自动补全和类型验证。

```py
import cmd2
from cmd2 import CommandSet

class MyApp(cmd2.Cmd):
    def __init__(self):
        super().__init__()
        self.custom_state = "Some important data"

class MyCommands(CommandSet[MyApp]):
    def do_check_state(self, _: cmd2.Statement):
        # Type checkers know self._cmd is an instance of MyApp
        self._cmd.poutput(f"State: {self._cmd.custom_state}")
```

### 动态命令

你还可以通过在运行时安装和移除 CommandSet 来动态加载和卸载命令。例如，你可以支持运行时可加载的插件或根据状态添加/移除命令。

如果需要在运行时动态加载命令，你可能需要禁用命令自动加载。

```py
import argparse
import cmd2
from cmd2 import CommandSet, with_argparser, with_category


class LoadableFruits(CommandSet["ExampleApp"]):
    DEFAULT_CATEGORY = 'Fruits'

    def __init__(self):
        super().__init__()

    def do_apple(self, _: cmd2.Statement):
        """Apple Command."""
        self._cmd.poutput('Apple')

    def do_banana(self, _: cmd2.Statement):
        """Banana Command."""
        self._cmd.poutput('Banana')


class LoadableVegetables(CommandSet["ExampleApp"]):
    DEFAULT_CATEGORY = 'Vegetables'

    def __init__(self):
        super().__init__()

    def do_arugula(self, _: cmd2.Statement):
        """Arugula Command."""
        self._cmd.poutput('Arugula')

    def do_bokchoy(self, _: cmd2.Statement):
        """Bok Choy Command."""
        self._cmd.poutput('Bok Choy')


class ExampleApp(cmd2.Cmd):
    """
    CommandSets are loaded via the `load` and `unload` commands
    """

    def __init__(self, *args, **kwargs):
        super().__init__(*args, auto_load_commands=False, **kwargs)
        self._fruits = LoadableFruits()
        self._vegetables = LoadableVegetables()

    load_parser = cmd2.Cmd2ArgumentParser()
    load_parser.add_argument('cmds', choices=['fruits', 'vegetables'])

    @with_argparser(load_parser)
    @with_category('Command Loading')
    def do_load(self, ns: argparse.Namespace):
        """Load Command."""
        if ns.cmds == 'fruits':
            try:
                self.register_command_set(self._fruits)
                self.poutput('Fruits loaded')
            except ValueError:
                self.poutput('Fruits already loaded')

        if ns.cmds == 'vegetables':
            try:
                self.register_command_set(self._vegetables)
                self.poutput('Vegetables loaded')
            except ValueError:
                self.poutput('Vegetables already loaded')

    @with_argparser(load_parser)
    def do_unload(self, ns: argparse.Namespace):
        """Unload Command."""
        if ns.cmds == 'fruits':
            self.unregister_command_set(self._fruits)
            self.poutput('Fruits unloaded')

        if ns.cmds == 'vegetables':
            self.unregister_command_set(self._vegetables)
            self.poutput('Vegetables unloaded')


if __name__ == '__main__':
    app = ExampleApp()
    app.cmdloop()
```

## 事件处理器

以下函数在 `CommandSet` 生命周期的不同阶段被调用：

- **on_register** - 由 `cmd2.Cmd` 在注册 `CommandSet` 的第一步调用。此时此类中定义的命令尚未添加到 CLI 对象。子类可以重写此方法以执行任何需要访问 Cmd 对象的初始化（例如，基于 CLI 状态数据配置命令及其解析器）
- **on_registered** - 由 `cmd2.Cmd` 在 `CommandSet` 注册完成且所有命令已添加到 CLI 后调用。子类可以重写此方法以执行与新添加命令相关的自定义步骤（例如，将它们设为禁用状态）
- **on_unregister** - 由 `cmd2.Cmd` 在注销 `CommandSet` 的第一步调用。子类可以重写此方法以执行任何需要其命令在 CLI 中注册的清理步骤
- **on_unregistered** - 由 `cmd2.Cmd` 在 `CommandSet` 注销完成且所有命令已从 CLI 中移除后调用。子类可以重写此方法以执行剩余的清理步骤

## 注入子命令

### 说明

使用 `@with_argparser` 和 `@as_subcommand_to` 装饰器，可以轻松地为命令定义子命令。这往往会将你的接口推向以对象为中心的接口。例如，假设你有一个管理媒体收藏的工具，你想管理电影或节目。以对象为中心的方式会要求你有 `movies` 和 `shows` 等基础命令，每个都有 `add`、`edit`、`list`、`delete` 子命令。如果你想呈现以操作为中心的命令集，使 `add`、`edit`、`list` 和 `delete` 成为基础命令，你就必须围绕这些类似的操作组织代码，而不是围绕被管理的类似对象组织代码。

子命令注入允许你将子命令注入到基础命令中，以呈现对用户合理的接口，同时仍然以对开发者更有逻辑意义的结构组织代码。

### 示例

此示例是上面动态命令示例的变体。引入了一个 `cut` 命令作为基础命令，每个 CommandSet 向其添加一个子命令。

```py
import argparse
import cmd2
from cmd2 import CommandSet, with_argparser, with_category


class LoadableFruits(CommandSet["ExampleApp"]):
    DEFAULT_CATEGORY = 'Fruits'

    def __init__(self):
        super().__init__()

    def do_apple(self, _: cmd2.Statement):
        """Apple Command."""
        self._cmd.poutput('Apple')

    banana_parser = cmd2.Cmd2ArgumentParser()
    banana_parser.add_argument('direction', choices=['discs', 'lengthwise'])

    @cmd2.as_subcommand_to('cut', 'banana', banana_parser)
    def cut_banana(self, ns: argparse.Namespace):
        """Cut banana"""
        self._cmd.poutput('cutting banana: ' + ns.direction)


class LoadableVegetables(CommandSet["ExampleApp"]):
    DEFAULT_CATEGORY = 'Vegetables'

    def __init__(self):
        super().__init__()

    def do_arugula(self, _: cmd2.Statement):
        """Arugula Command."""
        self._cmd.poutput('Arugula')

    bokchoy_parser = cmd2.Cmd2ArgumentParser()
    bokchoy_parser.add_argument('style', choices=['quartered', 'diced'])

    @cmd2.as_subcommand_to('cut', 'bokchoy', bokchoy_parser)
    def cut_bokchoy(self, _: argparse.Namespace):
        """Cut bok choy."""
        self._cmd.poutput('Bok Choy')


class ExampleApp(cmd2.Cmd):
    """
    CommandSets are loaded dynamically at runtime via other commands.
    """

    def __init__(self, *args, **kwargs):
        super().__init__(*args, auto_load_commands=False, **kwargs)
        self._fruits = LoadableFruits()
        self._vegetables = LoadableVegetables()

    load_parser = cmd2.Cmd2ArgumentParser()
    load_parser.add_argument('cmds', choices=['fruits', 'vegetables'])

    @with_argparser(load_parser)
    @with_category('Command Loading')
    def do_load(self, ns: argparse.Namespace):
        """Load Command."""
        if ns.cmds == 'fruits':
            try:
                self.register_command_set(self._fruits)
                self.poutput('Fruits loaded')
            except ValueError:
                self.poutput('Fruits already loaded')

        if ns.cmds == 'vegetables':
            try:
                self.register_command_set(self._vegetables)
                self.poutput('Vegetables loaded')
            except ValueError:
                self.poutput('Vegetables already loaded')

    @with_argparser(load_parser)
    def do_unload(self, ns: argparse.Namespace):
        """Unload Command."""
        if ns.cmds == 'fruits':
            self.unregister_command_set(self._fruits)
            self.poutput('Fruits unloaded')

        if ns.cmds == 'vegetables':
            self.unregister_command_set(self._vegetables)
            self.poutput('Vegetables unloaded')

    cut_parser = cmd2.Cmd2ArgumentParser()
    cut_subparsers = cut_parser.add_subparsers(title='item', help='item to cut')

    @with_argparser(cut_parser)
    def do_cut(self, ns: argparse.Namespace):
        """Cut Command."""
        handler = ns.cmd2_subcmd_handler
        if handler is not None:
            handler(ns)
        else:
            self.poutput('This command does nothing without sub-parsers registered')
            self.do_help('cut')


if __name__ == '__main__':
    app = ExampleApp()
    app.cmdloop()
```
