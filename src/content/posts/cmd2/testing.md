---
title: Python cmd2：测试指南
published: 2026-09-22
pinned: false
description: 介绍为 cmd2 应用编写单元测试和集成测试时的特殊注意事项，包括运行测试套件、测试命令以及 Mock 的正确用法。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 测试

## 概述

本文介绍为 cmd2 应用编写单元测试或集成测试时的特殊注意事项。

## 运行 cmd2 的测试套件

运行 `make test` 可执行测试套件，并附带覆盖率统计和 pytest-xdist 并行执行。默认的 `-n auto` 会根据可用 CPU 数自动选择工作进程数，并发运行相互独立的测试。同样的设置也适用于 `uv run pytest` 和 CI。

如果只想运行某个特定测试或进行交互式调试，可以禁用并行执行：

```sh
uv run pytest -n 0 tests/test_history.py
uv run pytest -n 0 --no-cov --pdb tests/test_history.py
```

也可以用 `-n 4` 之类的方式覆盖工作进程数。覆盖率会针对 `cmd2` 收集，并输出到终端、`coverage.xml` 和 `htmlcov/`。每次调用都会重新开始统计；如果希望合并多次运行的结果，请显式传入 `--cov-append`。

终端测试替身（test double）应当对光标位置查询作出应答，除非测试本身针对的就是缺失应答的情况。并发测试应使用事件同步，而不是任意的 sleep；当超时路径是被断言的对象时，应缩短仅用于测试的过期时间。对于依赖其他线程推进的等待，应保留足够宽裕的超时边界。

## 测试命令

我们鼓励 `cmd2` 应用开发者参考 [cmd2 tests](https://github.com/python-cmd2/cmd2/tree/main/tests) 中的示例，了解如何对 `cmd2` 命令进行单元测试和集成测试。其中提供了多种辅助工具，可以捕获并返回 stdout、stderr 以及命令特有的结果数据。

## Mock（模拟）

如果你需要在 cmd2 应用中 Mock 任何对象——尤其是在 [cmd2.Cmd][] 或 [cmd2.CommandSet][] 的子类中——必须使用 [Autospeccing](https://docs.python.org/3/library/unittest.mock.html#autospeccing)、[spec=True](https://docs.python.org/3/library/unittest.mock.html#patch)，或你所用 Mock 库提供的等价机制。

为了自动将函数加载为命令，`cmd2` 会执行大量反射调用来查找你在 cmd2 应用中定义的类的属性。许多 Mock 库会为请求的任何属性自动创建 Mock 对象，而不管该属性是否真实存在于被 Mock 的对象上。这种行为可能错误地引导 cmd2 把某个函数或属性当作它需要识别和处理的东西。为避免这种情况，你应该始终启用 [Autospeccing](https://docs.python.org/3/library/unittest.mock.html#autospeccing) 或 [spec=True](https://docs.python.org/3/library/unittest.mock.html#patch) 来进行 Mock。如果没有开启 autospeccing，你的单元测试会失败，并出现类似下面的错误信息：

```sh
cmd2.exceptions.CommandSetRegistrationError: Subcommand
<MagicMock name='cmdloop.subcommand_name' id='4506146416'> is not valid: must be a string.
Received <class 'unittest.mock.MagicMock'> instead
```

## 示例

```py
def test_mocked_methods():
    with mock.patch.object(MockMethodApp, 'foo', spec=True):
        cli = MockMethodApp()
```

另一个示例使用 [pytest-mock](https://pypi.org/project/pytest-mock) 提供的 `mocker` fixture：

```py
def test_mocked_methods2(mocker):
    mock_cmdloop = mocker.patch("cmd2.Cmd.cmdloop", autospec=True)
    cli = cmd2.Cmd()
    cli.cmdloop()
    assert mock_cmdloop.call_count == 1
```
