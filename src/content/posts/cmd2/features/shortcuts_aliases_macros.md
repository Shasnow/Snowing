---
title: Python cmd2：快捷方式、别名与宏
published: 2026-05-28
pinned: false
description: 介绍 cmd2 的快捷方式、别名和宏功能，帮助用户更高效地使用命令行。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
date: 2026-05-28
pubDate: 2026-05-28
---

# 快捷方式、别名与宏

## 快捷方式

长命令名称和常用命令的快捷方式可以为用户提供更多便利。快捷方式使用时与其参数之间没有空格分隔，如 `!ls`。默认情况下，定义了以下快捷方式：

- **`?`** - help
- **`!`** - shell：以操作系统级别命令运行
- **`@`** - 运行脚本文件
- **`@@`** - 运行脚本文件；文件名相对于当前脚本位置

要定义更多快捷方式，从 `cmd2.DEFAULT_SHORTCUTS` 常量（一个字典）开始，然后通过使用格式为 `{'shortcut': 'command_name'}` 的额外快捷方式字典更新它来添加更多快捷方式，其中命令名称省略 `do_`：

```py
class App(Cmd):
    def __init__(self):
        shortcuts = cmd2.DEFAULT_SHORTCUTS
        shortcuts.update({'*': 'sneeze', '~': 'squirm'})
        cmd2.Cmd.__init__(self, shortcuts=shortcuts)
```

:::warning
快捷方式需要在调用 `cmd2.Cmd` 超类 `__init__()` 方法之前通过更新 `shortcuts` 字典属性来创建。此外，超类 init 方法需要在更新 `shortcuts` 属性之后调用。

此警告通常适用于许多其他在运行时不可设置的属性。
:::

:::note
命令、别名和宏的名称不能以快捷方式开头。
:::

## 别名

除了快捷方式，`cmd2` 还通过 `alias` 命令提供别名功能。别名的工作方式类似于 Bash shell 中的别名。

创建别名的语法是：`alias create name command [args]`，例如 `alias create ls !ls -lF`。

重定向器和管道应在别名定义中加引号，以防止 `alias create` 命令被重定向：

    alias create save_results print_results ">" out.txt

Tab 补全能识别别名，并像命令行上的别名命令一样进行补全。

更多详情请运行：`help alias create`

使用 `alias list` 查看所有或部分别名。此命令的输出以可用于创建它们的格式显示你的别名。因此你可以将此输出放在 `cmd2` 启动脚本中，以便每次启动应用时重新创建你的别名。

使用 `alias delete` 删除别名。

更多详情请运行：`help alias delete`

:::note
别名不能与命令或宏同名。
:::

## 宏

`cmd2` 提供了一种类似于别名的功能，称为宏。宏和别名之间的主要区别是宏可以包含参数占位符。参数在创建宏时使用 `{#}` 表示法表达，其中 `{1}` 表示第一个参数。

以下创建了一个名为 `my_macro` 的宏，它期望两个参数：

    macro create my_macro make_dinner -meat {1} -veggie {2}

当宏被调用时，提供的参数会被替换，组装好的命令会被运行。例如：

    my_macro beef broccoli ---> make_dinner -meat beef -veggie broccoli

与别名类似，管道和重定向器需要在宏的定义中加引号：

    macro create lc !cat "{1}" "|" less

要在命令中使用字面字符串 `{1}`，请这样转义：`{{1}}`。

由于宏在你按 `<Enter>` 后才解析，它们的参数会作为路径进行 Tab 补全。你可以通过重写 `Cmd.macro_arg_complete()` 来实现宏参数的自定义 Tab 补全，从而更改此默认行为。

更多详情请运行：`help macro create`

macro 命令有 `list` 和 `delete` 子命令，其功能与同名的 alias 子命令相同。与别名一样，宏可以通过 `cmd2` 启动脚本创建，以便在应用会话之间保留它们。

更多关于列出宏的详情请运行：`help macro list`

更多关于删除宏的详情请运行：`help macro delete`

:::note
宏不能与命令或别名同名。
:::
