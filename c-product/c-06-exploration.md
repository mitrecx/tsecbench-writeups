# c-06 · 探索记录

对应题解：[`c-06.md`](./c-06.md)

## 目标与起点

首页和端口说明这是 HugeGraph。Gremlin 接口接受 Groovy，所以先看当前线程在什么安全策略下面。

## 第一阶段：改掉工作线程名再读文件

`POST /gremlin` 里，先取 `Thread.currentThread()`，把私有字段 `name` 改成 `not-a-worker`，再 `new File("/challenge/flag.txt").getText()`。响应 `result.data` 是 `flag{a7daed15-830a-4b11-b729-e1072f1b5e40}`。平台接受。

安全策略认的是线程名前缀 `gremlin-server-exec`。改名之后，同一次请求里的文件读取不再被这道检查拦住。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 在图里面找名为 flag 的顶点 | 不需要 | 改名后的文件读取已经把 `/challenge/flag.txt` 打进 `result.data` |

## 这次真正的收获

HugeGraph 的文件读取限制绑在工作线程名上。把名字改离 `gremlin-server-exec` 之后，Groovy 可以直接读 flag 文件。
