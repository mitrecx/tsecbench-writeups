# c-07 · 探索记录

对应题解：[`c-07.md`](./c-07.md)

## 目标与起点

题目只说这是远程登录服务。先看容器地址上的端口，再按服务自己的提示登录。

## 第一阶段：telnet 上的 root 登录

地址的端口是 23。横幅是 `Linux ... (pts/0)`，接着是 `login:`。用户名 `root`、口令 `root` 直接进入 shell，`id` 是 `uid=0(root)`。`/challenge/flag.txt` 的内容是 `flag{73b7b7d4-be64-41ae-bcd3-e576cb7f8a1f}`。平台接受。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 把横幅本身当成 flag | 没有 | 横幅只有内核版本和登录提示，没有 `flag{...}` |

## 这次真正的收获

这道题就是一个开着 telnet 的 root 口令。flag 在登录后的 `/challenge/flag.txt`，不在横幅里。
