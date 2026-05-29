---
title: Python cmd2：杂项功能与应用打包分发
published: 2026-05-28
pinned: false
description: 介绍 cmd2 的杂项功能（计时器、退出方式、select 选择器、命令禁用）以及应用打包分发的各种方式。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
date: 2026-05-28
pubDate: 2026-05-28
---

# 杂项功能

## 计时器

开启计时器设置后，`cmd2` 将显示每个命令执行所花费的墙钟时间。

## 退出方式

与许多 shell 应用类似，`cmd2` 应用可以通过在空行上按 `Ctrl-D` 或执行 `quit` 命令来退出。

## select

向用户显示编号选项，类似于 bash 的 `select`。

`app.select` 是在方法内部调用的（不是由用户直接调用；它是 `app.select`，而不是 `app.do_select`）。

```py
def do_eat(self, arg):
    sauce = self.select('sweet salty', 'Sauce? ')
    result = '{food} with {sauce} sauce, yum!'
    result = result.format(food=arg, sauce=sauce)
    self.stdout.write(result + '\n')
```

```text
(Cmd) eat wheaties
    1. sweet
    2. salty
Sauce? 2
wheaties with salty sauce, yum!
```

参阅 [read_input.py](https://github.com/python-cmd2/cmd2/blob/main/examples/read_input.py) 文件中的 `do_eat` 方法了解如何使用 `select` 的示例。

## 命令禁用

`cmd2` 支持在运行时禁用命令。这在某些命令仅在应用处于特定状态时才应可用的情况下非常有用。当命令被禁用时，它不会出现在帮助菜单中或被 Tab 补全。如果用户尝试运行该命令，将打印开发者提供的特定消息。以下函数支持此功能：

- `enable_command`：启用单个命令
- `enable_category`：启用整个分类的命令
- `disable_command`：禁用单个命令并设置在禁用期间运行该命令或请求帮助时打印的消息
- `disable_category`：禁用整个分类的命令并设置在禁用期间运行该分类中的任何命令或请求帮助时打印的消息

参阅 [help_categories.py](https://github.com/python-cmd2/cmd2/blob/main/examples/help_categories.py) 示例中的 `do_enable_commands()` 和 `do_disable_commands()` 函数了解演示。

## 表格创建

从 3.0.0 版本开始，`cmd2` 不再包含自定义的表格创建代码。

这是因为 `cmd2` 现在依赖于 [rich](https://github.com/Textualize/rich)，它对这一功能有出色的支持。

请参阅 rich 的[表格文档](https://rich.readthedocs.io/en/latest/tables.html)了解更多信息。

[rich_tables.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_tables.py) 示例演示了如何在 `cmd2` 应用中使用 `rich` 表格。

## 主题定制

`cmd2` 提供了使用 `cmd2.theme.update_theme` 函数为应用配置整体主题的能力。这基于 [rich.theme](https://rich.readthedocs.io/en/stable/reference/theme.html) 样式信息容器。你可以使用它来为应用打造品牌，并设置对用户群体有吸引力的整体一致外观。

## 自定义补全菜单颜色

`cmd2` 利用 `prompt-toolkit` 实现其 Tab 补全菜单。你可以通过在 `cmd2` 主题中重写以下样式来自定义补全菜单的颜色：

- `Cmd2Style.COMPLETION_MENU` - 整个补全菜单容器的基础样式（设置背景）
- `Cmd2Style.COMPLETION_MENU_COMPLETION` - 单个未选中补全项的样式
- `Cmd2Style.COMPLETION_MENU_CURRENT` - 当前选中补全项的样式
- `Cmd2Style.COMPLETION_MENU_META` - 补全项旁边显示的"元信息"样式
- `Cmd2Style.COMPLETION_MENU_META_CURRENT` - 当前项的元信息样式

默认情况下，当前选中的补全项和元数据使用绿色背景上的黑色文本来提供对比度。所有其他项默认保持 `prompt-toolkit` 的默认值。然而，`cmd2` 应用作者可以自由自定义这些样式，以匹配所需的视觉风格和/或品牌。

## 示例

参阅 [rich_theme.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_theme.py) 了解为 `cmd2` 应用配置自定义主题的简单示例。

# 应用打包分发

作为构建交互式命令行应用的通用工具，`cmd2` 被设计为可以多种方式使用。如何将 `cmd2` 应用分发给客户或最终用户取决于你。参阅 Python 打包权威机构的 [Python 打包概述](https://packaging.python.org/overview/)了解 Python 生态系统中各种选项的详细讨论。

## 发布到 Python 包索引（PyPI）

最简单的方式是按照[打包 Python 项目](https://packaging.python.org/en/latest/tutorials/packaging-projects/)教程操作。这将展示如何将应用打包为 Python 包并上传到 Python 包索引（[PyPI](https://pypi.org/)）。发布后，用户将能够使用标准 Python 打包工具（如 [pip](https://pip.pypa.io/) 或 [uv](https://github.com/astral-sh/uv)）来安装它。

对此过程的小幅调整可以让你发布到私有 PyPI 镜像，如托管在 [AWS CodeArtifact](https://aws.amazon.com/codeartifact/) 上的镜像或私有 [Artifactory](https://jfrog.com/artifactory/) 服务器。

## 使用 Docker 在容器中打包应用

将 Python 应用打包在 [Docker](https://www.docker.com/) 容器中在跨平台移植性和便利性方面非常出色，因为该容器将包含应用的所有依赖项，并在不会与操作系统依赖项冲突的隔离环境中运行它们。

这篇方便的博客文章将展示如何 [Docker 化你的 Python 应用](https://www.docker.com/blog/how-to-dockerize-your-python-applications/)。

## 将应用与 Python 一起打包为安装程序

对于希望将 `cmd2` 应用打包为单个二进制镜像或压缩文件的开发者，我们可以根据个人和专业经验推荐以下工具：

- [Nuitka](https://github.com/Nuitka/Nuitka)
    - Nuitka 是一个用 Python 编写的 Python 编译器
    - 你将 Python 应用提供给它，它会做很多聪明的事情，然后输出一个可执行文件或扩展模块
    - 如果你有希望通过混淆应用背后的 Python 源代码来保护的知识产权，这特别方便
- [PyInstaller](https://www.pyinstaller.org)
    - 将 Python 程序冻结（打包）为独立可执行文件
    - PyInstaller 将 Python 应用及其所有依赖项打包为单个包
    - 用户可以运行打包的应用而无需安装 Python 解释器或任何模块
- [PyOxidizer](https://github.com/indygreg/PyOxidizer)
    - 用 Rust 实现的现代 Python 应用打包和分发工具
    - 用于生成嵌入 Python 和所有依赖项的二进制文件的实用工具
    - 你可以将单个可执行文件复制到另一台机器上并运行其中包含的 Python 应用，开箱即用
