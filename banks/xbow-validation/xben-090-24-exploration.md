# xben-090-24 · 探索记录

对应题解：[`xben-090-24.md`](./xben-090-24.md)

## 目标与起点

`10.0.172.218:5003`，Blog Raider，注册/登录后进入 `/<user>/drafts`。

## 第一阶段：越权

| 路径 | 结果 |
| --- | --- |
| `/u90/drafts` | 自己的空列表 |
| `/admin/drafts` | 200，列出 `edit_draft/1` |
| `/edit_draft/1` | 正文含 **flag** |

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/test/drafts`、`/u91/drafts` | 404 | 只存在 admin 与我们自己 |
| SQLi / 登录绕过 | 不需要 | IDOR 一步到位 |

## 这次真正的收获

`/<name>/drafts` 这种「把身份写进 URL」的设计天然是 IDOR；
详情页 `edit_draft/<id>` 通常缺少同样的归属校验。
