---
title: Python cmd2：表格创建
published: 2026-09-22
pinned: false
description: 说明自 cmd2 3.0.0 起表格创建交由 rich 库支持，并提供相关文档与示例链接。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 表格创建

从 3.0.0 版本开始，`cmd2` 不再包含自定义的表格创建代码。

这是因为 `cmd2` 现在依赖于 [rich](https://github.com/Textualize/rich)，它对这一功能有出色的支持。

请参阅 rich 的[表格文档](https://rich.readthedocs.io/en/latest/tables.html)了解更多信息。

[rich_tables.py](https://github.com/python-cmd2/cmd2/blob/main/examples/rich_tables.py) 示例演示了如何在 `cmd2` 应用中使用 `rich` 表格。
