---
title: Python cmd2：钩子机制详解
published: 2026-05-28
pinned: false
description: 详细介绍 cmd2 的钩子机制，包括应用生命周期钩子、命令处理循环中的各类钩子（解析后、命令前、命令后、命令终结）及其使用方法。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 钩子

启动 `cmd2` 应用的典型方式如下：

```py
import cmd2
class App(cmd2.Cmd):
    # customized attributes and methods here

if __name__ == '__main__':
    app = App()
    app.cmdloop()
```

有多个现成的方法和属性可供你调整，以控制应用在命令处理循环之前、期间和之后的整体行为。

## 应用生命周期钩子

你可以通过在 `cmd2.Cmd.__init__` 的 `startup_script` 参数中传入脚本文件名，在初始化时运行脚本。

你还可以注册在命令循环开始时调用的方法：

```py
    class App(cmd2.Cmd):
        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            self.register_preloop_hook(self.myhookmethod)

        def myhookmethod(self) -> None:
            self.poutput("before the loop begins")
```

为了保持与 `cmd.Cmd` 的向后兼容性，在所有已注册的 preloop 钩子被调用之后，`cmd2.Cmd.preloop` 方法会被调用。

类似的方式允许你注册在命令循环结束后调用的函数：

```py
    class App(cmd2.Cmd):
        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            self.register_postloop_hook(self.myhookmethod)

        def myhookmethod(self) -> None:
            self.poutput("after the loop ends")
```

为了保持与 `cmd.Cmd` 的向后兼容性，在所有已注册的 postloop 钩子被调用之后，`cmd2.Cmd.postloop` 方法会被调用。

Preloop 和 postloop 钩子方法不接收任何参数，任何返回值都会被忽略。

注册钩子而非重写方法的方式允许多个钩子在命令循环开始或结束时被调用。插件作者应仔细完整地阅读本页以了解编写钩子的最佳实践。

## 应用生命周期属性

`cmd2.Cmd` 上有许多属性会影响进入或运行命令循环时的应用行为：

