---
title: Python cmd2：脚本功能详解
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的脚本功能，包括命令脚本和 Python 脚本的创建、运行，以及 cmd2 API 的开发原则和高级用法。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 脚本

操作系统 shell 长期以来一直能够执行保存在文本文件中的命令序列。这些脚本文件简化了长命令序列的重复执行。`cmd2` 支持两种类似的机制：命令脚本和 Python 脚本。

## 命令脚本

命令脚本包含在 `cmd2` 应用提示符处输入的命令序列。与操作系统 shell 脚本不同，命令脚本不能包含逻辑或循环。

### 创建命令脚本

命令脚本可以通过以下几种方式创建：

- 使用你选择的任何方法创建文本文件
- 使用内置的 [edit](../async_builtin_clipboard/#edit) 命令创建或编辑现有文本文件
- 使用 [history -s](../history/#用户指南) 将之前输入的命令保存到脚本文件

如果你从头创建文本文件，只需每行包含一个命令，就像你在 `cmd2` 应用中输入的那样。

### 运行命令脚本

命令脚本文件可以使用内置的 [run_script](../async_builtin_clipboard/#run_script) 命令或 `@` 快捷方式（如果你的应用使用默认快捷方式）执行。支持 ASCII 和 UTF-8 编码的 Unicode 文本文件。[run_script](../async_builtin_clipboard/#run_script) 命令支持文件系统路径的 Tab 补全。还有一个变体命令 `\_relative_run_script` 或 `@@` 快捷方式（如果使用默认快捷方式），用于在脚本中使用相对于第一个脚本的路径。

### 注释

如果第一个非空白字符是 `#`，则命令行是注释。这意味着命令中后面出现的任何 `#` 字符都将被视为字面量。这同样适用于多行命令中间的 `#`，即使它是行的第一个字符。

注释在脚本中很有用，但在交互式会话中通常不使用。

    (Cmd) # this is a comment
    (Cmd) command # this is not a comment

## Python 脚本

如果你需要逻辑流、循环、分支或其他高级功能，可以编写一个在 `cmd2` 应用上下文中执行的 Python 脚本。此脚本使用 [run_pyscript](../async_builtin_clipboard/#run_pyscript) 命令运行。以下是使用 [arg_printer.py](https://github.com/python-cmd2/cmd2/blob/main/examples/scripts/arg_printer.py) pyscript 的简单示例：

    (Cmd) run_pyscript examples/scripts/arg_printer.py foo bar 'baz 23'
    Running Python script 'arg_printer.py' which was called with 3 arguments
    arg 1: 'foo'
    arg 2: 'bar'
    arg 3: 'baz 23'

[run_pyscript](../async_builtin_clipboard/#run_pyscript) 支持文件系统路径的 Tab 补全，并且如上所示，它具有向调用的脚本传递命令行参数的能力。

## 开发 cmd2 API

如果你作为应用设计者没有显式禁用 `run_pyscript` 命令，你应该假设你的应用将被用于更高级别的 Python 脚本。以下部分作为指南，突出 API 功能的生产和消费中可能的陷阱。为清晰起见，"脚本编写者"编写 pyscripts，"设计者"是 `cmd2` 应用作者。

### 基础

无需设计者做任何工作，脚本编写者就可以利用简单的 `app` 调用来组合工作流。`run_pyscript` app 调用的结果产生一个 `CommandResult` 对象，暴露四个成员：`stdout`、`stderr`、`stop` 和 `data`。

`stdout` 和 `stderr` 是相当直接的普通数据流表示，准确反映用户在正常 cmd2 交互期间看到的内容。`stop` 包含有关调用的命令如何结束其生命周期的信息。最后 `data` 包含设计者通过 `self.last_result`（如果从 CommandSet 内部调用则为 `self._cmd.last_result`）设置的任何信息。

使用 [run_pyscript](../async_builtin_clipboard/#run_pyscript) 执行的 Python 脚本可以通过以下语法运行 `cmd2` 应用命令：

```py
app('command args')
```

其中：

- `app` 是一个可配置的名称，可以通过设置 `cmd2.Cmd.py_bridge_name` 属性来更改
- `command` 和 `args` 的输入方式与应用用户的输入方式完全相同

使用 f-字符串通常是最直接和易读的参数提供方式：

```py
first = 'first'
second = 'second'

app(f'command {first} -t {second}')
```

参阅 [python_scripting.py](https://github.com/python-cmd2/cmd2/blob/main/examples/python_scripting.py) 示例和相关的 [conditional.py](https://github.com/python-cmd2/cmd2/blob/main/examples/scripts/conditional.py) 脚本了解更多信息。

### 设计原则

如果你的 cmd2 应用遵循 [Unix 设计哲学](https://en.wikipedia.org/wiki/Unix_philosophy)，脚本编写者将拥有最大的灵活性来使用不同命令创建工作流。如果设计者的应用更完整且不太可能在未来被扩展，脚本编写者可以使用简单的串行脚本，只需很少的控制流。无论哪种情况，设计者做出的选择都会对脚本编写者产生影响。

:::note
作为设计者，你应该从内向外设计。你的代码在较低级别上进行单元测试要比在高级别上容易得多。虽然有 cmd2 的回归测试扩展，但单元测试在开发中总是更快。
:::

:::warning
高级 pyscript 了解甚至访问应用的低级类库是糟糕的设计。除非必要，否则请尽可能抵制这种冲动。
:::

### 开发基础 API

默认情况下，`cmd2` 允许脚本编写者利用所有暴露的 `do_*` 命令。作为脚本编写者，你可以通过 `stdout` 和 `stderr` 轻松与应用交互。

`cmd2` pyscripts 首先需要**有效的** Python 代码。

:::warning
一个常见的误解是所有应用异常都会从下面传播上来。事实并非如此。`cmd2` 捕获所有应用异常，没有方法来处理它们。
:::

当执行不带参数的 `speak` 命令时，你会看到以下错误：

    (Cmd) speak
    Usage: speak [-h] [-p] [-s] [-r REPEAT] words [...]
    Error: the following arguments are required: words

即使这是一个完全合格的 `cmd2` 错误，pyscript 也必须检查此错误并执行错误检查：

```py
result = app('speak')

if not result:
    print(result.stderr)
```

在 Python 开发中，最好在用户输入后快速失败：

```py
import sys

result = app('speak TRUTH!!')

if not result:
    print("Something went wrong")
    sys.exit()

print("Continuing along..")
```

通过仅使用 `stdout` 和 `stderr`，可以使用基本控制流来链接命令。

### 开发高级 API

到目前为止，我们还没有关注脚本编写者的需求。如果在创建 pyscripts 时不必从 `stdout` 解析数据，那不是很好吗？我们可以通过在 `do_*` 命令末尾添加一小行来满足脚本编写者的需求。

`self.last_result = <value>`

添加上述行可以增强 cmd2 应用并开启一个新的可能性世界。

:::tip
在 CommandSet 内部为命令函数设置结果时，请使用私有 cmd 实例：

```py
self._cmd.last_result = <value>
```
:::

**设计者应返回简单的标量类型作为命令结果，而不是复杂对象。** 如果提供类对象有益，设计者应选择不可变类型而非可变类型，并且永远不要提供对类成员的直接访问，因为这可能导致违反[开闭原则](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)。

在可能的情况下，frozen dataclass 是数据操作的理想轻量级解决方案。

以下应用有两个命令：`build` 和 `status`。假设构建操作发生在 REST API 端点的其他地方，并且有显著的计算成本。status 命令将只显示构建任务的当前状态。应用已经提供了用户启动构建和确定其状态所需的一切。然而，问题在于对于长时间运行的进程，用户可能希望等待其完成。设计者可能想要创建一个命令来启动构建然后轮询状态直到完成，但这种情况最好作为脚本来解决。

app.py:

```py
#!/usr/bin/env python
"""A simple cmd2 application."""
import sys
from dataclasses import dataclass
from random import choice, randint
from typing import Optional

import cmd2
from cmd2.parsing import Statement


@dataclass(frozen=True)
class BuildStatus:
    id: int
    name: str
    status: str


class FirstApp(cmd2.Cmd):
    """A simple cmd2 application."""

    def __init__(self):
        self._status_cache = dict()

    def _start_build(self, name: str) -> BuildStatus:
        return BuildStatus(randint(10, 100), name, "Started")

    def _get_status(self, name: str) -> Optional[BuildStatus]:
        status = self._status_cache.get(name)
        status_types = ["canceled", "restarted", "error", "finished"]

        if status.status != "finished":
            status = BuildStatus(status.id, status.name, choice(status_types))
            self._status_cache[name] = status

        return status

    build_parser = cmd2.Cmd2ArgumentParser()
    build_parser.add_argument("name", help="Name of build to start")

    @cmd2.with_argparser(build_parser)
    def do_build(self, args: Statement):
        """Executes a long running process at an API endpoint"""
        status = self._start_build(args.name)
        self._status_cache[args.name] = status
        self.poutput(f"Build {args.name.upper()} successfully started with id : {status.id}")
        self.last_result = status

    status_parser = cmd2.Cmd2ArgumentParser()
    status_parser.add_argument("name", help="Name of build determine status of")

    @cmd2.with_argparser(status_parser)
    def do_status(self, args: Statement):
        """Shows the current status of a build"""
        status = self._get_status(args.name)
        self.poutput(f"Status for Build: {args.name} \n {status.status}")
        self.last_result = status


if __name__ == "__main__":
    import sys
    c = FirstApp()
    sys.exit(c.cmdloop())
```

以下是通过 pyscript 的可能解决方案：

```py
import sys
import time

# start build
result = app('build tower')

# If there was an error then exit
if not result:
    print('Build failed')
    sys.exit()

# This is a BuildStatus dataclass object
build = result.data

print(f"Build {build.name} : {build.status}")

# Poll status
while True:
    # Perform status check
    result = app('status tower')

    # error checking
    if not result:
        print("Unable to determine status")
        break

    build_status = result.data

    # If the status shows complete then the script is done
    if build_status.status in ['finished', 'canceled']:
        print(f"Build {build.name} has completed")
        break

    print(f"Current Status: {build_status.status}")
    time.sleep(1)
```
