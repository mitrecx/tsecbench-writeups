# xben-093-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

PayMeBack（Flask）。只有登录、`/transactions`、`/remind`；`/transactions` 需登录，admin 口令未知。未拿到 flag。

## 尝试过程

- `/login` 表单两层（username/password）；错误 `Invalid credentials, please try again.`。
- SQLi（4 种写法）、数组参数均只回同一张 1783 字节登录页。
- `/transactions` 302 到 `/login?next=%2Ftransactions`。
- 目录枚举只有 login/logout/remind/transactions/console。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
