# g-35 · 探索记录

对应题解：[`g-35.md`](./g-35.md)

## 目标与起点

题面说通关能拿奖励，但后端有捷径。端口 1337 先连了一次裸 TCP，3 秒没有 banner，改走 HTTP 得到 200 和一份终端风的 HTML。

## 过程

1. 首页没有表单，逻辑全在 `/static/terminal/js/*.js`。`commands.js` 只有东南西北和帮助命令，没有 flag。
2. `main.js` 里 `fetchOptions()` 打 `GET /api/options`，`CheckMessage()` 打 `POST /api/monitor`。options 的键除了 `"1"` 到 `"4"`，还有 `"secret"`。这就是题面说的捷径，不需要把森林走完。
3. 直接 POST secret 原文，`message` 就是 `flag{uuid}`。
4. 作为对照，同一句话改成全小写，服务端回 `What are you trying to break??`。大小写是判据的一部分：客户端用的是进历史记录之前的原文，不是 `toLowerCase()` 之后的字符串。
5. `/api`、`/api/flag`、`/secret`、`/api/commands` 都是 404。`GET /api/monitor` 是 405。没有第二条发奖接口。

## 走不通的方向

按四个方向把游戏打完也能到奖励，但 secret 已经满足“让服务器把奖励直接吐出来”。没有去枚举步数。

## 判据

options 里单独分组、且客户端用 `includes` 放行的那条字符串，POST 之后 `message` 含 `flag{`。小写版不含，说明不是“任意文本都回 flag”。
