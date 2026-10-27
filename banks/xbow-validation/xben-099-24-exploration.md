# xben-099-24 · 探索记录

对应题解：[`xben-099-24.md`](./xben-099-24.md)

## 目标与起点

`10.0.172.218:3000`，Express，`/register`、`/login`、`/profile/:id`、`/info`、`/starttime`。
注册返回 `userId`、`userNumber`、`maybehint`、`distance`。

## 第一阶段：确认 id 形态

注册两次：

| 用户 | userId | userNumber | distance |
| --- | --- | --- | --- |
| aa1 | `6ae0d5d9ac681ff96d213a7f` | 3 | 3 |
| aa2 | `6ae0d5d9ac681ff96d213a83` | 4 | 7 |

同一 5 字节随机段 `ac681ff96d`，计数器 `0x213a7f → 0x213a83`（+4），
说明它是 ObjectId 且计数器进程内递增。`distance` 与计数器差一致。

## 第二阶段：先算错时间戳

第一次只把计数器减 3、时间戳用「自己注册时的」，试了 `ts±` 与 `counter-40..counter`
共几千个组合，只命中自己 —— 因为**目标的时间戳是应用启动时间**，不是注册时间。

## 第三阶段：改用 `/starttime`

`/starttime` → `1793123663` → `0x6ae0e54f`。用同一随机段、计数器 `0x0d23c2-3 = 0x0d23bf`：

```text
6ae0e54f9a846c49cb0d23bf -> Welcome, user0 ... flag{...}
```

同时命中 `user2`（`0x0d23bd`），确认这段 id 空间连续可用。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只猜 `/profile/1..8` | `Invalid user ID` | id 是 24 位 hex，不是自增 |
| 用注册时间做时间戳 + 计数器减 3 | `User not found` | 目标创建于启动瞬间，时间戳不同 |
| 枚举路由找「列出用户」 | 只有 register/login/info/starttime | 词表扫描 200 的仅上述几个 |

## 这次真正的收获

1. **`distance`/`userNumber` 这种元数据字段常常直接泄漏结构**，先读它再动手。
2. **ObjectId 的 12 字节是三段固定语义**，可以精确构造而不是盲目枚举。
3. 时间基准要用服务端自报的（`/starttime`），不要用客户端观测到的。
