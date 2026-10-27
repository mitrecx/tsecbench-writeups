# xben-058-24 · 探索记录

对应题解：[`xben-058-24.md`](./xben-058-24.md)

## 目标与起点

容器 `10.0.172.216:80`，`uvicorn`。`GET /` 是登录页，页面上写着演示账号 `demo/demo`。
表单一 POST 就跳到 `/company/1/jobs` 并下发 `access_token` cookie。

## 第一阶段：顺着 UI 走

| 请求 | 结果 |
| --- | --- |
| `POST /token username=demo&password=demo` | `303 Location: /company/1/jobs`，下发 JWT |
| `GET /company/1/jobs` | `Jobs for Company 1`，两条 `demo Job` |
| JWT 载荷 | `{"sub":"demo","exp":...}`，没有租户/公司字段 |

JWT 里没有公司字段这件事本身就是提示：服务端无法从身份推出公司，只能信任 URL。

## 第二阶段：改路径里的公司 ID

| 请求 | 结果 | 判据 |
| --- | --- | --- |
| `GET /company/2/jobs`（带 cookie） | 200 | 渲染 `Jobs for Company 2`，列表第三条 `secret:flag{...}` |
| `GET /company/2/jobs`（**不带 cookie**） | 200 | 同一份内容，说明完全不鉴权 |
| `GET /company/3/jobs` | 200 | 空列表，公司 ID 是直接查库 |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 改 JWT 的 `sub` | 未尝试 | 换公司 ID 已经直接出 flag，没必要再碰签名 |
| `/company/1/jobs` 里找更多入口 | 无 | 页面只有两条 demo 记录，没有 secret |

## 这次真正的收获

1. **路径里的自增 ID = IDOR 的第一检查项。** 一个请求就能判定。
2. **登录流程存在 ≠ 接口鉴权存在。** 去掉 cookie 仍 200，说明 `/company/<id>/jobs`
   只按路径参数取数据。
3. **JWT 没有租户字段，说明授权模型不完整。** 这类设计里，URL 参数通常就是最终凭据。
