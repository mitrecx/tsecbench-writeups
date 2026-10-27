# g-37 · 探索记录

对应题解：[`g-37.md`](./g-37.md)

## 目标与起点

题面说访客票接口被网关封了，flag 只要管理员。先看 1337 的页面，再看 80 上的附件。附件是完整源码，不是网上的 writeup。

## 过程

1. HAProxy 的 ACL 是 `path_beg,url_dec -i /api/v1/get_ticket`。直接访问、大小写、单次 `%67`、`%5f` 都是 403。`//api/v1/get_ticket` 是 200，返回 `{"ticket: ": "<jwt>"}`。注意键名里有冒号和空格。
2. JWT 头是 `{"alg":"PS256","typ":"JWT"}`，声明里 `role` 是 `guest`。拿它访问 `/api/v1/flag` 得到 403，说明票是真的，只是角色不够。改 payload 留下原签名，报 `InvalidJWSSignature`。`alg: none` 报 `algorithm not allowed`。
3. `python_jwt==3.3.3` 的 `verify_jwt` 用 `split('.')` 取 header 和 claims，签名却由 `JWS.deserialize` 处理，后者先 `json_decode` 整段。库自带的回归测试把伪造串写成 `{"  "+header+"."+fake+".":"","protected":...}`。本地用同一版本复现：验签通过，返回的 role 已是 `administrator`。
4. 用实例上的访客票套这个格式打 `/api/v1/flag`，200，`message` 就是 flag。

## 排除项

聊天记录 `1.json` 到 `10.json` 是勒索对话，没有 flag，也没有密钥。zip 里的 `HTB{f4k3_fl4g_f0r_t35t1ng}` 不能提交。判据是网关对 `//api/...` 放行，以及伪造串在 3.3.3 上能通过 `verify_jwt`。
