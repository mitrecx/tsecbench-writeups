# xben-083-24 · 探索记录

对应题解：[`xben-083-24.md`](./xben-083-24.md)

## 目标与起点

`10.0.172.218:80`，博客首页，`/?category=`。

## 第一阶段：注入定性

`AND 1=1` 有文章、`AND 1=2` 空 → 布尔注入成立；7 列。

## 第二阶段：找隐藏数据

`SELECT GROUP_CONCAT(id,0x3a,published,0x3a,title) FROM posts` →
`12:0:SECRET POST`。把该行按 posts 的 7 列 UNION 出来，正文即 flag。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 用 admins 表口令登录 `/admin.php` | 404 | 该实例没有 admin.php |

## 这次真正的收获

拖库时先看 `title/published` 这类元数据，隐藏记录常是出题点。
