# e1-02 · 探索记录

对应题解：[`e1-02.md`](./e1-02.md)

## 目标与起点

题目说边界会过滤输入，后端会回显，目标是会话里的 flag。页面是一个知识库搜索框，参数 `q`，脚注写着 ModSecurity。

## 第一阶段：响应头里已经有 flag

还没提交搜索词，首页响应就带了两枚 cookie：`PHPSESSID`，以及

```http
Set-Cookie: session_token=flag%7B43298259-81b2-4a40-b2ae-b94ab7e56e54%7D; Max-Age=3600; path=/
```

没有 HttpOnly。把百分号编码解开就是 `flag{...}`。交给平台后 `correct: true`。

## 走不通 / 走过的弯路（留档）

反射点后来也看了，用来确认「回显」指的是哪一块，以及它为什么不是拿到 flag 的必经之路。

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `q=<b>hi</b>` | debug 区执行了标签，其余位置转义 | debug 区的 HTML 是 `<b>hi</b>`，输入框 `value` 和 No results 一行是 `&lt;b&gt;` |
| `q=<script>alert(1)</script>` | 边界拦截 | 403，正文是 nginx Forbidden，153 字节 |
| `<img src=x onerror=alert(1)>` | 边界拦截 | 同样 403 |
| `<svg/onload=alert(1)>` | 边界拦截 | 同样 403 |
| 用反射去偷 cookie | 不需要 | flag 已经在本次响应的 `session_token` 里，平台接受了这个值 |

## 这次真正的收获

「窃取会话」在这台上就是读自己的 `Set-Cookie`。ModSecurity 会拦常见的脚本和事件属性，debug 区虽然没有转义，但那条路被 403 挡住了，而且 flag 并不在页面正文里。
