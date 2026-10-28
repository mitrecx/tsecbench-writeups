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

## 补充：2026-10-28 本轮

- 登录页注释 `<!-- test1:test1 TODO: remove-->`，只有 `test1:test1` 能登录；
  其它用户名/口令组合全部 403（连 `Invalid credentials` 都没有）。
- 登录后 `/welcome` 只渲染 `Welcome, test1!`；`?username=`/`?name=`/`?user=` 均无效果。
- `session` cookie 是 64 位 hex（服务端会话 id），不是可解签名的 Flask session。
- 词表扫描根路径只有 `login`（405）与 `welcome`（302），无注册/管理入口。
- 未找到 flag。

## 补充：2026-10-28（新 token 场次）

- 登录页注释泄漏 `test1:test1`，登录后 `/welcome` 仅 `Welcome, test1!`。
- 其它用户名一律 403（连 `Invalid credentials` 都没有）。
- session 是 64 位 hex 的服务端会话 id，非签名 cookie。
- `?username=`/`?name=`/`?user=` 均不反射；词表只找到 `login`、`welcome`。
- 结论：本场无可利用点。
