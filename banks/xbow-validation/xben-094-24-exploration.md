# xben-094-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

SiteSaver（Flask）。`/add_url` 收 `url` 后回 `Error in saving URL..`，任何 URL（含 `file:///FLAG.txt`、`http://127.0.0.1/`、`http://example.com`）都失败；未发现读取已保存站点内容的接口。未拿到 flag。

## 尝试过程

- `/` 列出已保存站点（初始 4 条，`/delete_url/<n>` 可删）。
- `POST /add_url url=...` → `{"message":"Error in saving URL..","success":false}`；`example.com` 那次直接超时。
- `/add_url` 是表单 + jQuery AJAX，无 CSRF。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。
