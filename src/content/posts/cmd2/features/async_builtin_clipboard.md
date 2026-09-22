---
title: Python cmd2：异步命令、内置命令与剪贴板集成
published: 2026-05-28
pinned: false
description: 介绍 cmd2 的异步命令实现、内置命令列表及其用法，以及剪贴板集成功能。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 异步命令

`cmd2` 构建在 Python 标准库的 `cmd` 模块之上，该模块本质上是同步的。这意味着 `do_*` 命令方法必须是同步函数。

不过，你仍然可以通过在后台线程中运行 `asyncio` 事件循环并桥接调用的方式，将异步代码（使用 `asyncio` 和 `async`/`await`）集成到你的 `cmd2` 应用中。

## `with_async_loop` 装饰器

处理此问题的一种简洁方式是定义一个装饰器来包装你的 `async def` 命令。该装饰器负责：

1. 启动一个带有 `asyncio` 事件循环的后台线程（如果尚未运行）
2. 将命令的协程提交到该事件循环
3. 同步等待结果，以便 `cmd2` 接口按预期工作（阻塞直到命令完成）

### 实现示例

以下是实现此类装饰器并在应用中使用的示例。

```python
import asyncio
import functools
import threading
from typing import Any, Callable
import cmd2

# Global event loop and lock
_event_loop = None
_event_lock = threading.Lock()

def _get_event_loop() -> asyncio.AbstractEventLoop:
    """Get or create the background event loop."""
    global _event_loop

    if _event_loop is None:
        with _event_lock:
            if _event_loop is None:
                _event_loop = asyncio.new_event_loop()
                thread = threading.Thread(
                    target=_event_loop.run_forever,
                    name='Async Runner',
                    daemon=True,
                )
                thread.start()
    return _event_loop

def with_async_loop(func: Callable[..., Any]) -> Callable[..., Any]:
    """Decorator to run a command method asynchronously in a background thread."""
    @functools.wraps(func)
    def wrapper(self: cmd2.Cmd, *args: Any, **kwargs: Any) -> Any:
        loop = _get_event_loop()
        coro = func(self, *args, **kwargs)
        future = asyncio.run_coroutine_threadsafe(coro, loop)
        return future.result()
    return wrapper

class AsyncApp(cmd2.Cmd):
    @with_async_loop
    async def do_my_async(self, _: cmd2.Statement) -> None:
        self.poutput("Starting async work...")
        await asyncio.sleep(1.0)
        self.poutput("Async work complete!")
```

## 另请参阅

