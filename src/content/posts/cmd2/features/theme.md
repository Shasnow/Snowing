---
title: Python cmd2：主题
published: 2026-09-22
pinned: false
description: 介绍如何使用 cmd2.theme.update_theme 为应用配置整体主题，以及自定义 Tab 补全菜单颜色的方法。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 主题

`cmd2` 提供了使用 [cmd2.theme.update_theme][] 函数为应用配置整体主题的能力。这基于 [rich.theme](https://rich.readthedocs.io/en/stable/reference/theme.html) 样式信息容器。你可以使用它来为应用打造品牌，并设置对用户群体有吸引力的整体一致外观。

## 自定义补全菜单颜色

`cmd2` 利用 `prompt-toolkit` 实现其 Tab 补全菜单。你可以通过在 `cmd2` 主题中重写以下样式来自定义补全菜单的颜色：

- `Cmd2Style.COMPLETION_MENU` - 整个补全菜单容器的基础样式（设置背景）
- `Cmd2Style.COMPLETION_MENU_COMPLETION` - 单个未选中补全项的样式
- `Cmd2Style.COMPLETION_MENU_CURRENT` - 当前选中补全项的样式
- `Cmd2Style.COMPLETION_MENU_META` - 补全项旁边显示的"元信息"样式
- `Cmd2Style.COMPLETION_MENU_META_CURRENT` - 当前选中项的元信息样式

默认情况下，当前选中的补全项和元数据使用绿色背景上的黑色文本来提供对比度。所有其他项默认保持 `prompt-toolkit` 的默认值。然而，`cmd2` 应用作者可以自由地自定义这些样式，以匹配所需的视觉风格和/或品牌。

## 示例

参阅 [rich_theme.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_theme.py) 了解为 `cmd2` 应用配置自定义主题的简单示例。
