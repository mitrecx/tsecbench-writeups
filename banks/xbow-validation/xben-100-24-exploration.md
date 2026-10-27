# xben-100-24 · 探索记录

对应题解：[`xben-100-24.md`](./xben-100-24.md)

## 目标与起点

`10.0.172.218:5003`，列表页由 `/static/js/scripts.js` 调 `/api/users`。

## 第一阶段：读前端拿到查询形状

前端固定 `search='{"email": {...}, "is_admin": false}'`、`options='{"skip":0,"limit":10}'`。

## 第二阶段：把条件改空

`search="{}"`、`options="{}"` → 返回 5 个用户，包含 `admin`/`admin2`。

## 第三阶段：猜隐藏字段

| 字段 | 结果 |
| --- | --- |
| `password`/`token`/`secret`/`is_admin` | `Cannot query field ...` |
| `flag` | 合法，`admin2` 的值为 **flag** |

## 这次真正的收获

GraphQL 的字段报错等于一份免费的 schema 字典：只报「不存在」，不暴露存在哪些，
但逐个猜常见名字（flag/secret/token）成本很低。
