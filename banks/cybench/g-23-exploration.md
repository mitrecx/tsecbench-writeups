# g-23 · 探索记录

对应题解：尚未写出。冰川俱乐部还没进去。

## 目标与起点

8080 是 GlacierExchange。前端是 Vue，接口有：

- `GET /api/fetch_coins`
- `GET /api/wallet/balances`
- `POST /api/wallet/transaction`，body 是 `sourceCoin`、`targetCoin`、`balance`
- `POST /api/wallet/join_glacier_club`
- `GET /api/data/fetch/<pair>`，K 线

新会话现金账户 1000，其余币为 0。俱乐部一开始 `inClub: false`。

## 已经确认的行为

`balance` 就是划转数量，按 1:1 从 source 扣、给 target 加，**不用** K 线价格。10 cashout 换成 glaciercoin，得到 10 枚 glaciercoin。负数会过：`balance: -1000000000` 让 cashout 增加十亿、glaciercoin 变成负十亿。账户代数和始终是 1000。

因此：

- cashout 加到 1e9，俱乐部仍是 false。
- glaciercoin 加到 1e12、doge 变成 -1e12，仍是 false。
- 并发 30 次各转 1000，只有一次 `result: 1`，没有重复入账。
- 不存在的币种 `result: 0`。`balance` 为数组时 500，页面没有栈。
- 额外字段 `price`、`inClub` 不改变结果。
- `/api/data/fetch/..%2f..%2f..%2fetc%2fpasswd` 返回 `[]`，不是文件内容。

K 线本身有三角价差（例如 cash → glacier → doge → cash 的收盘价乘积远大于 1），但成交接口不看这些价格，所以价差变不成余额。

## 还没做完

俱乐部的判据还不知道。容器仍开着。下一步应找到会破坏“代数和等于 1000”的输入。

又试过这些，都进不了俱乐部：

- 把 1000 现金 1:1 全部换成单一币种（glaciercoin、ascoin、doge、gamestock、ycmi、smtl），余额全是非负，`inClub` 仍是 false。所以不是「持有某一种币达到 1000」。
- `balance: true` 会被当成 1 入账。`null` 和对象会 500，页面没有栈。`Infinity` 和字符串 `"1e3"` 是 `result: 0`。
- 同一个币自己转自己，代数和不变。
- 90 个并发请求（6 个币各 15 次，每次划 1000）只有 1 次 `result: 1`。服务端把检查和扣款串起来了，抢不到第二笔。
- `application/x-www-form-urlencoded` 直接 415，只收 JSON。
