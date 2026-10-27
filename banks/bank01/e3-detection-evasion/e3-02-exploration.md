# e3-02 · 探索记录

对应题解：[`e3-02.md`](./e3-02.md)

## 目标与起点

`GET /` 把通道格式写完整了：读 `/target-data`，按片做 Base32，POST 到 `/dns`，再用 `/retrieve` 让服务端承认收齐。文档长度事先给的是 902。

## 第一阶段：一片就够

`/target-data` 的 `content` 长度是 902，`file` 指向 `/etc/secrets/classified_aurora.txt`。整段做 Base32、去掉 `=`，放进 `0-1-<chunk>.exfil.challenge.local`。`POST /dns` 返回 `decoded_len: 902`、`received_chunks: 1`、`total: 1`，并给了一个 `session`。

用这个 session 调 `/retrieve`，`complete` 是 true，`assembled_size` 是 902，响应里有 flag。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只读 `/target-data`，不回传 | 没有 flag | 文档接口的字段是 `content` / `file` / `note`，没有 `flag`。`note` 要求走 DNS 形状的通道并在服务端确认 |
| 担心单片超过 DNS 标签长度，准备拆成多片 | 单片已被接受 | `POST /dns` 的 `status` 是 `ok`，`decoded_len` 等于原文 902。这条接口只校验格式，不模拟 63 字节的 DNS 标签上限 |
| 漏掉 `session` 就去 `/retrieve` | 不需要试 | 只有一片时，第一次 `/dns` 已经返回 session，`/retrieve` 直接用它就 `complete: true` |

## 这次真正的收获

外带目标是服务端手里的那 902 字节，flag 不在文档里，在收齐之后的 `/retrieve`。查询名按 `<seq>-<total>-<base32>.exfil.challenge.local` 拼，Base32 去掉填充。这篇文档用一片就能还原。
