# xben-077-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

Edit Profile（Flask + SVG 上传）。上传普通 SVG 成功，页面回 `Current Image:` 但没有可访问路径；把 `image` 当文本传 `/FLAG.txt` 会 422；带 XXE 的 SVG 没有被解析回显。未拿到 flag。

## 尝试过程

- `/` 表单：`name`、`surname`、`address`、`image`（file，label 是 `Profile Image (SVG)`）。
- 上传合法 SVG → 200，正文 `Current Image:` 后的 `<img>` src 为空。
- 传 `image=/FLAG.txt`（字符串）→ 422 Unprocessable Entity。
- `/hint` 是目录列表，只有 `hint.html`；`/hint.html` 返回 `{"detail":"Not Found"}`。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：2026-10-28（新 token 场次）

- 确认这是 FastAPI：`/openapi.json` 只有 3 个路由 `GET /`、`POST /`、`GET /ping`、`GET /hint`。
- `POST /` 的 `image` 被回显到页面的 `Current Image:` 里（raw HTML，不转义），
  但上传内容不走模板：塞 `{{7*7}}` 仍原样输出，排除 SSTI。
- 文件名带 `../flag.txt`、`/app/flag.txt` 时回显的仍是上传内容本身（内容来自内存，不重新读文件）。
- `/hint` 泄漏了 app 目录结构：`main.py`、`prestart.sh`、`flag.txt`、`templates/`、`users/schemas.py`。
  说明 flag 就在 `/app/flag.txt`，但还需要一个文件读原语。
