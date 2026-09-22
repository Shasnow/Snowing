---
title: Python cmd2：注解式参数处理
published: 2026-09-22
pinned: false
description: 介绍实验性的 @with_annotated 装饰器，如何用类型注解自动构建 argparse 解析器、注解到 argparse 的映射规则、元数据类与子命令。
tags: [Python, CLI, cmd2]
category: Python
licenseName: "Unlicensed"
draft: false
---

# 注解式参数处理

:::warning 实验性功能

`@with_annotated` 装饰器及其配套的 `Argument` / `Option` 元数据类是**实验性**的。其公开 API、可接受的类型注解范围以及生成的 argparse 行为都可能在未来的版本中发生变化，且不提供弃用周期。如果你依赖当前的确切语义，请锁定特定的 `cmd2` 版本，并在升级时重新审视你的用法。

对于需要稳定行为的生产代码，请改用 [@with_argparser](../argument_processing/#with_argparser-装饰器)。

:::

[@with_annotated][cmd2.with_annotated] 装饰器会根据被装饰函数的类型注解自动构建 argparse 解析器。无需手动调用 `add_argument()`，命令函数体直接接收带类型的关键字参数，而不是一个 `argparse.Namespace`。

两个装饰器可以互换使用——下面是同一个命令的两种写法：

**使用 `@with_annotated`：**

```py
@with_annotated
def do_greet(self, name: str, count: int = 1, loud: bool = False):
    for _ in range(count):
        msg = f"Hello {name}"
        self.poutput(msg.upper() if loud else msg)
```

**使用 `@with_argparser`：**

```py
parser = Cmd2ArgumentParser()
parser.add_argument("name", help="person to greet")
parser.add_argument("--count", type=int, default=1, help="repetitions")
parser.add_argument("--loud", action="store_true", help="shout")


@with_argparser(parser)
def do_greet(self, args):
    for _ in range(args.count):
        msg = f"Hello {args.name}"
        self.poutput(msg.upper() if args.loud else msg)
```

注解版本更简洁、能给你带类型的参数，并且直接支持若干高级 cmd2 特性，包括 `ns_provider`、`with_unknown_args` 和类型化子命令。当你需要一个稳定、成熟的 API 或需要对解析器进行细粒度控制时，选择 `@with_argparser`；当你想要类型注解驱动的简洁体验、并且能接受其试验性状态时，选择 `@with_annotated`。

## 基本用法

没有默认值的参数成为位置参数。有默认值的参数成为 `--option` 标志。仅限关键字的参数（在 `*` 之后）始终成为选项，若没有默认值则成为必填选项。

参数名中的下划线会在生成的标志中转换为短横线，因此 `dry_run` 会变成 `--dry-run`。你在函数体中读取的 Python 标识符仍保留下划线形式（`args.dry_run`）。若要关闭这一转换，可通过 `Option("--my_flag", ...)` 传入显式名称。

```py
from cmd2.annotated import with_annotated


class MyApp(cmd2.Cmd):
    @with_annotated
    def do_greet(self, name: str, count: int = 1, loud: bool = False):
        """Greet someone."""
        for _ in range(count):
            msg = f"Hello {name}"
            self.poutput(msg.upper() if loud else msg)
```

命令 `greet Alice --count 3 --loud` 会解析出 `name="Alice"`、`count=3`、`loud=True`，并把它们作为关键字参数传入。

## 注解如何映射到 argparse

该装饰器把 Python 类型注解转换为 `add_argument()` 调用：

| 类型注解 | 生成的 argparse 设置 |
| --- | --- |
| `str` | 默认（无需 `type=`） |
| `int`、`float` | `type=int` 或 `type=float` |
| 带默认值的 `bool` | 通过 `BooleanOptionalAction` 生成布尔可选标志 |
| 位置参数 `bool` | 从 `true/false`、`yes/no`、`on/off`、`1/0` 解析 |
| `Path` | `type=Path` |
| `Enum` 子类 | `type=converter`，`choices` 来自成员值 |
| `EnumA \| EnumB`（所有成员均为 `Enum`） | 逐个尝试，第一个接受该词法单元的成员胜出；`choices` 合并 |
| `decimal.Decimal` | `type=decimal.Decimal` |
| `Literal[...]` | `type=literal-converter`，`choices` 来自各值 |
| `list[T]` / `set[T]` / `frozenset[T]` / `tuple[T, ...]` | `nargs='+'`（若有默认值或是 `\| None` 则为 `'*'`） |
| `tuple[T, T]` | 固定 `nargs=N` 并使用 `type=T` |
| `T \| None`（无默认值） | 位置参数 `nargs='?'`（接受 0 或 1 个词法单元） |
| `T \| None = None` | `--flag` 选项，`default=None` |

当集合类型与 `@with_annotated` 一起使用时，解析出的值按以下类型传给命令函数：

- `list[T]` 传为 `list`
- `set[T]` 传为 `set`
- `frozenset[T]` 传为 `frozenset`
- `tuple[T, ...]` 传为 `tuple`

不支持的模式会抛出 `TypeError`，包括：

- 含有多个非 `None` 成员的联合类型，如 `str | int`（除非每个成员都是 `Enum` 子类，例如 `EnumA | EnumB`，此时按顺序尝试每个成员的转换器）
- 混合类型的元组，如 `tuple[int, str]`
- `Annotated[T, meta] | None`；应改写为 `Annotated[T | None, meta]`
- `Annotated[T, Argument(nargs=N)]`，其中 `N` 为 `'*'`、`'+'` 或 `>= 1` 的整数，而 `T` 不是集合类型。会产生值列表的 `nargs` 需要集合注解，如 `list[T]` 或 `tuple[T, ...]`
- 可选的固定元数位置参数，如 `Annotated[tuple[int, int], Argument()] = (1, 2)`、`Annotated[tuple[int, int] | None, Argument()]`，或任何带默认值或 `| None` 的位置参数 `Argument(nargs=N)`。argparse 无法让固定元数的位置参数变为可选（没有表示"缺席或恰好 N 个词法单元"的 `nargs`），因此请改用可变元数类型如 `tuple[T, ...]`、去掉默认值，或把它改成选项（给出默认值但不带 `Argument()`）
- `Annotated[tuple[T, T], Argument(nargs=N)]`，其中 `N` 与元组声明的元素数量不一致。元组类型已经固定了 `nargs`，用户元数据无法更改它

参数名 `dest` 和 `subcommand` 是保留字，不能用作注解参数名。`cmd2_statement` 接收解析后的 [cmd2.Statement][] 对象，`cmd2_subcommand_func`（仅在用 `@with_annotated(base_command=True)` 装饰的命令上有效）接收子命令处理函数。

## 注解元数据

若需更精细的控制，可使用 `typing.Annotated` 搭配 [Argument][cmd2.annotated.Argument] 或 [Option][cmd2.annotated.Option] 元数据：

```py
from typing import Annotated
from cmd2.annotated import Argument, Option, with_annotated


class MyApp(cmd2.Cmd):
    def sport_choices(self) -> cmd2.Choices:
        return cmd2.Choices.from_values(["football", "basketball"])

    @with_annotated
    def do_play(
        self,
        sport: Annotated[
            str,
            Argument(
                choices_provider=sport_choices,
                help_text="Sport to play",
            ),
        ],
        venue: Annotated[
            str,
            Option(
                "--venue",
                "-v",
                help_text="Where to play",
                completer=cmd2.Cmd.path_complete,
            ),
        ] = "home",
    ):
        self.poutput(f"Playing {sport} at {venue}")
```

`Argument` 和 `Option` 都接受与 `add_argument()` 相同的 cmd2 专属字段：`choices`、`choices_provider`、`completer`、`table_columns`、`suppress_tab_hint`、`metavar`、`nargs` 和 `help_text`。它们还接受用于[自定义转换](#自定义转换converter-与-preprocess)的 `converter` / `preprocess`，以及用于[枚举别名](#枚举别名与特殊关键字-allow_unknown_entry)的 `allow_unknown_entry`。

`Option` 额外接受 `action`、`required`，以及用于自定义标志字符串的位置 `*names`（例如 `Option("--color", "-c")`）。

### 枚举别名与特殊关键字（`allow_unknown_entry`）

默认情况下，`Enum` 参数只接受成员值和成员名。若要让枚举还接受别名、替代拼写或特殊关键字，可在枚举上定义标准的 Python [`_missing_`](https://docs.python.org/3/library/enum.html#enum.Enum._missing_) 钩子，并用 `allow_unknown_entry=True` 显式启用：

```py
import enum
from typing import Annotated
from cmd2.annotated import Argument, with_annotated


class Color(enum.Enum):
    red = "red"
    green = "green"
    blue = "blue"

    @classmethod
    def _missing_(cls, value):
        # 把特殊关键字映射到真实成员；返回 None 则拒绝
        return cls.red if str(value).lower() == "auto" else None


class MyApp(cmd2.Cmd):
    @with_annotated
    def do_theme(self, choice: Annotated[Color, Argument(allow_unknown_entry=True)]) -> None:
        self.poutput(f"theme set to {choice.value}")
```

现在 `theme auto` 会通过 `_missing_` 解析为 `Color.red`，而 `theme red`（值）和 `theme green`（名）仍照常工作。若 `_missing_` 拒绝某个词法单元（返回 `None`），仍会抛出通常的"choose from ..."错误。`_missing_` 自身抛出的任何异常都会原样传播（不会被掩盖成"invalid choice"）。该标志对非 `Enum` 注解无效；未重写 `_missing_` 的 `Enum` 继承默认实现（返回 `None`），因此该标志对其没有作用。它同样适用于枚举作为集合元素的场景（如 `Annotated[list[Color], Argument(allow_unknown_entry=True)]`）。

由于 `_missing_` 别名是动态的，它们不会加入对外公布的 `choices`，因此不会出现在 `--help` 或 Tab 补全中；规范的成员值仍是列出的选项集。

### 枚举的联合类型

标注为联合类型、且所有成员都是 `Enum` 子类的参数（例如 `EnumA | EnumB`）是被接受的。每个成员保留自己的转换器，词法单元按**第一个接受它的成员**解析：

```py
@with_annotated
def do_pick(self, choice: Suit | Rank) -> None:
    if isinstance(choice, Suit):
        self.poutput(f"suit {choice.name}")
    else:
        self.poutput(f"rank {choice.name}")
```

由于解析是"先到先得"，**顺序很重要**：如果一个词法单元是多个成员的合法值（或名称），联合类型中列在前面的成员胜出，后面成员的相同词法单元将变得不可达。`allow_unknown_entry` 和各成员的 `_missing_` 钩子仍按成员生效；若某成员的 `_missing_` 对某个词法单元是*抛出异常*（而非返回 `None`），它只是拒绝该词法单元，下一个成员仍会被尝试，且该异常不会中止整个联合类型。只有当所有成员都拒绝时，才会抛出通常的"choose from ..."错误。仅支持 `Enum` 成员；包含 `Literal` 或任何非 `Enum` 类型的联合类型仍会因歧义而被拒绝。

当两个值集有重叠时，请优先选用[类型化子命令](#注解式子命令)（每个子命令一个 `Enum`），让选择变得明确且无冲突。

### 动作（action）

当 `Option(action=...)` 使用不从命令行取值的零参数 argparse 动作（`count`、`store_true`、`store_false`、`store_const`、`append_const`）时，`@with_annotated` 会在调用 `add_argument()` 之前剥离它从类型推断出的面向取值的元数据：

- `type` 转换器
- 静态 `choices`
- 任何推断出的 Tab 补全器（例如 `Path` 的路径补全器）或 `choices_provider`

这与 argparse 的行为一致（它会拒绝给无值动作配备补全器），并避免诸如把 `action='count'` 与 `type=int` 组合这样的解析器构建错误。*确实*会取值的动作（在 `list[T]` 上的 `append` / `extend`，或普通取值选项）会保留推断出的转换器和补全器。

在标量 `Option` 上把 `const` 与显式 `nargs` 搭配，会选用 argparse 的可选取值惯用法而非 `store_const`。`Annotated[str | None, Option("--log", nargs='?', const="CONSOLE")]` 保留 `store` 动作和推断出的 `type` 转换器，因此该标志是三态的：

- 缺席时得到默认值
- 只写 `--log` 时得到 `const`
- `--log VALUE` 时得到转换后的 `VALUE`

`const` 按原样存储（不会经过转换器），因此它必须已经符合声明的类型。若没有显式 `nargs`，仅写 `const` 仍会推断为无值的 `store_const`（出现即得到 `const`，提供取值则是错误）。

`Option(action=...)` 还接受自定义的 `argparse.Action` 子类。该类会直接传给 `add_argument()` 并自行负责解析值的存储，因此类型推断的集合转换以及动作专属的类型/const/形状约束都会被跳过；推断出的 `type=` 转换器、默认值和 `required` 仍会应用，该类会像任何手写的 `add_argument()` 调用一样接收到它们。

```py
class UpperAction(argparse.Action):
    def __call__(self, parser, namespace, values, option_string=None):
        setattr(namespace, self.dest, values.upper())


@with_annotated
def do_shout(self, name: Annotated[str, Option("--name", action=UpperAction)] = ""):
    self.poutput(name)
```

`@with_annotated` 不支持 `action='help'` 和 `action='version'`；如需使用请改用 `@with_argparser`。

### 保留的关键字参数

`Argument()` 和 `Option()` 会拒绝一小部分 `add_argument()` 的关键字参数——这些参数由装饰器从函数签名本身推导而来，这样误用它们会表现为清晰的 `TypeError`，而不是被静默覆盖。被拒绝的关键字参数包括：

- `type` —— 来自参数注解；若需自定义的字符串到值的可调用对象，请使用 [`converter`](#自定义转换converter-与-preprocess)（或用 `preprocess` 先转换词法单元）
- `dest` —— 来自参数名
- `Argument` 上的 `action` 和 `required` —— 只有 `Option` 接受它们；位置参数没有 action，且除非带默认值或 `| None`，否则总是必填

其他所有 `add_argument()` 参数都会透传，包括通过 [register_argparse_argument_parameter][cmd2.argparse_utils.register_argparse_argument_parameter] 注册的任何自定义参数。

### 默认值

`default` 既可以通过函数签名提供，也可以作为元数据关键字参数提供。两种形式等价：

```py
# 签名默认值
def do_x(self, name: Annotated[str, Option("--name")] = "HI"): ...


# 元数据默认值（行为相同）
def do_x(self, name: Annotated[str, Option("--name", default="HI")]): ...
```

同时指定两者属于冲突，会抛出 `TypeError`。任何来源的 `argparse.SUPPRESS` 默认值都会被拒绝，因为抑制 namespace 属性会导致调用函数时缺少其期望的关键字参数。

`add_help`、`prefix_chars`、`fromfile_prefix_chars`、`argument_default`、`conflict_handler` 和 `allow_abbrev` 等解析器构建参数不会由 `@with_annotated` 暴露。请在自定义的 `parser_class` 子类上设置它们，再通过 `parser_class=` 传入。

### choices 与枚举

当为推断出的 `Enum` 或 `Literal` 提供了用户自定义的 `choices_provider` 或 `completer` 时，推断出的静态 `choices` 列表会被丢弃，改由 provider 或 completer 驱动补全。推断出的 `type` 转换器会保留，因此解析值仍会强制转换为声明的类型（`Literal[1, 2]` 得到 `int`，`Enum` 得到其成员），类型之外的值会在解析时被拒绝。

显式的 `choices=` 会与推断出的类型相协调，而不是与之冲突：

- 各值会先经过推断出的 `type` 转换器，以匹配 argparse 转换后的比较逻辑。`Annotated[int, Option("--n", choices=["1", "2"])]` 会被规范化为 `choices=[1, 2]`，因此 `--n 1` 被接受。若某个 choice 被转换器拒绝（`int` 上的 `choices=["1", "nope"]`），会在构建时抛出 `TypeError`。已经是声明类型的值保持原样。
- 显式 `choices=` 优先于*类型推断出的*补全器（例如 `Path` 补全器）：choices 会被保留（用于校验和驱动补全），推断出的补全器被丢弃。你自己传入的 `choices_provider`/`completer` 仍优先于 `choices=`。

`Enum` 参数在命令行上同时接受成员**值**和成员**名**（值为 `"red"` 的 `Color.RED` 可以用 `red` 或 `RED` 选中）；Tab 补全和 `--help` 列出的是值。

### 自定义转换（`converter` 与 `preprocess`）

使用 `@with_argparser` 时，你可以把任意可调用对象作为 `add_argument(type=...)` 来把词法单元解析成自定义值。`@with_annotated` 则从注解推导 `type=`，并拒绝元数据中的裸 `type=`（它会静默遮蔽推断出的转换器）。有两个钩子可以提供同等能力而没有这个坑：

- `converter` —— 一个 `Callable[[str], Any]`，**替换**推断出的 `type=` 转换器。
- `preprocess` —— 一个 `Callable[[str], str]`，在推断出的转换器**之前**运行。

二者区别在于保留了什么。`converter` 主管整个转换，因此推断出的 `choices` 和补全器（它们描述的是推断出的值空间）会被丢弃。`preprocess` 只转换原始词法单元，因此推断出的 `type=`、`choices`、补全器和强制转换全部保留。

#### `converter`：替换转换

当注解内置的转换不适合你的输入，或该类型根本没有内置转换时，使用 `converter`。由于转换器主管转换，注解不再必须是受支持的标量类型之一——任何类型都合法，通常的"不支持的类型"错误也不会出现：

```py
import datetime
from typing import Annotated
from cmd2.annotated import Argument, Option, with_annotated


def parse_size(value: str) -> int:
    """解析带可选 K/M/G 后缀的整数。"""
    multiplier = {"K": 1_000, "M": 1_000_000, "G": 1_000_000_000}.get(value[-1:].upper(), 1)
    return int(value[:-1] if multiplier != 1 else value) * multiplier


class MyApp(cmd2.Cmd):
    @with_annotated
    def do_alloc(self, size: Annotated[int, Argument(converter=parse_size)]) -> None:
        self.poutput(f"Allocating {size} bytes")  # `alloc 64K` -> 64000

    @with_annotated
    def do_at(self, when: Annotated[datetime.datetime, Option("--when", converter=datetime.datetime.fromisoformat)]):
        self.poutput(when.isoformat())  # `datetime` 没有推断出的转换器，converter= 使其合法
```

argparse 按词法单元逐一应用 `type=`，因此在 `list[T]` 上转换器会对每个值各运行一次。若要接收**单个**词法单元并让转换器返回整个集合（一到多的惯用法），请标注为非集合类型如 `Any`——像 `set[int]` 这样的集合注解会改为推断 `nargs` 并把输入拆分成多个词法单元：

```py
from typing import Annotated, Any


def parse_intset(value: str) -> set[int]:
    return {int(piece) for piece in value.split(",")}


@with_annotated
def do_select(self, idx: Annotated[Any, Option("--idx", converter=parse_intset)]) -> None:
    self.poutput(sorted(idx))  # `select --idx 1,3,5` -> [1, 3, 5]
```

显式的 `choices=` 仍会与转换器相协调（其各值会经过转换器），因此你可以在替换转换后重新加上受限的选项集。

#### `preprocess`：在推断的转换之前做规范化

当推断的转换是正确的、只是原始词法单元需要先加工时，使用 `preprocess`。推断出的 `type=`、`choices` 和补全器会保留，因此 `Enum` 在接受规范化输入的同时保留其选项和补全，`Path` 保留其补全器：

```py
import os
from typing import Annotated
from cmd2.annotated import Argument, with_annotated


class MyApp(cmd2.Cmd):
    @with_annotated
    def do_tag(self, color: Annotated[Color, Argument(preprocess=str.lower)]) -> None:
        self.poutput(color.value)  # `tag RED` 可用，`tag <TAB>` 仍会列出颜色

    @with_annotated
    def do_open(self, path: Annotated[Path, Argument(preprocess=os.path.expanduser)]) -> None:
        self.poutput(path)  # `open ~/file` 会展开，路径补全仍可用
```

对于普通的 `str`（没有推断出的转换器），`preprocess` 可调用对象会直接成为 `type=`。

`converter` 和 `preprocess` 在同一参数上**互斥**——转换器已经接收原始词法单元，请把预处理并入其中。二者都不能与无取值动作（`store_true`、`store_false`、`count`、`store_const`、`append_const`）组合，因为后者不消费任何需要转换的词法单元。违反者都会在装饰时抛出 `TypeError`。

## 装饰器选项

`@with_annotated` 目前支持：

- `ns_provider` —— 在解析前预填充命名空间，与 `@with_argparser` 对应
- `preserve_quotes` —— 为 `True` 时保留参数中的引号
- `with_unknown_args` —— 为 `True` 时，无法识别的参数通过 `_unknown` 传入
- `subcommand_to` —— 把该函数注册为某个父命令下的注解式子命令
- `base_command` —— 创建基础命令，其解析器同时添加子解析器并暴露 `cmd2_subcommand_func`。`cmd2_subcommand_func` 参数只在用 `base_command=True` 装饰的命令上有效；在别处声明会抛出 `TypeError`
- `subcommand_required` —— 是否必须提供子命令（仅在 `base_command=True` 时有效，默认 `True`）
- `subcommand_metavar` —— 子命令组显示的 metavar（仅在 `base_command=True` 时有效，默认 `"SUBCOMMAND"`）
- `subcommand_title` —— 子命令 `--help` 小节的标题（仅在 `base_command=True` 时有效）。设置它或 `subcommand_description` 会把子命令从位置参数小节移到独立小节，与 argparse 中的行为一致
- `subcommand_description` —— 子命令 `--help` 小节的描述（仅在 `base_command=True` 时有效）。只提供它而不提供 `subcommand_title` 时，该小节使用 argparse 的默认标题 `subcommands`
- `help` —— 注解式子命令的帮助文本（仅在配合 `subcommand_to` 时有效）
- `aliases` —— 注解式子命令的别名（仅在配合 `subcommand_to` 时有效）
- `deprecated` —— 在 `--help` 中把该子命令标记为已弃用（仅在配合 `subcommand_to` 时有效）
- `groups` —— 把参数名分配给参数组的 `Group` 实例
- `mutually_exclusive_groups` —— 互斥参数的 `Group` 实例
- `parser_class` —— 自定义解析器类（默认为配置的默认值）
- `**parser_kwargs` —— 其他所有 `Cmd2ArgumentParser` 构造关键字参数，通过 PEP 692 [`Unpack[Cmd2ParserKwargs]`][cmd2.annotated.Cmd2ParserKwargs] 转发。完整列表及 `description` / `prog` 的特例见下文[解析器定制](#解析器定制)

```py
@with_annotated(with_unknown_args=True)
def do_rawish(self, name: str, _unknown: list[str] | None = None):
    self.poutput((name, _unknown))
```

## 解析器定制

每个 `Cmd2ArgumentParser` 构造关键字参数都通过 PEP 692 [`Unpack[Cmd2ParserKwargs]`][cmd2.annotated.Cmd2ParserKwargs] 直接流经 `@with_annotated` 和 `build_parser_from_function`。[`Cmd2ParserKwargs`][cmd2.annotated.Cmd2ParserKwargs] 这个 `TypedDict` 是被转发关键字参数的唯一权威来源，并为类型检查器/IDE 在装饰器调用点提供自动补全：给 `Cmd2ArgumentParser` 新增构造关键字参数只需在 `Cmd2ParserKwargs` 上添加对应字段，注解式装饰器就会自动识别。

被转发的关键字参数包括 `description`、`epilog`、`prog`、`usage`、`parents`、`argument_default`、`prefix_chars`、`fromfile_prefix_chars`、`conflict_handler`、`add_help`、`allow_abbrev`、`exit_on_error`、`formatter_class`、`completer_class`，以及 Python ≥ 3.14 上的 `suggest_on_error` / `color`。其中两个在裸透传之上叠加了额外行为：

- `description` —— 省略时会用函数的 docstring 填充（详见下文）；传入显式值可覆盖
- `prog` —— 设置了 `subcommand_to` 时会被拒绝；cmd2 的子命令机制会根据父命令层级重写 `prog`，这里的任何值都会被静默覆盖

`parser_class` 保持为独立的显式关键字参数，因为它选择的是类本身，而不是传给它的值。参数组通过 [Group][cmd2.annotated.Group] 声明；传入 `title` 和 `description` 可得到带标题的帮助小节（省略则为无标题分组）：

```py
from cmd2.annotated import Group, with_annotated


class App(cmd2.Cmd):
    @with_annotated(
        description="Open a network connection.",
        epilog="Example: connect example.com --port 2222",
        groups=(Group("host", "port", title="connection", description="where to connect"),),
    )
    def do_connect(self, host: str, port: int = 22, verbose: bool = False):
        self.poutput(f"connecting to {host}:{port}")
```

若省略 `description`，函数 docstring 的第一段（到第一个空行为止的所有内容）会被用作解析器描述；后续段落会被丢弃，以免 rst 字段指令（如 `:param name:`）泄漏进 `--help`。传 `description=""` 可关闭自动填充，传 `description="..."` 可覆盖它。

```py
@with_annotated
def do_greet(self, name: str):
    """Greet someone by name.

    :param name: who to greet
    """
    self.poutput(f"hello {name}")


# parser.description == "Greet someone by name."
```

`mutually_exclusive_groups` 同样接受 `Group` 实例。传 `Group(..., required=True)` 可让互斥组本身成为必填——argparse 会强制要求其成员中恰好有一个被提供。普通（非互斥）`Group` 上的 `required=True` 会被拒绝，因为 `add_argument_group` 没有 `required` 标志。

```py
@with_annotated(
    mutually_exclusive_groups=(Group("verbose", "quiet", required=True),),
)
def do_run(self, verbose: bool = False, quiet: bool = False): ...
```

给 `mutually_exclusive_groups` 的 `Group` 加上 `title`/`description`，可以把它渲染为带标题的帮助小节——这是 argparse 唯一支持的嵌套形式：互斥组*位于*参数组内部。你只需声明一次，无需配对的 `groups=` 条目。对每个 `bool` 成员使用 `Option(action="store_true")`，可让选项显示为 `[--json | --csv]`，而不是展开成 `--json`/`--no-json` 和 `--csv`/`--no-csv`：

```py
@with_annotated(
    mutually_exclusive_groups=(Group("json", "csv", title="output", description="how to write results"),),
)
def do_render(
    self,
    json: Annotated[bool, Option(action="store_true")] = False,
    csv: Annotated[bool, Option(action="store_true")] = False,
): ...
```

若要把非互斥参数放进同一小节，请用包含它们全部的 `groups=` 条目声明，并且不要给互斥组加标题；argparse 会把互斥组嵌套在该组内。在两处都声明该小节、互斥组只有部分成员出现在 `groups=` 条目中、或它横跨两个条目，都会抛出 `ValueError`。另外三种嵌套形式（参数组嵌套在另一个组或互斥组内、互斥组嵌套在互斥组内）在 Python 3.14 的 argparse 中已移除，此处也无法表达。

`Group` 的成员命名的是命令行**参数**。对于普通参数，参数即参数本身，因此名称两边相同。`ArgumentBlock` 参数则不同：它会展开为每个字段一个参数（字段名 == 参数名），因此你要命名的是它的字段而非整个块：

```python
@dataclass
class Conn(ArgumentBlock):
    host: Annotated[str, Option("--host")] = "localhost"
    port: Annotated[int, Option("--port")] = 8080


@with_annotated(groups=(Group("host", "port", title="connection"),))
def do_connect(self, conn: Conn) -> None: ...
```

给块参数本身命名（`Group("conn")`）会抛出 `ValueError`：该参数已被展开，没有属于自己的参数。逐个命名块的字段还有一个好处：你可以指明哪些字段是互斥选项——把 `Group("json", "csv")` 放进 `mutually_exclusive_groups`，就只有这两个互相排斥，而不是它们所在块的每个字段都互相排斥。

分组规格的*形状*规则——成员重复出现、一个成员被分到两个组、普通组上的 `required=True`，以及上文的互斥嵌套规则——都在装饰器运行时校验，因此形状不合法的分组会在类定义时抛出 `ValueError`，而不是等到第一次使用命令时。这些检查只读取 `Group` 规格，不读取签名或类型注解，因此前向引用的注解仍能正常装饰。

依赖成员*身份*的规则（例如每个成员都指向真实参数、互斥组成员必须可省略——即有默认值或是 `T | None`）在构建解析器时才触发，因为不解析类型注解就无法知道块的字段名。

`parents=` 与 argparse 标准的 parents 机制一致，用于在多个解析器间共享参数定义。`argument_default=argparse.SUPPRESS` 不受支持并抛出 `TypeError`。它会把缺席的参数从解析出的命名空间中移除，但 `@with_annotated` 是按函数签名构建调用的，因此每个声明的参数在调用时都必须存在；参数从命名空间中消失在这里永远不可能合法（与逐参数的 `default=argparse.SUPPRESS` 被拒绝相对应）。其他任何 `argument_default` 值都会原样转发给解析器。

其余 argparse 关键字参数覆盖较少见的需求，但都原样接入：

- `prefix_chars="+-"` 接受以 `+` 开头的选项（如 `+verbose`）；配合显式的 `Option("+verbose")` 来声明此类标志
- `fromfile_prefix_chars="@"` 让用户写 `mycmd @args.txt`，把文件内容作为参数拼接进来
- `conflict_handler="resolve"` 让父解析器的选项可以在本地重新定义而不报错——配合 `parents=` 覆盖继承的标志时很有用
- `add_help=False` 移除自动添加的 `-h`/`--help` 动作（cmd2 的标准解析器默认保留它）
- `allow_abbrev=False` 要求用户输入完整的长选项名（`--verbose` 不能写成 `--verb`）
- `exit_on_error=False` 让解析失败抛出 `argparse.ArgumentError` 而不是调用 `sys.exit`——在把解析器嵌入其他流程时很有用

## 注解式子命令

`@with_annotated` 还能无需手动构造子解析器就构建类型化的子命令树。

```py
@with_annotated(base_command=True)
def do_manage(self, *, cmd2_subcommand_func):
    if cmd2_subcommand_func:
        cmd2_subcommand_func()


@with_annotated(subcommand_to="manage", help="list projects")
def manage_list(self):
    self.poutput("listing")
```

对于嵌套子命令，`subcommand_to` 可以是空格分隔的，例如 `subcommand_to="manage project"`。中间层级也必须声明为创建自己的子解析器的子命令：

```py
@with_annotated(subcommand_to="manage", base_command=True, help="manage projects")
def manage_project(self, *, cmd2_subcommand_func):
    if cmd2_subcommand_func:
        cmd2_subcommand_func()


@with_annotated(subcommand_to="manage project", help="add a project")
def manage_project_add(self, name: str):
    self.poutput(f"added {name}")
```

## 参数块

当多个命令共享同一组参数时，可复用的*参数块*（argument block）能消除重复。在 `@dataclass` 上继承 `cmd2.ArgumentBlock` 特性类，并用它注解某个参数。每个字段成为一个扁平的命令行参数（字段名 == 参数名），解析出的值会被重建成 dataclass 实例传给命令：

```py
from dataclasses import dataclass
from typing import Annotated
from pathlib import Path

import cmd2
from cmd2 import with_annotated
from cmd2.annotated import Option


@dataclass
class CommonArgs(cmd2.ArgumentBlock):
    verbose: Annotated[bool, Option("-v", "--verbose")] = False
    output: Annotated[Path | None, Option("--output")] = None


class App(cmd2.Cmd):
    @with_annotated
    def do_build(self, target: str, common: CommonArgs):
        self.poutput(f"{target} verbose={common.verbose} output={common.output}")
```

`build app --verbose --output /tmp/x` 会重建出 `CommonArgs(verbose=True, output=Path("/tmp/x"))` 并作为 `common` 传入。块参数本身永远不是参数，只有它的字段才是。

字段携带常规的 `Annotated[T, Option(...)]` / `Annotated[T, Argument(...)]` 元数据，其行为与同形状的顶层参数完全一致。dataclass 是默认值的唯一权威来源：带默认值（`default` 或 `default_factory`）的字段在调用时由 dataclass 构造器填充，因此 `default_factory` 每次调用都会产生新值，`__post_init__` 也会运行。没有默认值的字段成为必填参数。

继承是复用机制：块的子类本身也是块，因此可以在共享基块的基础上按命令扩展，而无需重复其字段。

```py
@dataclass
class TracedArgs(CommonArgs):
    trace: bool = False  # do_test 会得到 verbose、output 和 trace
```

几条规则保证块的语义无歧义：

- 触发条件是 `ArgumentBlock` 特性类，而非"是 dataclass"。普通 `@dataclass` 不受影响，仍可作为普通的单值使用（例如通过 `Argument(converter=...)`）
- 块必须是普通参数的*裸*注解。把它包进 `Annotated`/`Optional`/联合类型，或用作 `*args`/`**kwargs`，都会抛出清晰的错误
- 由于字段是扁平展开的，字段名与其他参数或其他块的字段冲突时，会在构建解析器时报错，而不是静默共享同一个目标名
- 与块参数同名的字段（例如接收为 `opts: Opts` 的块上的 `opts`）会被拒绝：字段的扁平参数与接收块实例的参数会共享同一个目标名
- 字段的默认值必须写在 dataclass 字段上（`= value` 或 `= field(default_factory=...)`），不能写在 `Option`/`Argument` 元数据中，也不能通过动作默认值（`append`/`count`）提供。默认值由 dataclass 掌管以保证每次调用得到新值，因此在没有 dataclass 默认值的字段上提供元数据或动作默认值会被拒绝
- 类型本身是块的字段不会被展开（不递归），会作为不支持的类型被拒绝

### 与子命令共享块（`cmd2_base_args` / `cmd2_parent_args`）

基础命令及其子命令解析到同一个共享命名空间。若要沿命令链向下共享某个块，请在拥有这些标志的命令上把参数命名为 `cmd2_base_args`，在每个应接收它的子命令上命名为 `cmd2_parent_args`，两处都用同一个块类型注解：

```py
@dataclass
class SharedOpts(cmd2.ArgumentBlock):
    verbose: Annotated[bool, Option("-v", "--verbose")] = False
    level: Annotated[int, Option("--level")] = 1


class App(cmd2.Cmd):
    @with_annotated(base_command=True)
    def do_root(self, cmd2_subcommand_func, cmd2_base_args: SharedOpts):
        if cmd2_subcommand_func:
            cmd2_subcommand_func()

    @with_annotated(subcommand_to="root", help="show the inherited block")
    def root_show(self, cmd2_parent_args: SharedOpts):
        self.poutput(f"verbose={cmd2_parent_args.verbose} level={cmd2_parent_args.level}")
```

`root --verbose --level 5 show` 输出 `verbose=True level=5`：在 `root` 上解析的选项以类型化方式流入子命令，无需重新声明。`cmd2_base_args` 把块的标志添加到自己命令的解析器上，而 `cmd2_parent_args` *不*添加任何参数，它由祖先解析出的值重建（`root --verbose show`，而不是 `root show --verbose`）。若某子命令使用了 `cmd2_parent_args`，但其祖先从未声明匹配的 `cmd2_base_args`，它第一次运行时会抛出清晰的错误。这是通过 `ns_provider` 转发父级状态的类型化替代方案。

每个共享块字段都解析到一个由其块类型限定的目标名，因此同一条命令链上不同层级的两个不同类型的 `cmd2_base_args` 块可以使用相同的字段名而不冲突。继承到的块总是接收自己类型的值，无论中间层级上另一个块的同名字段解析出了什么。在两个层级复用同一块类型会共享一个目标名，因此设置该标志的最近层级胜出。

子命令也可以在继承的块之外声明自己的普通块。二者相互独立：继承块的标志在父命令上提供，而子命令自己的块把它的标志添加到子命令的解析器上。

```py
@dataclass
class RunOpts(cmd2.ArgumentBlock):
    retries: Annotated[int, Option("--retries")] = 0


class App(cmd2.Cmd):
    @with_annotated(base_command=True)
    def do_root(self, cmd2_subcommand_func, cmd2_base_args: SharedOpts):
        if cmd2_subcommand_func:
            cmd2_subcommand_func()

    @with_annotated(subcommand_to="root")
    def root_run(self, name: str, cmd2_parent_args: SharedOpts, run: RunOpts):
        # --verbose/--level 来自 `root`；--retries 是本子命令自己的标志
        self.poutput(f"run {name} verbose={cmd2_parent_args.verbose} retries={run.retries}")
```

`root --verbose run job --retries 3` 会在 `root` 上解析 `--verbose`、在 `run` 上解析 `--retries`。

## 底层解析器构建

[cmd2.annotated.build_parser_from_function][cmd2.annotated.build_parser_from_function] 可以直接从函数构建解析器，而无需注册命令。它接受与 `@with_annotated` 相同的 `groups`、`mutually_exclusive_groups`、`parser_class` 和转发的 [`Unpack[Cmd2ParserKwargs]`][cmd2.annotated.Cmd2ParserKwargs]。与装饰器一样，它会跳过第一个参数作为方法接收者（`self`/`cls`）。

```py
from cmd2.annotated import build_parser_from_function


def greet(self, name: str, count: int = 1):
    """Greet someone."""


parser = build_parser_from_function(greet)
namespace = parser.parse_args(["Alice", "--count", "3"])
# namespace.name == "Alice", namespace.count == 3
```

## 基于类型的自动补全

使用 `@with_annotated` 时，标注为 `Path` 或 `Enum` 的参数无需显式的 `choices_provider` 或 `completer` 即可获得自动补全。

具体来说：

- `Path`（或任何 `Path` 子类）触发文件系统路径补全
- `MyEnum`（任何 `enum.Enum` 子类）触发基于枚举成员值的补全

使用 `@with_argparser` 时，若需要补全行为，请显式提供 `choices`、`choices_provider` 或 `completer`。

## 稳定性与反馈

由于该功能是实验性的：

- 边缘情况的行为（混合类型元组、深层嵌套的 `Annotated`、冲突的元数据）可能发生变化
- 诊断性错误消息的措辞可能调整
- 支持的类型注解集合可能扩充或缩减

如果你依赖 `@with_annotated`，请通过 [issue tracker](https://github.com/python-cmd2/cmd2/issues) 分享反馈和边缘案例，以便在该功能结束实验性状态之前把行为固定下来。

# API 参考：cmd2.annotated

官方文档中的 API 参考页由 mkdocstrings 从 `cmd2.annotated` 模块的 docstring 自动生成（源文件仅包含 `::: cmd2.annotated` 指令）。其主要公开成员如下：

- [cmd2.annotated.with_annotated][cmd2.with_annotated] —— 根据类型注解构建解析器的命令装饰器
- [cmd2.annotated.Argument][] —— 位置参数的 `typing.Annotated` 元数据类
- [cmd2.annotated.Option][] —— 选项参数的 `typing.Annotated` 元数据类
- [cmd2.annotated.Group][] —— 用于参数组与互斥组的分组规格
- [cmd2.annotated.ArgumentBlock][] —— 参数块所继承的 dataclass 特性类
- [cmd2.annotated.build_parser_from_function][] —— 不注册命令、直接从函数构建解析器
- [cmd2.annotated.Cmd2ParserKwargs][] —— 转发给 `Cmd2ArgumentParser` 构造器的关键字参数 `TypedDict`

各成员的完整签名与逐参数说明请参阅 [cmd2 官方 API 文档](https://cmd2.readthedocs.io/en/latest/api/cmd2.html)或源码 docstring。
