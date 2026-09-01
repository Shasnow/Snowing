---
title: Python cmd2：命令历史功能详解
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的命令历史功能，包括开发者接口和用户操作方式，涵盖历史记录的保存、搜索、编辑和脚本导出等。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 命令历史

## 开发者指南

此前，`cmd2` 依赖 GNU Readline 库来处理命令历史。从 4.0.0 版本开始，`cmd2` 已迁移到 [prompt-toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit)，用于所有输入和历史记录的处理。

`cmd2.Cmd` 使用 `prompt-toolkit` 提供用户熟悉的命令行历史功能，同时维护自己的数据结构来存储用户输入的所有命令历史。当类被初始化时，它会创建一个 `cmd2.history.History` 类（`list` 的子类）的实例作为 `cmd2.Cmd.history`。

每次命令被执行时（这个过程比较复杂，具体时机请参阅[命令处理循环](../hooks/#命令处理循环)），解析后的 `cmd2.Statement` 对象会被追加到 `cmd2.Cmd.history` 中。

`cmd2` 通过 `cmd2.Cmd.__init__` 的可选参数提供了将历史记录持久化的功能。如果你在 `persistent_history_file` 参数中传入一个文件名，`cmd2.Cmd.history` 的内容将以压缩 JSON 格式写入该历史文件。我们选择这种格式而非纯文本，是为了保留每条命令的完整 `cmd2.Statement` 对象。

:::note
`prompt-toolkit` 会保存你输入的所有内容，无论是否为有效命令。而 `cmd2` 仅在命令解析成功且为有效命令时，才会将输入保存到内部历史记录中。这种设计是有意为之的，因为历史记录的内容可以保存为脚本文件，也可以重新执行。不保存无效输入可以减少重新执行时出现的意外错误。

然而，这种设计会导致当你输入无效命令时，`prompt-toolkit` 历史记录与 `cmd2` 历史记录之间出现不一致：无效命令会被保存到 `prompt-toolkit` 的历史记录中，但不会保存到 `cmd2` 的历史记录中。
:::

`cmd2.Cmd.history` 属性、`cmd2.history.History` 类和 `cmd2.history.HistoryItem` 类都是 `cmd2.Cmd` 公共 API 的一部分。你可以使用这些类来实现自己的 `history` 命令（关于内置 `history` 命令的工作方式，请参阅下文文档）。

## 用户指南

你可以使用 ↑ 和 ↓ 方向键在之前输入的命令历史中导航。

你可以按 `Control-p` 移动到上一条输入的命令，按 `Control-n` 移动到下一条命令。你还可以使用 `Control-r` 搜索命令历史。

默认情况下，`prompt-toolkit` 提供 Emacs 风格的快捷键绑定，这对 GNU Readline 库的用户来说会很熟悉。你可以参考 [readline 速查表](http://readline.kablamo.org/emacs.html)或深入阅读 [Prompt Toolkit 用户手册](https://python-prompt-toolkit.readthedocs.io/en/stable/pages/advanced_topics/key_bindings.html)获取所有细节，包括自定义快捷键绑定的说明。

`cmd2` 通过 `history` 命令提供了第三种历史访问方式。每次用户输入命令时，`cmd2` 都会保存该输入。`history` 命令让你可以对这些已保存的输入进行各种有趣的操作。以下示例均假设你已经输入了以下命令：

    (Cmd) alias create one !echo one
    Alias 'one' created
    (Cmd) alias create two !echo two
    Alias 'two' created
    (Cmd) alias create three !echo three
    Alias 'three' created
    (Cmd) alias create four !echo four
    Alias 'four' created

`history` 命令最简单的形式是显示之前输入的命令。不带任何参数时，它会显示所有之前输入的命令：

    (Cmd) history
        1  alias create one !echo one
        2  alias create two !echo two
        3  alias create three !echo three
        4  alias create four !echo four

如果提供一个正整数作为参数，则只显示指定的命令：

    (Cmd) history 4
        4  alias create four !echo four

如果提供一个负整数 _N_ 作为参数，则显示倒数第 _N_ 条命令。例如，`-1` 会显示你最后输入的命令，`-2` 会显示倒数第二条命令，以此类推：

    (Cmd) history -2
        3  alias create three !echo three

你可以使用类似的机制来显示一个范围内的命令。只需提供两个用 `..` 或 `:` 分隔的命令编号，你就能看到这两个编号之间（包含两端）的所有命令：

    (Cmd) history 1:3
        1  alias create one !echo one
        2  alias create two !echo two
        3  alias create three !echo three

如果省略第一个编号，将从头开始；如果省略最后一个编号，将继续到末尾：

    (Cmd) history :2
        1  alias create one !echo one
        2  alias create two !echo two
    (Cmd) history 2:
        2  alias create two !echo two
        3  alias create three !echo three
        4  alias create four !echo four

如果你想显示最后输入的三条命令：

    (Cmd) history -- -3:
        2  alias create two !echo two
        3  alias create three !echo three
        4  alias create four !echo four

注意这里的双横线 `--`。这是必需的，因为 history 命令使用 `argparse` 来解析命令行参数。正如 [argparse 文档](https://docs.python.org/3/library/argparse.html)中所述，`-3:` 会被解析为选项而非参数：

> 如果你的位置参数必须以 `-` 开头且看起来不像负数，你可以插入伪参数 `--`，它会告诉 `parse_args()` 在此之后的所有内容都是位置参数。

不存在第零条命令，所以不要去请求它。如果你是 Python 程序员，你可能已经注意到这很像列表和数组的切片语法。确实如此，唯一的区别是历史记录的第一条命令编号为 1，而 Python 数组的第一个元素索引为 0。

除了通过编号选择历史命令，你还可以搜索它们。可以使用简单的字符串搜索：

    (Cmd) history two
        2  alias create two !echo two

或者将正则表达式放在斜杠之间进行正则搜索：

    (Cmd) history '/te\ +th/'
        3  alias create three !echo three

如果你的正则表达式包含 `argparse` 会特殊处理的字符（如破折号或加号），还需要将正则表达式用引号括起来。

这些功能听起来很棒，但如果我们只能显示历史命令，这么多选择方式是不是有点多余？实际上，显示历史命令只是开始。history 命令还能执行许多其他操作：

- 运行之前输入的命令
- 将之前输入的命令保存到文本文件
- 在你喜欢的文本编辑器中打开之前输入的命令
- 运行之前输入的命令，并将命令及其输出保存到文本文件
- 清除已输入的命令历史

每种操作都通过命令行选项来调用。`-r` 或 `--run` 选项用于运行一条或多条之前输入的命令。运行第 1 条命令：

    (Cmd) history --run 1

重新运行最后两条命令（再次使用双横线来阻止 argparse 查找选项）：

    (Cmd) history -r -- -2:

假设你想重新运行一些之前输入的命令，但在运行前想做一些修改。当你使用 `-e` 或 `--edit` 选项时，`history` 会将选中的命令写入一个文本文件，并用文本编辑器打开该文件。你可以进行任何修改、添加或删除操作。当你退出文本编辑器后，文件中的所有命令都会被执行。要编辑并重新运行第 2-4 条命令：

    (Cmd) history --edit 2:4

如果你想将命令保存到文本文件，但不编辑和重新运行它们，可以使用 `-o` 或 `--output-file` 选项。这是创建[脚本](../scripting/)的好方法，脚本可以通过 `run_script` 命令执行。要将本次会话中输入的前 5 条命令保存到文本文件：

    (Cmd) history :5 -o history.txt

history 命令可以执行的最后一个操作是使用 `-c` 或 `--clear` 来清除命令历史：

    (Cmd) history -c

除了这五种操作外，`history` 命令还有一些选项来控制输出格式。不带参数时，`history` 命令会在每条命令前显示命令编号。这在屏幕显示历史记录时很有用，因为它为你提供了一个便捷的参考标识来识别之前输入的命令。但是，在创建脚本时，命令编号会阻止脚本正常加载。`-s` 或 `--script` 选项指示 `history` 命令不显示行号。该选项会被 `--output-file` 和 `--edit` 选项自动设置。如果你想将带行号的历史命令输出到文件，可以使用输出重定向：

    (Cmd) history 1:4 > history.txt

你也可以单独使用 `-s` 或 `--script` 来在屏幕上显示不带行号的历史命令，以便复制到剪贴板：

    (Cmd) history -s 1:3

`cmd2` 支持别名和宏，它们允许你将一个简短、方便的输入字符串替换为更长的字符串。假设我们创建了这样一个别名，然后使用它：

    (Cmd) alias create ls shell ls -aF
    Alias 'ls' created
    (Cmd) ls -d h*
    history.txt     htmlcov/

默认情况下，`history` 命令显示的正是我们输入的内容：

    (Cmd) history
        1  alias create ls shell ls -aF
        2  ls -d h*

有两种方式可以修改显示内容，让你看到别名和宏展开后的结果。第一种是使用 `-x` 或 `--expanded`。这些选项显示展开后的命令而非输入的命令：

    (Cmd) history -x
        1  alias create ls shell ls -aF
        2  shell ls -aF -d h*

如果你想同时看到输入的命令和展开后的命令，使用 `-v` 或 `--verbose` 选项：

    (Cmd) history -v
        1  alias create ls shell ls -aF
        2  ls -d h*
        2x shell ls -aF -d h*

如果输入的命令没有展开操作，则照常显示。但如果由于宏和别名的展开导致内容发生了变化，则输入的命令会带有编号显示，展开后的命令会带有编号加 `x` 后缀显示。
