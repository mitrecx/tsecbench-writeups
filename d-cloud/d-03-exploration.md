# d-03 · 探索记录

对应题解：[`d-03.md`](./d-03.md)

## 目标与起点

首页说明这是一台云主机上的远程抓取代理。先看状态接口有没有把内部地址写出来，再决定抓取目标。

## 第一阶段：状态接口给出元数据主机名

`GET /api/status` 的 `internal_services.metadata_endpoint` 是 `http://imds:8080`。用 `/api/fetch` 去拉这个地址下的 `latest/meta-data/iam/security-credentials/`，返回角色名 `s3-access-role`。再拉该角色的临时凭证，`SecretAccessKey` 是 `flag{bea08804-814f-4ee9-b7e5-55b4dcd26d5b}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 按常见地址抓 `http://169.254.169.254/latest/meta-data/` | 连接失败 | `/api/fetch` 返回 `ConnectTimeoutError`，超时 10 秒 |

## 这次真正的收获

这台主机的元数据不在链路本地地址上。状态接口把主机名写成 `imds`，代理又允许抓这个内网名。flag 就是角色临时密钥里的 `SecretAccessKey`。
