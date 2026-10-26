# e2-02 · 探索记录

对应题解：[`e2-02.md`](./e2-02.md)

## 目标与起点

`GET /` 写明这是一个受限 unpickler，输入是 base64 的 pickle。先交普通对象，确认还原通路，再看哪些全局名会被拒绝。

## 第一阶段：白名单用 TypeError 和 Forbidden 区分

`pickle.dumps(1)` 和 `pickle.dumps("hi")` 都正常返回 `result`。自己写的 `__reduce__ -> eval` 被拒绝，原文是 `Forbidden: builtins.eval — only whitelisted classes allowed`。

接着用协议 0 的 `GLOBAL builtins.<name>` 加空参数 `REDUCE` 扫名字。被调用起来的名字会报参数错误，被白名单拒绝的名字会报 Forbidden。

放行并能无参构造的有 `list`、`dict`、`bytes`、`bytearray`、`tuple`、`set`、`str`、`int`、`bool`、`frozenset`、`float`。`getattr` 和 `type` 被放行，但空参数调用分别是 `getattr expected at least 2 arguments, got 0` 和 `type() takes 1 or 3 arguments`。`object` 本身作为 `GLOBAL` 是 Forbidden。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `eval` / `__import__` / `exec` / `open` | 白名单拒绝 | `Forbidden: builtins.<name>` |
| 直接 `GLOBAL builtins.object` | 白名单拒绝 | 同上，所以不能把 `object` 写进 pickle 的全局名 |
| 第一版属性链 | 参数顺序错了 | `TypeError: attribute name must be string, not 'type'`。`MARK` 把 `getattr` 自己卷进了参数元组 |
| `read` 用 `MARK` + 空元组 + `TUPLE` | 参数类型错了 | `argument should be integer or None, not 'tuple'`。那样拼出来的是 `((),)`，不是空参数 |
| `object.__subclasses__()` 经 `type.__base__` | 列出 492 个类 | `result` 文本里有 `<class 'os._wrap_close'>`，这次下标是 141 |
| 取到 `popen` 再调用 | 得到文件对象 | `result` 是 `<os._wrap_close object at 0x...>`，说明命令已经启动 |
| 对这个对象 `read()` | 读到 flag | `result` 是 `flag{...}`，平台接受 |

## 这次真正的收获

白名单挡的是全局名，不是属性访问。`type` 和 `getattr` 足够走到 `object.__subclasses__`，再从 `os._wrap_close.__init__.__globals__` 拿出 `popen`。无参调用在 pickle 里要用 `EMPTY_TUPLE`（opcode `)`），不要再包一层元组。
