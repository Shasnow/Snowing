---
title: Python cmd2：混入类（Mixin）详解
published: 2026-09-22
pinned: false
description: 介绍 cmd2 的混入类（Mixin）机制，包括混入类的初始化顺序、添加命令与设置、重写方法以及注册钩子等最佳实践。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 混入类（Mixin）

## 混入类基础

在 Python 中，混入（mixin）是一种通过多重继承向其他类提供特定功能集合的类。混入不打算被单独实例化；相反，它们的作用是将行为"混合"或组合到一个基类中，而不建立僵化的"是一个（is-a）"关系。

关于混入类的更多信息，我们推荐这篇 `Real Python` 文章：[What Are Mixin Classes in Python?](https://realpython.com/python-mixin/)

## cmd2 混入类概览

如果你有一组可复用的行为，希望应用于多个不同的 `cmd2` 应用，那么创建一个混入类来封装这些行为是个好主意。这是依赖多重继承来扩展 `cmd2` 的一种方式。它快速而简单，但有一些潜在的陷阱需要了解，以便你知道如何正确使用。

[mixins.py](https://github.com/python-cmd2/cmd2/blob/main/examples/mixin.py) 示例展示了如何为 `cmd2` 应用开发混入类。过去我们称这些为"插件（Plugins）"，但回想起来这可能不是最好的名字。它们本质上是混入类，为继承自 [cmd2.Cmd][] 的类添加一些额外功能。

## 使用本模板

本文件提供了一个非常基础的模板，说明你如何创建自己的 cmd2 混入类，将可复用的行为封装起来，通过多重继承应用于多个 `cmd2` 应用。

## 命名

如果你决定将混入发布为 Python 包，应考虑给项目名称加上 `cmd2-` 前缀。采用这种方式后，项目内部应有一个带 `cmd2_` 前缀的包。

## 添加功能

有许多方式可以通过混入为 `cmd2` 添加功能。混入是一个封装并将代码注入另一个类的类。在 `cmd2` 项目中使用混入的开发者，会将混入的代码注入到他们继承 [cmd2.Cmd][] 的子类中。

### 混入类与初始化

下面这个简短示例展示了如何创建混入类，以及所有内容是如何初始化的。

混入类：

```python
class MyMixin:
    def __init__(self, *args, **kwargs):
        # 此处的代码在 cmd2.Cmd 初始化之前运行
        super().__init__(*args, **kwargs)
        # 此处的代码在 cmd2.Cmd 初始化之后运行
```

使用该混入的示例应用：

```python
import cmd2


class Example(MyMixin, cmd2.Cmd):
    """展示如何使用混入类的 cmd2 应用类。"""

    def __init__(self, *args, **kwargs):
        # 此处的代码在 cmd2.Cmd 或
        # 任何混入初始化之前运行
        super().__init__(*args, **kwargs)
        # 此处的代码在 cmd2.Cmd 和
        # 所有混入初始化之后运行
```

注意混入必须在 `cmd2.Cmd` 之前被继承（或混入）。这有两个原因：

- Python 标准库中的 `cmd.Cmd.__init__()` 方法不会调用 `super().__init__()`。由于这一疏漏，如果你不先继承 `MyMixin`，`MyMixin.__init__()` 方法将永远不会被调用。
- 你可能希望混入能够重写 `cmd2.Cmd` 的方法。如果你在 `cmd2.Cmd` 之后混入混入类，Python 的方法解析顺序（MRO）会先调用 `cmd2.Cmd` 的方法，再调用混入中的方法。

### 添加命令

你的混入可以添加用户可见的命令。在混入中的做法与在 `cmd2.Cmd` 应用中完全相同：

```python
class MyMixin:
    def do_say(self, statement):
        """简单的 say 命令"""
        self.poutput(statement)
```

在混入中你拥有与 `cmd2.Cmd` 应用中相同的所有能力，包括通过装饰器进行参数解析和自定义帮助方法。

### 添加（或隐藏）设置

混入可以向应用添加用户可控制的设置。示例如下：

```python
class MyMixin:
    def __init__(self, *args, **kwargs):
        # 此处的代码在 cmd2.Cmd 初始化之前运行
        super().__init__(*args, **kwargs)
        # 此处的代码在 cmd2.Cmd 初始化之后运行
        self.mysetting = "somevalue"
        self.settable.update({"mysetting": "short help message for mysetting"})
```

你也可以通过从 `self.settable` 中移除的方式对用户隐藏设置。

### 装饰器

你的混入可以提供装饰器，供混入的使用者将其包装在自己的命令周围以附加功能。

### 重写方法

你的混入可以重写 `cmd2.Cmd` 的核心方法，改变其行为。这种方式应谨慎使用，因为它非常脆弱。如果开发者在应用中使用多个混入，且多个混入重写了同一个方法，只有最先被混入的混入的重写方法会被调用。

钩子是好得多的方案。

### 钩子

混入可以注册钩子，由 `cmd2.Cmd` 在应用和命令处理生命周期的各个时点调用。混入不应重写任何遗留的 `cmd` 钩子方法，而应按照 `cmd2` 文档中的[描述](features/hooks/)注册自己的钩子。

你应该以混入的名称作为钩子名称的开头。钩子方法会被混入 `cmd2` 应用，这种命名约定有助于避免无意间的方法重写。

一个简单示例：

```python
class MyMixin:
    def __init__(self, *args, **kwargs):
        # 此处的代码在 cmd2 初始化之前运行
        super().__init__(*args, **kwargs)
        # 此处的代码在 cmd2 初始化之后运行
        # 在这里注册你的钩子函数
        self.register_postparsing_hook(self.cmd2_mymixin_postparsing_hook)

    def cmd2_mymixin_postparsing_hook(self, data: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
        """在解析用户输入之后、运行命令之前调用的方法"""
        self.poutput("in postparsing_hook")
        return data
```

注册方式允许多个混入（甚至应用本身）各自注入代码，在应用或命令处理生命周期期间被调用。

完整的应用和命令生命周期信息，包括所有可用钩子以及钩子影响生命周期的方式，参阅 [cmd2 钩子文档](features/hooks/)。

### 类与函数

你的混入还可以提供类和函数，供基于 `cmd2` 的应用开发者使用。请在文档中描述这些类和函数，让混入的用户知道有哪些可用。
