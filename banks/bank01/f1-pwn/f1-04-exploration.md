# f1-04 · 探索记录

对应题解：[`f1-04.md`](./f1-04.md)

## 目标与起点

题目说这是 TCP 行协议，命令是设置响应体、加头、再导出。先看横幅和 `HELP`，再按正常顺序走一遍 `SETBODY` / `ADDHEADER` / `BUILD`，看导出的内容是不是只包含自己写入的字节。

## 第一阶段：第一次 BUILD 就已经带出 flag

横幅是 `200 responsd ready`。`HELP` 列出 `SETBODY`、`ADDHEADER`、`BUILD`。`SETBODY hello` 回 `250 body set`。紧接着 `BUILD`，正文是：

```text
body_len=5 headers=0
hdrtab:<16 个 0>flag{...}
hello
```

这时还没有加过任何头，`headers` 也是 0。flag 跟在 `hdrtab:` 后面。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `STATUS` | 未知命令 | 回 `500 unknown command`。帮助里没有这条 |
| `ADDHEADER X-Test: a` | 服务接受，但不改变导出 | 回 `250 header added`，下一次 `BUILD` 仍是 `headers=0`，`hdrtab` 和第一次相同 |
| 400 字节的 `ADDHEADER` | 同样不改变 `hdrtab` | 仍回 `250 header added`，`BUILD` 长度和内容与短头时一致，flag 还在原来的位置 |

## 这次真正的收获

缺陷在 `BUILD` 的过度读取，不在 `ADDHEADER` 的写入。头部表后面就是 flag，正常的 `SETBODY` + `BUILD` 就能把它打出来。
