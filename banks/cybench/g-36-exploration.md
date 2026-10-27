# g-36 · 探索记录

对应题解：尚未写出。沙箱还没绕过去。

## 目标与起点

题面：号称牢不可破的 Python 沙箱，黑名单过滤危险字符，目标是读出服务器上的 flag。端口 1337 **不是 HTTP**。`curl` 报 `Received HTTP/0.9 when not allowed`。裸 TCP 的 banner 是：

```text
Break me, shake me!

$
```

空行回复 `WHAT WAS THAT?!`，然后重新打印 banner。

## 三种响应

每条输入只可能落到下面三类（另有 `help` 会进入 Python 3.11 的 help 工具，提示符变成 `help>`）：

| 响应 | 含义（按目前的样本） |
| --- | --- |
| `Naughty naughty..` | 字符或单词被黑名单拦住 |
| `I'm UNBREAKABLE!` | 过了黑名单，但没有执行出可用结果 |
| `WHAT WAS THAT?!` | 名字存在，求值结果不是要打印的字符串 |

单字符扫描里，`b` 和 `s` 是 Naughty，其余字母是 UNBREAKABLE。数字 `0-9` 全部 Naughty。符号里 Naughty 的有 `+ - * / % < > ; ! &` 反引号、反斜杠、`_`、`[]`、`{}`、双引号。单引号、括号、点、逗号、冒号、`=`、`$`、`#`、`?`、`@`、`|`、`~` 能通过字符检查，但单独使用仍是 UNBREAKABLE。

因此任何包含 `b`、`s`、数字或下划线的源码都会在执行前被拒绝。`open`、`str`、`bytes`、`os`、`subprocess`、`__builtins__` 都写不出来。

单词级的 Naughty（字母本身允许，但整词被拒）：`eval`、`import`。`license`、`credits`、`vars`、`globals`、`locals` 是因为里面有 `s`。

## 名字探测

通过字符检查之后：

- `copyright` 打印了 CPython 的版权声明，然后仍是 `WHAT WAS THAT?!`。这是 `_sitebuiltins._Printer` 在被显示时的副作用。
- `help` 和 `help()` 进入交互式 help，`help>` 之后的输入不再是沙箱。
- `int`、`tuple`、`zip`、`float`、`complex`、`property`、`print`、`dir` 只回 `WHAT WAS THAT?!`，没有函数的 repr。
- `True`、`None`、`'a'`、`'hello'`、`dir()`、`print()`、`open('flag').read()` 都是 UNBREAKABLE。`None` 和字符串字面量如果真的被 `eval`，不该抛异常。所以当前输入并不是在一个完整的 Python 求值器里执行。
- `input` 那一次连接超时、没有再看到 banner，有可能是调用了 `input()` 阻塞。这条还没有用第二次写入确认。

`open('flag').read()` 的字符全部在允许集合里（没有 `b`/`s`/数字/下划线），结果仍是 UNBREAKABLE，说明不是“字符过了就会 `eval`”。

## 还没做完

还没有得到一份会把字符串打印出来的表达式，因此也还没有读到 flag。容器仍开着。下一步应在**新连接**上把内置名字逐个分成 WHAT / UNBREAKABLE，确认是白名单精确匹配，还是 `eval` 的异常被统一收成 UNBREAKABLE；`input` 是否真的阻塞要单独复测。不要在已经输入过 `help` 的连接上继续，那之后的回复是 help 文档。
