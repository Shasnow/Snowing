---
title: Python cmd2：操作系统集成
published: 2026-05-28
pinned: false
description: 介绍 cmd2 与操作系统的集成，包括输出重定向、执行 OS 命令、编辑器、终端分页器、退出码和命令行参数传递。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
date: 2026-05-28
pubDate: 2026-05-28
---

# 操作系统集成

## 如何重定向输出

参阅[输出重定向和管道](../prompt_redirection/#输出重定向和管道)。

## 在 cmd2 中执行操作系统命令

`cmd2` 包含一个 `shell` 命令，它在操作系统 shell 中执行其参数：

    (Cmd) shell ls -al

如果你使用 `cmd2` 中定义的默认[快捷方式](../shortcuts_aliases_macros/#快捷方式)，你将获得一个 `!` 快捷方式用于 `shell`，允许你输入：

    (Cmd) !ls -al

:::note
`cmd2` 在运行 shell 命令的整个过程中提供用户友好的 Tab 补全——首先是 shell 命令名称本身，然后是参数部分中的文件路径。

然而，`cmd2` 应用实际上**成为**了 shell，所以如果你为特定的 shell（如 `bash`、`zsh`、`fish` 等）配置了_额外的_ shell 补全，这在 `cmd2` 中将不可用。
:::

## 编辑器

`cmd2` 包含内置的 `edit` 命令，它运行文本编辑器并可选择打开文件：

    (Cmd) edit foo.txt

使用的编辑器由 `editor` 可设置参数决定，可以是文本编辑器（如 **vim**）或图形编辑器（如 **VSCode**）。设置方式：

    set editor <program_name>

如果你设置了 `EDITOR` 环境变量，这将是 `editor` 的默认值。如果没有，`cmd2` 将尝试在你操作系统的常见编辑器列表中搜索。

## 终端分页器

任何命令的输出都可以使用 `cmd2.Cmd.ppaged` 方法逐页显示。

或者，可以使用 `!` 快捷方式直接调用终端分页器：

    (Cmd) !less foo.txt

:::warning
一旦你进入终端分页器，该程序就暂时控制了你的终端，**而不是** `cmd2`。通常你可以使用方向键或 `<PageUp>`/`<PageDown>` 键滚动，或输入 `q` 退出分页器并将控制权返回给 `cmd2` 应用。
:::

## 退出码

`cmd2` 应用的 `self.exit_code` 属性控制 `cmdloop()` 完成时返回的退出码。你的工作是确保在应用退出时通过调用 `sys.exit(app.cmdloop())` 将此退出码发送给 shell。

## 带参数调用

通常你会通过输入以下内容来调用 `cmd2` 程序：

    $ python mycmd2program.py

或：

    $ mycmd2program.py

这两种方法都会启动你的程序并进入 `cmd2` 命令循环，允许用户输入命令，然后由你的程序执行。

你可能希望在不提示用户输入任何内容的情况下在程序中执行命令。有几种方法可以完成此任务。最简单的一种是通过标准输入将命令及其参数管道传递给你的程序。你不需要对程序做任何事情就可以使用此技术。

    $ echo "speak -p some words" | python examples/cmd_as_argument.py
    omesay ordsway

使用相同的方法，你可以创建一个包含要运行的命令的文本文件，文件中每行一个命令。假设你的文件名为 `somecmds.txt`：

    c:\cmd2> type somecmds.txt | python.exe examples/cmd_as_argument.py
    omesay ordsway

默认情况下，`cmd2` 程序还会查找从操作系统 shell 传递的参数作为命令，并在进入命令循环之前执行这些命令：

    $ python examples/cmd_as_argument.py help

你可能需要对从操作系统 shell 传递的命令行参数有更多控制。例如，你可能有一个命令本身接受参数，甚至选项字符串。设置 `allow_cli_args=False` 你可以自行解析命令行：

    $ python examples/cmd_as_argument.py speak -p hello there
    ellohay heretay

或者，你可以简单地将命令加参数用引号括起来：

    $ python examples/cmd_as_argument.py "speak -p hello there"
    ellohay heretay
    (Cmd)

### 从其他 CLI/CLU 工具自动化 cmd2 应用

虽然 `cmd2` 旨在创建进入 Read-Evaluate-Print-Loop（REPL）的**交互式**命令行应用，但很多时候将 `cmd2` 应用作为一次性运行的命令行实用程序用于自动化和脚本目的非常有用。

这可以通过结合 `cmd2` 的以下功能轻松实现：

1. 带参数调用 `cmd2` 应用的能力
2. 离开 `cmd2` 应用时设置退出码的能力
3. 使用 `quit` 命令退出 `cmd2` 应用的能力

    $ python examples/cmd_as_argument.py "speak -p hello there" quit
    ellohay heretay
    $
