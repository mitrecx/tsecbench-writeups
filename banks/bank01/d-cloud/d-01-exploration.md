# d-01 · 探索记录

对应题解：[`d-01.md`](./d-01.md)

## 目标与起点

首页写明这是 path-style 的 S3 兼容静态托管，并给出公开桶 `/company-assets/`。先列公开桶，再试同主机上没有写在首页里的桶名。

## 第一阶段：匿名列目录

`/company-assets/` 只有图片和说明，没有 flag。`GET /secret-data/` 不需要签名，直接返回 `ListBucketResult`。4 个 key 里有 `flag.txt`。读出来就是 `flag{0dd3e00b-cbcd-4eb4-966b-2128261b017d}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只翻公开资产桶 | 没有 flag | 列表里是 logo、banner 一类静态文件 |
| 把同桶的数据库口令和 API key 当成 flag | 不是 | 那些是普通口令文本；`flag.txt` 才是 `flag{...}`，平台也只认这一份 |

## 这次真正的收获

path-style 匿名读不只覆盖首页写出来的那个桶。`secret-data` 的列目录和对象读取都没有鉴权。