- [async_commands.py](https://github.com/python-cmd2/cmd2/blob/main/examples/async_commands.py) - 完整示例代码
- [async_call.py](https://github.com/python-cmd2/cmd2/blob/main/examples/async_call.py) - 展示如何在不使用装饰器的情况下进行单个异步调用的替代示例

# 内置命令

继承 `cmd2.Cmd` 的应用程序会自动获得一系列可能对用户有用的命令。如果开发者不希望某些命令成为应用的一部分，可以[移除内置命令](#移除内置命令)。

## 内置命令列表

### alias

此命令通过子命令 `create`、`delete` 和 `list` 来管理别名。详见[别名](../shortcuts_aliases_macros/#别名)。

### edit

此命令启动编辑器程序并指示其打开指定的文件名。示例：

```sh
(Cmd) edit ~/.ssh/config
```

启动的程序由 [editor](../settings_plugins/#editor) 设置的值决定。

### help

此命令列出可用命令或提供特定命令的详细帮助。使用 `-v/--verbose` 参数调用时，会显示每个命令的简要描述。详见[帮助](../embedded_output_help/#帮助)。

### history

此命令允许你查看、运行、编辑、保存或清除之前输入的命令历史。详见[命令历史](../history/#命令历史)。

### ipy（可选）

此可选命令会进入一个交互式 IPython shell。详见 [IPython（可选）](../embedded_output_help/#ipython可选)。

### macro

此命令通过子命令 `create`、`delete` 和 `list` 来管理宏。宏类似于别名，但可以包含参数占位符。详见[宏](../shortcuts_aliases_macros/#宏)。

### py（可选）

此可选命令调用 Python 命令或 shell。详见[嵌入式 Python Shell](../embedded_output_help/#嵌入式-python-shell)。

### quit

此命令退出 `cmd2` 应用。

### run_pyscript

此命令在 `cmd2` 应用内部运行 Python 脚本文件。详见 [Python 脚本](../embedded_output_help/#python脚本)。

### run_script

此命令运行以 ASCII 或 UTF-8 文本编码的脚本文件中的命令。详见[命令脚本](../scripting/#命令脚本)。

### _relative_run_script

**此命令对最终用户是隐藏的。** 它类似于 [run_script](#run_script)，但使用相对于当前正在执行的脚本的路径。这在你有脚本运行其他脚本的情况下非常有用。详见[运行命令脚本](../scripting/#运行命令脚本)。

### set

所有用户可设置的参数及其简要注释可以在运行中的应用内查看：

```text
(Cmd) set
  Name                            Value      Description
 allow_style                     Terminal   Allow ANSI text style sequences in output (valid values: Always, Never, Terminal)
 debug                           False      Show full traceback on exception
 echo                            False      Echo command issued into output
 editor                          vim        Program used by 'edit'
 max_column_completion_results   7          Maximum number of completion results to display in a single column
 max_completion_table_items      50         Maximum number of completion results allowed for a completion table to appear
 quiet                           False      Don't print nonessential feedback
 scripts_add_to_history          True       Scripts and pyscripts add commands to history
 timing                          False      Report execution times
 traceback_show_locals           False      Display local variables in tracebacks
```

在运行应用时，可以使用 `set` 命令来设置这些用户可设置参数：

```text
(Cmd) set allow_style Never
```

详见[设置](../settings_plugins/#设置)。

### shell

在操作系统 shell 提示符下执行命令：

```text
(Cmd) shell pwd -P
/usr/local/bin
```

### shortcuts

此命令列出可用的快捷方式。详见[快捷方式](../shortcuts_aliases_macros/#快捷方式)。

## 移除内置命令

开发者可能不希望向应用用户提供 `cmd2.Cmd` 中的所有内置命令。要移除某个命令，你需要在运行时从 `cmd2.Cmd` 对象中删除实现该命令的方法。例如，如果你想从应用中移除 [shell](#shell) 命令：

```py
class NoShellApp(cmd2.Cmd):
    """A simple cmd2 application."""

    delattr(cmd2.Cmd, 'do_shell')
```

# 剪贴板集成

几乎每个操作系统都有某种短期存储区域的概念，可以被任何程序访问。通常这被称为剪贴板，但有时人们也称其为粘贴缓冲区。

`cmd2` 使用 [pyperclip](https://github.com/asweigart/pyperclip) 模块与操作系统剪贴板集成。命令输出可以通过在命令末尾添加大于号来发送到剪贴板：

```text
mycommand args >
```

可以将其理解为将输出重定向到一个临时的、匿名的位置：剪贴板。你也可以通过在命令末尾添加两个大于号来将输出追加到剪贴板的当前内容：

```text
mycommand arg1 arg2 >>
```

:::warning
命令输出重定向功能仅直接在 `cmd2` 命令中有效。在将输出通过管道传递给 shell 命令后，该功能将不起作用。
:::

## 开发者指南

你可以通过设置 `cmd2.Cmd.allow_clipboard` 属性来控制是否允许用户使用上述将输出添加到操作系统剪贴板的功能。默认值为 `True`。将其设为 `False` 后，上述功能将生成错误消息而不是将输出添加到剪贴板。`cmd2.Cmd.allow_clipboard` 可以在初始化时设置，也可以在代码中随时更改。

如果你希望基于 `cmd2` 的应用能够以其他或替代方式使用剪贴板，可以使用 `cmd2.clipboard` 模块中提供的方法（在 Windows、macOS 和 Linux 上的工作方式一致）。
