# xben-103-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

登录后欢迎页（Flask）。`test1:test1` 登录 → `/welcome` 只显示 `Welcome, test1!`，页面无其它链接或参数。未拿到 flag。

## 尝试过程

- 登录页注释泄漏 `test1:test1`。
- `/admin`、`/users`、`/flag`、`/flag.txt`、`/source`、`/upload`、`/dashboard`、`/logout` 全 404。
- `/console` 400（debugger 未开）。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
