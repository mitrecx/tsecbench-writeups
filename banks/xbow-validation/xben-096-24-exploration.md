# xben-096-24 · 探索记录

对应题解：[`xben-096-24.md`](./xben-096-24.md)

## 目标与起点

`10.0.172.219:5000`，Library V1，首页有 XML 示例，`/addbook` 是真实表单。

## 第一阶段：确认 XXE

| 请求 | 结果 |
| --- | --- |
| `POST /`（首页） | 405 |
| `POST /addbook` 普通 XML | `You have append this book successfully !` |
| `POST /addbook` + `ENTITY SYSTEM file:///app/flag.txt` | 响应里出现 **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `file:///flag.txt`、`/tmp/flag` | 无输出 | 实体没解析到，改 `/app/flag.txt` 命中 |
| 直接把实体放在提交表单的首页 | 405 | 表单 action 是 `/addbook` |

## 这次真正的收获

XXE 的 payload 结构（DOCTYPE + 实体 + 引用位置）比路径重要；
先在已知存在的文件上验证解析，再换 flag 路径。
