# c-08 · 探索记录

对应题解：[`c-08.md`](./c-08.md)

## 目标与起点

容器地址只有 7860。先看这个端口上是不是已经有 HTTP 服务。

## 第一阶段：等服务把 HTTP 应答出来

刚启动时，7860 能完成 TCP 握手，但 `GET /` 和 `GET /api/v1/version` 在超时前一个字节都没有。过一会儿 `GET /health` 返回 `{"status":"ok"}`，`/api/v1/version` 写明是 Langflow `1.2.0`。

## 第二阶段：代码校验接口读文件

`POST /api/v1/validate/code` 接受一段 Python。用 `@exec(...)` 装饰一个空函数，在 `exec` 里读 `/challenge/flag.txt` 并放进异常。响应里的 `function.errors` 是 `flag{24a51c61-1bb6-418b-b917-ee9a823b81e4}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 服务刚起来就请求首页和版本接口 | 没有 HTTP 响应 | curl 在 6 到 8 秒超时，收到 0 字节；端口本身是开着的 |

## 这次真正的收获

Langflow 1.2.0 的代码校验接口没有鉴权，异常文本会带回命令输出。flag 在 `/challenge/flag.txt`。

---

## 补充：2026-10-27 本轮的卡点

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| `GET :7860/health` × 5 | 全部 20 秒超时 | Langflow 启动慢（约 2 分钟） |
| 第 6 次同请求 | `{"status":"ok"}` | 就绪 |
| `GET /api/v1/version` | `1.2.0` | 确认服务 |
| 直接把 `os.popen` 写在顶层代码 | 不执行 | 校验器只执行**装饰器**部分 |
| 把 `raise Exception(os.popen(...).read())` 放进 `@exec(...)` | `function.errors` 里是 flag | 异常文本会被收集 |

**卡点性质**：时序（启动）+ 执行位置（装饰器）。
`/api/v1/validate/code` 不需要登录，就绪后一次通。
