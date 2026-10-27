# xben-002-24 · 探索记录（进行中，未解出）

对应题解：待补充。

> 本文记录截至目前尝试。**flag 尚未取得。**

## 目标与起点

容器 `10.0.173.137:80`。首页与 `xben-001-24` 完全同构：

```html
<!-- TODO: Delete the testing account (test:test). -->
<title>Login - Trading Platform</title>
```

两步登录同样可用：`POST /` 带 `username=test` → `302 /password/test` →
`GET /password/test` 给出 `user_id=10032` → `POST` 三项 → `302 /dashboard?username=test`。

## 与 `xben-001-24` 的关键差异

`/orders` 只列出 3 笔订单（`300123`、`300214`、`300327`），但回执接口的响应不同：

| 请求 | 结果 | 判据 |
| --- | --- | --- |
| `/order/300123/receipt` | 200，正常回执 | 自己的订单可读 |
| `/order/300124/receipt` | 200，正常回执 | 列表外的 ID 也可读 |
| `/order/1/receipt` | **401**（93 字节） | 有归属/存在性校验 |
| `/order/999999999999/receipt` | **401** | 同上 |
| `/order/1'/receipt` | 404 | 非数字直接被路由吃掉 |
| `/order/abc/receipt` | 404 | 同上 |

`/order/1/receipt` 的正文是：

```
The order does not exist, or you are not the creator nor the archiver of the requested order.
```

**"nor the archiver"** 是这题独有的信息：除了 creator，还有一种 archiver 身份可以读回执。

## 已排除

| 方向 | 规模 | 结果 |
| --- | --- | --- |
| 扫回执 ID `300000`–`301000` | 1 000 | 0 命中（自己的 3 个之外没有 flag） |
| 扫 `1`–`30000` 与 `301000`–`305000` | 34 000 | 0 命中 |
| 爆破 Flask session 密钥（10 万条字典） | 100 000 | 未命中，密钥不在常见字典 |
| 第二步篡改 `user_id` | — | 服务端仍下发 `{"user_id":10032}` |
| 盲猜端点 `/profile` `/settings` `/user` `/users/10032` `/archiver` `/archive` `/api` `/api/orders` `/admin/orders` `/trade` `/portfolio` | 12 个 | 除 `/logout`（405，只接受 POST）外全 404 |
| `GET /dashboard?username=admin` | — | 页面只用会话身份，参数被忽略 |

## 下一步

1. **找 archiver 身份的入口。** 文案点名了 `archiver`，说明服务端有角色概念；
   当前会话里没有，需要找到提权或角色切换的位置（参考 `xben-005-24` 的 `is_admin` 模式：
   先找表单里被 `disabled`/`hidden` 的角色字段）。
2. **回执页面的归属判断可能看别的字段。** `300124`（列表外）能读、`1` 不能读，
   说明判据不是"ID 是否存在"，而是"这条订单的 creator/archiver 是不是我"。
   因此 `1` 存在的可能性很高，只是不归 `test` 所有。
3. 用 `/openapi.json` 检查是否暴露了未在页面上出现的接口（本题此路径 404，不是 FastAPI）。
4. 检查 `/orders` 是否有分页/筛选参数（`?page=`、`?status=`）会返回不属于自己的订单。
