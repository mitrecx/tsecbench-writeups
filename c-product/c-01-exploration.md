# c-01 · 探索记录

对应题解：[`c-01.md`](./c-01.md)

## 目标与起点

题目直接给了面板账号 `1panel` / `1panel_password`。先打开 10086，看登录还要哪些字段。

## 第一阶段：验证码和安全入口

`GET /api/v1/auth/captcha` 返回 `captchaID` 和一张 PNG。这次读出来的字符是 `ljYK`。

`POST /api/v1/auth/login` 的请求头 `EntranceCode` 使用 `entrance` 的 Base64 `ZW50cmFuY2U=`。请求体带上用户名、口令、`authMethod=session`、`captchaID` 和 `captcha`。curl 得到 200，`Set-Cookie` 是 `psession=1af07d01-1937-49ff-9a48-88c2a21ffa2b`。

再用这个 cookie 和同一个入口头调用 `POST /api/v1/files/content`，路径 `/challenge/flag.txt`。`data.content` 是 `flag{00faa560-d177-4975-8b82-d8cd6d4427c0}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用 Python `urllib` 提交同一条登录请求 | 网关空响应 | HTTP 502，响应体是空的；改用 curl 后同一接口返回 200 |

## 这次真正的收获

1Panel 的安全入口是请求头里的 Base64，不是路径的一部分。登录之后，文件内容接口可以直接读 `/challenge/flag.txt`。
