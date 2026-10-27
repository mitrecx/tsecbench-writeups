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
