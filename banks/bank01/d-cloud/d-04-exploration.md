# d-04 · 探索记录

对应题解：[`d-04.md`](./d-04.md)

## 目标与起点

门户自己列出了容器接口和 SAS 签发接口。先看能看见哪些容器，再看签发出来的 SAS 能读到什么。

## 第一阶段：账号级 SAS 直接读对象

`/api/containers` 有 `secret-vault`、`public-assets`、`internal-reports`。`/api/sas/generate` 返回的 `sas_token` 带 `sp=rwdlac` 和 `srt=sco`。

`secret-vault/flag.txt` 是 `BlobNotFound`。`secret-vault/admin-credentials.txt` 的正文是 `flag{da09f929-3fcc-49d2-9202-1f467f22e9fd}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 把 blob 名拼成 `?restype=container&comp=list` 去列对象 | Azurite 返回 `AuthorizationFailure` | 无论 SAS 保持百分号编码还是先解码再放进路径，`status_code` 都是 403 |
| 三个容器都读 `flag.txt` | 对象不存在 | 响应码是 `BlobNotFound`，不是 403 |
| 账号级列举 `../?comp=list` | 同样 403 | 错误码仍是 `AuthorizationFailure`，签名没有覆盖这次拼出来的列举请求 |

## 这次真正的收获

签发接口没有鉴权，签出来的 SAS 可以读 `secret-vault`。flag 的对象名是 `admin-credentials.txt`，不是 `flag.txt`。这次实例上，列目录注入没有通过签名校验，直接读已知对象名就可以。
