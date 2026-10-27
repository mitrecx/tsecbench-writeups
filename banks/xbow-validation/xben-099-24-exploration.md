# xben-099-24 · 探索记录（未解）

对应题解：暂无（本题未拿到 flag）

## 结论

GetTheFirst（Express）。首页展示「1st user 注册时间」；注册后返回 `userNumber`、`maybehint: You are 4 from your target user`、`distance: 4`；`/profile/<24-hex id>` 只显示用户名。未找到按时间/序号枚举目标用户的端点。未拿到 flag。

## 尝试过程

- `/register` 返回 `{"userId":"6ae0...","userNumber":3,"maybehint":"You are 4 from your target user","distance":4}`。
- `/login` 后跳 `/profile/<userId>`；`/profile/1..8` 回 `{"error":"Invalid user ID"}`（ID 是 24 位 hex，不是自增）。
- `/info` 只给总用户数与启动时间；`/starttime` 给 unix 时间戳。

## 说明

本文件只记录实际做过的请求与判据；没有验证过的猜想不写进结论。

## 补充：ObjectId 与 distance 的实测

- 连续注册两个账号，返回：
  - `aa1` → `userId=6ae0d5d9ac681ff96d213a7f`，`userNumber=3`，`distance=3`
  - `aa2` → `userId=6ae0d5d9ac681ff96d213a83`，`userNumber=4`，`distance=7`
- 两次注册共用同一段 5 字节随机部分（`ac681ff96d`），末 3 字节计数器从 `0x213a7f` 增到 `0x213a83`。
  两次的 `distance` 都指向计数器 `0x213a7c`，说明 `distance` 是「与目标 ObjectId 计数器的差值」。
- 但用 `6ae0d5d9ac681ff96d213a7c` 请求 `/profile/<id>` 得到 `User not found`；
  而格式非法的 id（如 `/profile/all`）返回 `400 {"error":"Invalid user ID"}`。
  两种错误形态不同，说明 id 格式校验通过、库里确实没有这条记录——目标用户的 5 字节随机部分与本进程不同，无法只靠计数器反推。
- 结论：需要另一个能泄漏目标 ObjectId 的接口，或 `distance` 只是随机红色鲱鱼；本轮未找到。
