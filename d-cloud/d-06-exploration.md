# d-06 · 探索记录

对应题解：[`d-06.md`](./d-06.md)

## 目标与起点

控制台要先注册。注册后的仪表盘直接写出三条线索：JWKS 还能从 `/files/` 提供、制品桶里有 Terraform 状态、恢复桶只给 root。先把这三条按顺序走完。

## 第一阶段：伪造 JWKS 换制品桶密钥

上传的对象出现在 `/files/shared-assets/users/1/jwks-hs.json`，不带登录也能读。JWT 头里的 `jku` 指到这个地址，`kid` 是 `1`，对称密钥是自己选的 `fed-secret-key`。

载荷 `group=ops` 时，`/api/sts/exchange` 返回 `audit-svc` / `AudSvc_P7j3n!`，策略是 `infra-artifacts-readonly`。`root`、`admin`、`auditor`、`operator` 都返回 `unknown federation group`。

## 第二阶段：状态文件里的 root 口令

用 `audit-svc` 列出 `infra-artifacts`，只有 `runbooks/storage-ops.txt` 和 `terraform/prod.tfstate`。运行手册写明 root 引导材料还在 Terraform 状态里。状态文件的 `data_json` 是 `cvault-admin` / `CVault_Adm1n#2026!`。

这组密钥能列出 `secrets-vault`。唯一对象是 `final/flag.txt`，正文是 `flag{93b4d1ad-4d2b-4c04-8eae-6153791070fc}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 把联邦组写成 `root` 或 `admin` | 交换失败 | 响应是 `unknown federation group` |
| 用 `audit-svc` 直接读 `secrets-vault` | 读不到 | 交换结果的 `bucket_scope` 只有 `infra-artifacts` |
| 在 `secrets-vault` 里读 `flag.txt` | 对象不存在 | MinIO 返回 `NoSuchKey`；列表里只有 `final/flag.txt` |

## 这次真正的收获

普通用户能把自己的 JWKS 放到公开路径。联邦交换只认 `ops`，换到的密钥只能读制品桶。root 口令在 Terraform 状态里，flag 在 `secrets-vault/final/flag.txt`。
