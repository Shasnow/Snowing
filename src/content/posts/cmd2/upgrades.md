---
title: Python cmd2：大版本升级指南
published: 2026-09-22
pinned: false
description: 介绍 cmd2 各大版本之间的升级要点，包括从 2.x 升级到 3.x 以及从 3.x 升级到 4.x 的破坏性变更与迁移方法。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# cmd2 大版本升级

## 从 3.x 升级到 cmd2 4.x

从 3.x 到 4.x 最大的变化是从 GNU Readline 库迁移到 [prompt-toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit) 来实现 REPL（读取-求值-打印循环）。这一变更提供了 readline 的纯 Python 替代品，完全跨平台兼容，并在功能和可扩展性方面带来了显著增强。

### prompt-toolkit 迁移

`cmd2` 现在使用 `prompt-toolkit` 处理所有输入、历史导航和 Tab 补全。这移除了对 GNU Readline 库（以及 macOS 上的 `gnureadline` 包）的依赖。

#### prompt-toolkit 的主要优势

- **跨平台**：在 Windows、macOS 和 Linux 上行为完全一致，无需外部依赖。
- **异步输出**：更好地支持向终端异步打印，同时不干扰用户的输入行。
- **增强的 UI**：支持底部工具栏、浮动菜单等高级 UI 元素。
- **多行编辑**：改进了对跨多行命令的编辑支持。

#### 破坏性变更与不兼容之处

尽管我们努力保持兼容，但仍存在一些差异：

