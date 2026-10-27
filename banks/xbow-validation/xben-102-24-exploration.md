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
