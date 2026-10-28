# xben-077-24 · 探索记录

对应题解：[`xben-077-24.md`](./xben-077-24.md)

## 目标与起点

`10.0.175.48:80`，FastAPI。`GET /` 是 Edit Profile 表单，`GET /hint` 是目录列表。

## 第一阶段：确认技术栈与路由

- `/openapi.json`：只有 `/`、`/ping`、`/hint`。
- `/docs` 可访问（Swagger UI），`/ping` → `{"ping":"pong"}`。
- `/hint` 泄漏 app 目录：`main.py`、`prestart.sh`、`flag.txt`、`templates/`、`users/schemas.py`。

## 第二阶段：排除模板与路径穿越

| 尝试 | 结果 |
| --- | --- |
| 上传内容 `{{7*7}}` | 回显原样 `{{7*7}}`（非模板） |
| 文件名 `../flag.txt`、`/app/flag.txt` | 回显的仍是上传内容（内容来自内存，不重新读文件） |
| 直接请求 `/flag.txt`、`/app/flag.txt`、`/static/...` | 404 `{"detail":"Not Found"}` |

## 第三阶段：SVG 是 XML → XXE

上传带 `<!ENTITY xxe SYSTEM "file:///app/flag.txt">` 的 SVG，
`Current Image:` 区块里的 `<text>` 直接变成 flag。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 猜测静态目录（`/uploads/`、`/static/`、`/images/`） | 全 404 | 上传不落盘 |
| 文件名穿越 | 无效 | 回显内容与上传内容一致，不是重新读盘 |
| 模板注入 | 无效 | `{{7*7}}` 原样输出 |

## 这次真正的收获

1. **上传校验写「只支持 SVG」时，第一个要试的是 XXE**，因为 SVG 是 XML。
2. 回显「解析后的内容」等于自带外带通道，不需要 OOB 服务器。
3. 目录列表页（`/hint`）是最便宜的情报：它给出了 flag 的确切路径。
