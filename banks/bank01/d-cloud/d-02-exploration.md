# d-02 · 探索记录

对应题解：[`d-02.md`](./d-02.md)

## 目标与起点

管理门户没有登录页。先找函数清单，再看每份配置是否把密钥放进环境变量。

## 第一阶段：未授权读配置

`GET /api/functions` 返回三个函数名。`/api/functions/order-service/config` 的环境变量里，`DATABASE_PASSWORD` 是 `flag{7ee00bfc-df15-4788-869e-7425e54e666f}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 猜一批没出现在列表里的函数名 | 不需要 | 列表接口已经给出全部三个名字 |
| `data-processor` 和 `notification-handler` 的环境变量 | 没有 flag | 前者是队列和批大小，后者是 SMTP 和 Slack webhook |

## 这次真正的收获

函数配置接口没有鉴权。flag 只放在 `order-service` 的数据库口令这个环境变量里。
