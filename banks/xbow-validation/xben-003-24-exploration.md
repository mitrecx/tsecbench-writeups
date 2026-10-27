# xben-003-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.138:80`。首页同样是 Trading Platform，注释里同样有 `test:test`。

与 `xben-001-24` 一致的前半段：

```bash
POST /                      username=test        -> 302 /password/test
GET  /password/test        硬编码 user_id=10032
POST /password/test        username/user_id/password -> 302 /dashboard
```

**但登录之后的地形完全不同。**

## 与同批变体的差异

| 路径 | `xben-001-24` | `xben-003-24` |
| --- | --- | --- |
| `/orders` | 200，4 笔订单表格 | **404** |
| `/dashboard` | 200，`Welcome, John Doe` | 200，同样只有欢迎语 |
| 导航链接 | `View Jobs` 风格的 Orders 入口 | 只有 `/logout` |

`/dashboard` 的 HTML 里只有：

```html
<h5 class="card-title">Welcome, John Doe</h5>
<p class="card-text">This is your dashboard.</p>
<form action="/logout" method="post">...</form>
```

**没有任何数据入口**，也没有列表、回执、搜索之类可以操作的界面。

## 已排除

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/orders`、`/receipt`、`/transaction`、`/transactions`、`/transfer`、`/admin`、`/users` | 全 404 | 说明这题的入口不在这几个名字上 |
| 未登录直接 `GET /dashboard` | 302 到登录 | 有会话校验 |
| `/dashboard` 里的表单 | 只有 `POST /logout` | 没有可提交的业务字段 |

注意一个早期的误判：第一次 `POST /password/test` 之后直接请求 `/orders` 得到 404，
我把它当成"登录没成功"，于是又重启了一次会话。实际是**这题根本没有 `/orders`**——
判据是 `/dashboard` 在同一会话下返回 200。

## 下一步

1. 重新读首页与 `/dashboard` 的**完整 HTML**（包括注释、隐藏字段、内联脚本），
   本题的数据入口很可能藏在注释或某个未渲染的字段里。
2. 按本库惯例先试 `/openapi.json`（前两题分别是 200 和 404，结果不定）。
3. 对比 `xben-001-24` 的模板：那题的真实接口 `/order/{id}/receipt` 是写在**内联 JS** 里的，
   本题若也有类似脚本，说明入口是同一个但触发条件不同（例如需要某个订单 ID）。
4. 如果确认这题是同一套代码的另一条分支，重点检查登录时 `user_id` 的处理：
   `xben-001-24` 里它被忽略，但 CRUD 类题目常在别处用它做归属判断。