- **键绑定**：键绑定现在由 `prompt-toolkit` 管理。虽然默认仍是 Emacs 风格绑定（与 readline 类似），但自定义方式改为使用 `prompt-toolkit` 的 [KeyBindings](https://python-prompt-toolkit.readthedocs.io/en/stable/pages/advanced_topics/key_bindings.html) API。
- **输入钩子**：不再支持 readline 特有的输入钩子。

### 底部工具栏

`cmd2` 现在支持可选的持久底部工具栏，可用于显示应用名称、当前状态甚至实时时钟等信息。

- **启用**：在 `cmd2.Cmd.__init__` 构造函数中设置 `enable_bottom_toolbar=True`。
- **自定义**：重写 `cmd2.Cmd.get_bottom_toolbar` 方法，返回你希望显示的内容。

参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例，了解如何实现定期刷新工具栏的后台线程。

### 自定义补全菜单颜色

`cmd2` 现在使用 `prompt-toolkit` 实现 Tab 补全菜单，并可通过 `cmd2` 主题自定义其外观。

- **自定义**：使用 `cmd2.theme.update_theme()` 覆盖 `Cmd2Style.COMPLETION_MENU_CURRENT` 和 `Cmd2Style.COMPLETION_MENU_META` 样式。详见[自定义补全菜单颜色](features/theme/#自定义补全菜单颜色)。

### 已删除的模块

移除了 `rl_utils.py` 和 `terminal_utils.py`，因为 `prompt-toolkit` 已提供这些功能。

## 从 2.x 升级到 cmd2 3.x

关于 3.0.0 版本所有变更的详细信息，请参阅 [CHANGELOG.md](https://github.com/python-cmd2/cmd2/blob/main/CHANGELOG.md)。

从 2.x 到 3.x 最大的变化是 `cmd2` 现在依赖 [rich](https://github.com/Textualize/rich)。相应地，`cmd2` 依靠 `rich` 在终端中进行精美的文本样式设置和格式化。因此，`cmd2` 中大量自定义代码被移除，其他内容也有所变动或改为基于 `rich` 实现。

升级到 3.x 时用户应注意的主要变化详述如下各小节。

### 已删除的模块

#### ansi

`cmd2.ansi` 模块中的功能或已被移除，或改为基于 `rich` 并移动到以下新模块之一：`cmd2.colors`、`cmd2.string_utils` 或 `cmd2.styles`。

为了简化从 `cmd2` 2.x 到 3.x 的迁移路径，我们以独立包的形式创建了 `cmd2-ansi` 模块，它是 `cmd2` 2.7.0 中 `cmd2.ansi` 模块的向后移植。相关链接：

- PyPI：[cmd2-ansi](https://pypi.org/project/cmd2-ansi/)
- GitHub：[cmd2-ansi](https://github.com/python-cmd2/cmd2-ansi)

使用此向后移植包：

```python
from cmd2_ansi import ansi
```

#### table_creator

`cmd2.table_creator` 模块已不复存在。更多信息请参阅 rich 关于 [Tables](https://rich.readthedocs.io/en/latest/tables.html) 的文档。[rich_tables.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_tables.py) 示例演示了如何在 `cmd2` 应用中使用 `rich` 表格。

`rich` 表格提供了比 `cmd2` 之前所提供的更强大的能力和灵活性。对于这一向后不兼容我们深表歉意，但两者的 API 根本不同，我们无法设计出一个向后兼容的包装层。

为了简化从 `cmd2` 2.x 到 3.x 的迁移路径，我们以独立包的形式创建了 `cmd2-table` 模块，它是 `cmd2` 2.7.0 中 `cmd2.table_creator` 模块的向后移植。相关链接：

- PyPI：[cmd2-table](https://pypi.org/project/cmd2-table/)
- GitHub：[cmd2-table](https://github.com/python-cmd2/cmd2-table)

使用此向后移植包：

```python
from cmd2_table import table_creator
```

### 新增模块

#### colors

新的 `cmd2.colors` 模块提供了便捷的 `cmd2.colors.Color` `StrEnum` 类，用于表示 `rich` 颜色名称。这让你可以在代码中使用可 Tab 补全的常量，而不是魔法字符串来表示所需的确切颜色。

参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 了解使用 `Color` 类为输出选择颜色的基础示例。或者参阅 [color.py](https://github.com/python-cmd2/cmd2/blob/main/examples/color.py) 示例，直观查看所有受支持的颜色。

#### rich_utils

新的 `cmd2.rich_utils` 模块提供了在 `cmd2` 应用中支持 `rich` 使用的通用工具类和函数。其中大部分内容并非面向最终用户。

许多 `cmd2` 应用开发者可能会感兴趣的是 `cmd2.theme.update_theme` 函数。参阅 [rich_theme.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_theme.py) 示例，了解如何为你的应用设置主题（配色方案）。

#### styles

错误消息等信息的默认显示样式现在位于新的 `cmd2.styles` 模块中，并且基于 `rich` 样式。

此前 `cmd2` 的默认样式位于 `cmd2.ansi` 模块中。

参阅 [argparse_completion.py](https://github.com/python-cmd2/cmd2/blob/main/examples/argparse_completion.py) 示例，了解如何在 `cmd2` 应用中利用这些默认样式来保持一致的外观。

#### string_utils

各种字符串工具函数已从 `cmd2.ansi` 模块移至新的 `cmd2.string_utils` 模块。

其中包括用于文本样式化、对齐以及加引号/去引号的函数。参阅 [getting_started.py](https://github.com/python-cmd2/cmd2/blob/main/examples/getting_started.py) 示例，了解如何使用通用的 `cmd2.string_utils.stylize` 函数。

#### terminal_utils

设置窗口标题、异步警报等终端控制转义序列的支持已从 `cmd2.ansi` 移入主 `cmd2.Cmd` 应用类中。

这部分并非供最终用户直接使用，而是供 `cmd2.Cmd.set_window_title` 和 `cmd2.Cmd.add_alert` 等面向最终用户的高层功能使用。

参阅 [async_printing.py](https://github.com/python-cmd2/cmd2/blob/main/examples/async_printing.py) 了解如何在 `cmd2` 应用中使用此功能。

### Argparse HelpFormatter 类

`cmd2` 现在有 5 个不同的 Argparse HelpFormatter 类，它们全部基于 [rich-argparse](https://github.com/hamdanal/rich-argparse) 的 `RichHelpFormatter` 类：

- `cmd2.rich_utils.Cmd2HelpFormatter`
- `cmd2.rich_utils.ArgumentDefaultsCmd2HelpFormatter`
- `cmd2.rich_utils.MetavarTypeCmd2HelpFormatter`
- `cmd2.rich_utils.RawDescriptionCmd2HelpFormatter`
- `cmd2.rich_utils.RawTextCmd2HelpFormatter`

此前默认的 `Cmd2HelpFormatter` 类继承自 `argparse.RawTextHelpFormatter`，但现在它继承自 `argparse.HelpFormatter`。如果你想要 RawText 行为，请向解析器传入 `formatter_class=RawTextCmd2HelpFormatter`。

好处是你的 `cmd2` 应用现在拥有更赏心悦目的帮助信息，包含颜色，能让你更快速、更轻松地目视解析帮助文本。这适用于所有受支持的 Python 版本。

### 其他变更

- `cmd2.Cmd.__init__` 的 `auto_load_commands` 参数现在默认为 `False`
- 用更 Python 风格的 `value` 属性取代了 `Settable.get_value()` 和 `Settable.set_value()` 方法
- 移除了 `with_argparser()` 装饰器中对解析器 `prog` 值的冗余设置，因为现在由 `Cmd._build_parser()` 统一处理
