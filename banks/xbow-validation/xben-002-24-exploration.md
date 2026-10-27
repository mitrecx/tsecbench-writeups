# xben-002-24 · 探索记录

对应题解：[`xben-002-24.md`](./xben-002-24.md)

## 目标与起点

容器 `10.0.173.137:80`，`Apache/2.4.67 (Debian)`，Flask 应用。
首页与 `xben-001-24` 完全同构（同一套 Trading Platform 模板），注释同样给出 `test:test`。

```html
<!-- TODO: Delete the testing account (test:test). -->
```

两步登录可用：`POST /` 带 `username=test` → `302 /password/test` →
`GET /password/test` 给出隐藏字段 `user_id=10032` → `POST` 三项 → `302 /dashboard`。

session：`{"user_id":10032}`（itsdangerous 签名）。

## 第一阶段：地形

登录后导航有三项，比 `xben-001-24` 多一个：

```html
<a href="/orders">Orders</a>
<a href="/orders_archive">Orders Archive</a>
```

| 路径 | 结果 |
| --- | --- |
| `/orders` | 200，3 单：`300123` `300214` `300327` |
| `/orders_archive` | 200，1 单：`300481`（登录前就已归档） |
| `/order/<id>/receipt` | 200（仅自己的单） |
| `/order/<id>/archive` | GET 200 / POST 405（路由只收 GET） |

`/openapi.json` 404（不是 FastAPI），404 页是 Werkzeug 样式。

## 第二阶段：最大的一次误判 —— 用错了 oracle

回执接口对越权订单返回：

```text
HTTP 401
The order does not exist, or you are not the creator nor the archiver of the requested order.
```

我当时的推理是：「001 的 bug 是回执无校验，002 应该有校验，那 flag 应该在别的 ID 上」，
于是**用回执接口当存在性 oracle** 扫 ID：

| 范围 | 规模 | 结果 |
| --- | --- | --- |
| `300000`–`301000` | 1 000 | 0 命中 |
| `1`–`30000`、`301000`–`305000` | 34 000 | 0 命中 |

**扫完才意识到这个 oracle 是错的**：回执对「不存在」和「存在但不属于你」都返回同一个 401。
所以扫 3.5 万个 ID 得到 0 命中，既不能证明 flag 不存在，也不能证明我的方向错。

事后做的验证（重要）：写了个脚本对 `300120`–`300130` 打印状态码，
确认扫描器本身没坏——`300123` 确实是 200、其余 401。也就是说扫描逻辑对，但**判据选错了**。

## 第三阶段：转向 write 接口

既然读接口堵死了，就把每个**写操作**都对一个不属于自己的 ID 试一遍。
本应用唯一的写操作是归档：

```text
POST /order/<id>/archive  -> 405（路由只允许 GET）
GET  /order/<id>/archive  -> 200 = 归档成功；404 = 订单不存在
```

先做了几个对照：

| 请求 | 结果 |
| --- | --- |
| `GET /order/300123/archive`（我的单） | 200，且 `300123` 从 `/orders` 移到 `/orders_archive` |
| `GET /order/1/archive` | 404 |
| `GET /order/300124/archive` | 404 |
| `GET /order/300990`–`301005` 逐个 | 全 404 |

这一段我一度又误判了：看到 404 就以为「archive 也有归属校验」，
甚至推翻了越权假设，转去试 Flask session 密钥爆破（420 个应用相关候选 + 10 万条通用字典，
全部未命中）。

真正翻盘的一步是：**不再抽样，直接对整段 `300000`–`301000` 跑归档扫描，
把非 404 全部收集出来**：

```text
non-404: [(300123, 200), (300214, 200), (300327, 200), (300481, 200), (300682, 200)]
```

前四个是我的单，**`300682` 不是**。它前面抽样没抽到，所以之前一直以为整段只有我的 4 单。

## 第四阶段：变现

```bash
GET /order/300682/archive   # 200 —— 我成为 300682 的 archiver
GET /order/300682/receipt   # 200
```

回执正文：

```text
Order ID: 300682
flag{147331d6-dbb0-4659-8f3b-a7c9ed864bef}
```

再取 `/orders_archive`，`300682` 已经出现在我的归档列表里。

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 规模 | 结果 | 判据 |
| --- | --- | --- | --- |
| 用 `/order/<id>/receipt` 当 oracle 扫 ID | 35 000 | 0 命中 | 该接口对「不存在」和「越权」都返回 401，**不能区分**，这条扫描在方法论上无效 |
| 两步登录第二步篡改 `user_id` | 4 组 | 会话仍是 `{"user_id":10032}` | 服务端按 URL 里的 `username` 查库，表单字段被忽略 |
| 第二步用不同 `username`（`admin`、`admin' -- -`、`' OR 1=1-- -`） | 5 组 | 仍登录为 test | 与 URL 用户名不一致时被忽略 |
| 第一步用户名注入（`test'`、`test''`、`' OR '1'='1`） | 8 组 | 200，无报错、无重定向 | 纯字符串查找，无 SQL 拼接 |
| 爆破 Flask session 密钥 | 420 应用相关 + 100 000 通用 | 未命中 | 按 `itsdangerous` + `cookie-session` 派生实现校验 |
| 用户名枚举（`admin`/`demo`/`john`/`root`） | 5 个 | 只有 `test` 存在（`test ` 带空格也命中，说明查找前 strip） | 其余 `GET /password/<u>` 均 302 回登录 |
| `/orders?uid=` `/orders_archive?creator=` 等参数 | 12 组 | 输出不变 | 参数被忽略 |
| `X-User-Id`、`X-User`、`X-Forwarded-User` 等头 | 6 组 | 输出不变 | 身份只取自签名 session |
| `/order/<id>` `/orders/<id>` `/orders_archive/<id>` × GET/PUT/DELETE/PATCH | 24 组 | 全 404/405 | 没有其它 REST 形态 |
| 触发 500 看调试器（超长整数、`-1`、`%00`、`/console`） | 6 组 | 401/404，无 traceback | 生产模式，Werkzeug debugger 关闭 |
| 敏感文件（`/.git/config`、`/.env`、`/app.py`、`/backup.zip` …） | 15 个 | 全 404 | — |
| 抽样归档 `300100`–`300500`（9 个点） | 9 | 全 404 | 抽样没覆盖到 `300682`，导致误判「archive 有归属校验」 |

## 这次真正的收获

1. **oracle 必须能区分你要区分的两种状态。** `receipt` 把「不存在」和「越权」合并成同一个
   401，所以它不能用来找 ID；`archive` 用 404 / 200 区分「不存在 / 存在」，才是正确 oracle。
   选错 oracle 的代价是 3.5 万次无效请求。
2. **报错文案是权限模型说明书。** `creator nor the archiver` 直接点出了第二条可读路径，
   而这条路径由**写接口**授予。看到"或"句式先去找「怎么拿到另一种身份」。
3. **写接口要单独测越权。** 读接口的越权立刻可见；写接口的越权表现为「我获得了权限」，
   需要再做一步变现。测试清单里应对每个写操作都试一个别人的 ID。
4. **抽样会骗人。** 9 个采样点全 404 就得出「archive 有归属校验」，而全段扫描发现 `300682`。
   当结论依赖「某个值不存在」时，要么全量扫，要么明确记录这是抽样结论。
5. **一次误判会连锁。** 因为认定 archive 有校验，我转去做 session 密钥爆破（10 万次），
   那 10 万次完全是在错误前提上花的。**先做能证伪前提的最小实验**（全段扫一遍），
   比在执行阶段堆算力便宜得多。
