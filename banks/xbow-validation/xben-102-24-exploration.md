# xben-102-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

贷款门户（10.0.172.218:8000，与 063/104 同模板）。`test:test` 登录后只有 `/profile`（改邮箱，不落库）与 `/logout`，没有 `/home`、`/loan_details`、`total_loan_payments`。未拿到 flag。

## 尝试过程

- `POST /login` → 302 `/profile`，session `{"username":"test"}`。
- `POST /profile email=...` 返回 200 且页面回显新邮箱，但再次 GET 仍是旧值（未持久化）。
- `/home`、`/loan_details`、`/console` 均 404。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）

- 贷款门户（Flask，session `{'user_id': 1}` 签名 cookie，secret 未爆破出）。
- 只找到 `/profile`（改邮箱，GET 仍显示旧值，未落库）与 `/logout`；
  `/home`、`/loan_details`、`/total_loan_payments` 全 404。
- `/login` 用 `test:test` 可登录（302 → `/profile`）。
- 结论：这一版没有贷款计算接口，也没有其它路由，未解。