- `cmd2.Cmd.intro` - 如果提供，它会作为启动横幅在应用启动时打印一次，在 `cmd2.Cmd.preloop` 被调用之后
- `cmd2.Cmd.prompt` - 详见[提示符](../prompt_redirection/#提示符)
- `cmd2.Cmd.continuation_prompt` - 在[多行命令](../app_setup/#多行命令)的第 2 行及后续行请求输入时显示的提示符
- `cmd2.Cmd.echo` - 如果为 `True`，将提示符和命令写入输出流

此外，`cmd2.Cmd.__init__` 的几个参数也会影响命令循环行为：

- `allow_cli_args` - 允许在操作系统命令行上指定命令，这些命令会在命令处理循环开始之前执行
- `startup_script` - 在初始化时运行脚本。详见[脚本](../scripting/)

## 命令处理循环

当你调用 `cmd2.Cmd.cmdloop` 时，以下事件序列会重复执行直到应用退出：

1. 启动 `prompt-toolkit` 事件循环
2. 调用 `cmd2.Cmd.pre_prompt`，用于在事件循环启动后但提示符显示之前执行任何行为
3. 输出提示符
4. 接受用户输入
5. 将用户输入解析为 `cmd2.Statement` 对象
6. 调用通过 `cmd2.Cmd.register_postparsing_hook` 注册的方法
7. 如果用户请求且允许，重定向输出
8. 启动计时器
9. 调用通过 `cmd2.Cmd.register_precmd_hook` 注册的方法
10. 调用 `cmd2.Cmd.precmd` - 为了与 `cmd.Cmd` 向后兼容
11. 将语句添加到[命令历史](../history/)
12. 调用 `do_command` 方法
13. 调用通过 `cmd2.Cmd.register_postcmd_hook` 注册的方法
14. 调用 `cmd2.Cmd.postcmd` - 为了与 `cmd.Cmd` 向后兼容
15. 停止计时器并显示经过的时间
16. 如果输出被重定向，停止重定向
17. 调用通过 `cmd2.Cmd.register_cmdfinalization_hook` 注册的方法

通过注册钩子方法，你可以在命令处理循环的多个步骤中运行代码并控制流程。请注意，插件也会使用这些钩子，因此可能会有不属于你应用代码的代码在运行。为某个钩子注册的方法按注册顺序调用。你可以多次注册同一个函数，它会在每次注册时都被调用。

解析后、命令前和命令后钩子方法共享一些影响命令处理循环的通用方式。

如果钩子引发异常：

- 不会再调用任何类型的钩子（命令终结钩子除外）
- 如果命令尚未执行，它将不会被执行
- 异常消息将显示给用户

特定类型的钩子方法有额外的选项，如下所述。

## 解析后钩子

解析后钩子在用户输入被解析后但在命令执行之前调用。这些钩子可用于：

- 修改用户输入
- 在每个命令执行前运行代码
- 取消当前命令的执行
- 退出应用

当解析后钩子被调用时，输出尚未被重定向，命令执行的计时器也尚未启动。

要定义和注册解析后钩子，请按以下方式操作：

```py
class App(cmd2.Cmd):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.register_postparsing_hook(self.myhookmethod)

    def myhookmethod(self, params: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
        # the statement object created from the user input
        # is available as params.statement
        return params
```

`cmd2.Cmd.register_postparsing_hook` 会检查传入可调用对象的方法签名，如果参数数量错误会引发 `TypeError`。如果传入的参数和返回值没有标注为 `PostparsingData`，也会引发 `TypeError`。

钩子方法会接收一个参数，一个 `cmd2.plugin.PostparsingData` 对象，我们称之为 `params`。`params` 包含两个属性。`params.statement` 是一个 `cmd2.Statement` 对象，描述了解析后的用户输入。`cmd2.Statement` 对象中有许多有用的属性，包括 `.raw`（包含用户输入的完整内容）。`params.stop` 默认设为 `False`。

钩子方法必须返回一个 `cmd2.plugin.PostparsingData` 对象，最方便的方式是直接返回传入钩子方法的对象。钩子方法可以修改对象的属性来影响应用的行为。如果 `params.stop` 被设为 `True`，则在命令执行之前会触发致命失败，应用将退出。

要修改用户输入，你需要创建一个新的 `cmd2.Statement` 对象并将其赋值给 `params.statement`。不要尝试直接修改 `cmd2.Statement` 对象的内容，那里有陷阱。相反，使用 `cmd2.Statement` 对象中的各种属性来构造新字符串，然后解析该字符串以创建新的 `cmd2.Statement` 对象。

`cmd2.Cmd` 使用 `cmd2.parsing.StatementParser` 的实例来解析用户输入。该实例已配置了正确的命令终止符、多行命令和其他解析相关设置。该实例可通过 `cmd2.Cmd.statement_parser` 属性访问。以下是一个展示正确技术的简单示例：

```py
def myhookmethod(self, params: cmd2.plugin.PostparsingData) -> cmd2.plugin.PostparsingData:
    if not '|' in params.statement.raw:
        newinput = params.statement.raw + ' | less'
        params.statement = self.statement_parser.parse(newinput)
    return params
```

如果解析后钩子返回的 `cmd2.plugin.PostparsingData` 对象的 `stop` 属性被设为 `True`：

- 不会再调用任何类型的钩子（[命令终结钩子](#命令终结钩子)除外）
- 命令将不会被执行
- 不会向用户显示错误消息
- 应用将退出

## 命令前钩子

命令前钩子可以修改用户输入，但不能请求应用终止。如果你的钩子需要能够退出应用，应将其实现为解析后钩子。

一旦输出被重定向且计时器启动，所有通过 `cmd2.Cmd.register_precmd_hook` 注册的钩子都会被调用。操作方式如下：

```py
class App(cmd2.Cmd):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.register_precmd_hook(self.myhookmethod)

    def myhookmethod(self, data: cmd2.plugin.PrecommandData) -> cmd2.plugin.PrecommandData:
        # the statement object created from the user input
        # is available as data.statement
        return data
```

`cmd2.Cmd.register_precmd_hook` 会检查传入可调用对象的方法签名，如果参数数量错误会引发 `TypeError`。如果参数和返回值没有标注为 `PrecommandData`，也会引发 `TypeError`。

你可以选择通过创建具有不同属性的新 `cmd2.Statement` 来修改用户输入（见上文）。如果这样做，请将新的 `cmd2.Statement` 对象赋值给 `data.statement`。

命令前钩子必须返回一个 `cmd2.plugin.PrecommandData` 对象。你不需要从头创建此对象，可以直接返回传入钩子的那个。

在所有已注册的命令前钩子被调用之后，`cmd2.Cmd.precmd` 会被调用。为了保持与 `cmd.Cmd` 的完全向后兼容性，此方法接收的是 `cmd2.Statement`，而非 `cmd2.plugin.PrecommandData` 对象。

## 命令后钩子

一旦命令方法返回（即 `do_command(self, statement)` 方法已被调用并返回），所有命令后钩子都会被调用。如果用户重定向了输出，此时输出仍然被重定向，命令计时器也仍在运行。

定义和注册命令后钩子的方式如下：

```py
class App(cmd2.Cmd):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.register_postcmd_hook(self.myhookmethod)

    def myhookmethod(self, data: cmd2.plugin.PostcommandData) -> cmd2.plugin.PostcommandData:
        return data
```

你的钩子会接收一个 `cmd2.plugin.PostcommandData` 对象，它有一个 `statement` 属性，描述了已执行的命令。如果你的命令后钩子方法被调用，你可以确定命令方法已被调用，且它没有引发异常。

如果任何命令后钩子引发异常，异常将显示给用户，不会再调用更多的命令后钩子方法。命令终结钩子（如果有）仍会被调用。

在所有已注册的命令后钩子被调用之后，`self.postcmd` 会被调用以保持与 `cmd.Cmd` 的完全向后兼容性。

如果任何命令后钩子（已注册的或 `self.postcmd`）返回的 `cmd2.plugin.PostcommandData` 对象的 stop 属性被设为 `True`，后续的命令后钩子仍会被调用，命令终结钩子也会被调用，但一旦所有这些钩子都被调用完毕，应用将终止。同样，如果 `self.postcmd` 返回 `True`，命令终结钩子会在应用终止之前被调用。

任何命令后钩子都可以在返回前更改 `stop` 属性的值，修改后的值将传递给下一个命令后钩子。最后一个命令后钩子返回的值将传递给命令终结钩子，后者可以进一步修改该值。如果你的钩子盲目地返回 `False`，之前钩子请求退出应用的请求将不会被满足。除非有充分理由，否则最好返回传入的值。

要故意且静默地跳过命令后钩子，命令可以引发以下任一异常：

- `cmd2.exceptions.SkipPostcommandHooks`
- `cmd2.exceptions.Cmd2ArgparseError`

## 命令终结钩子

命令终结钩子即使在其他类型的钩子或命令方法引发异常时也会被调用。创建和注册命令终结钩子的方式如下：

```py
class App(cmd2.Cmd):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.register_cmdfinalization_hook(self.myhookmethod)

    def myhookmethod(self, data: cmd2.plugin.CommandFinalizationData) -> cmd2.plugin.CommandFinalizationData:
        return data
```

命令终结钩子必须检查传入的 `cmd2.plugin.CommandFinalizationData` 对象的 `statement` 属性是否包含值。在某些情况下，这些钩子可能在用户输入被解析之前就被调用，因此你不能总是依赖于有 `statement` 属性。

如果之前的解析后或命令前钩子已请求应用终止，传递给第一个命令终结钩子的 `stop` 属性值将为 `True`。任何命令终结钩子都可以在返回前更改 `stop` 属性的值，修改后的值将传递给下一个命令终结钩子。最后一个命令终结钩子返回的值将决定应用是否终止。

这种命令终结钩子的方式很强大，但也可能引起问题。如果你的钩子盲目地返回 `False`，之前钩子请求退出应用的请求将不会被满足。除非有充分理由，否则最好返回传入的值。

如果任何命令终结钩子引发异常，不会再调用更多的命令终结钩子。如果最后一个返回值的钩子返回了 `True`，则异常将被渲染，应用将终止。
