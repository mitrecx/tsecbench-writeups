# e3-04 · 探索记录

对应题解：[`e3-04.md`](./e3-04.md)

## 目标与起点

`GET /` 要求提交有真实程序结构的代码，5 条启发式，bypass 分数至少 60%。先交一段只有函数和赋值的代码，确认预检和评分是不是两条分开的响应。

## 第一阶段：经典注入名会打满 5 条

`int main(){return 0;}` 和只有 `helper` 的 C 函数返回 200，但正文是 `does not appear to be executable code with real program structure`。带 `import`、函数和赋值的 Python，如果里面没有注入相关的名字，服务直接 HTTP 500，没有 JSON。

一旦出现 `OpenProcess`、`VirtualAllocEx`、`WriteProcessMemory`、`CreateRemoteThread`、`ptrace` 或 `dlopen`，响应变成 JSON：`triggered_rules` 是 5，`bypass_score` 是 0.0，`status` 是 `detected`。把 `OpenProcess` 拆成 `"Open"+"Process"` 再 `getattr`，又回到 HTTP 500。名字不在词表里时，评分路径进不去。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 空 `code` | 400 | `No code provided` |
| 只有 `int main` 或一个 C 函数 | 预检拒绝 | `status: rejected`，没有分数 |
| 有函数和赋值、没有注入名 | 服务 500 | 正文是 Internal Server Error，不是 rejected |
| `OpenProcess` 等 Windows API，以及 `ptrace`、`LD_PRELOAD` | 5/5 命中 | `bypass_score` 0.0 |
| 把 API 名拆开 | 500 | 和「完全没有这些名字」同一类错误 |
| 单独的 `memfd_create` | 3/5 命中 | 分数 0.4，`detected`，没有 flag |
| 单独的字符串 `/proc/self/mem` | 3/5 命中 | 同样 0.4 |
| `process_vm_writev` | 4/5 命中 | 分数 0.2 |
| 大小写变化 `MEMFD_CREATE` | 仍然 3/5 | 匹配不分大小写 |
| `/proc/%d/mem` 这种拼出来的路径 | 500 | 字面量 `/proc/self/mem` 不在源码里，评分路径没有进入 |
| 同一个函数里同时写出 `memfd_create` 和 `/proc/self/mem` | 2/5 命中 | `bypass_score` 0.6，`status: bypassed`，响应带 flag。消息写的是 3/5 通过 |

把这两个调用真的写进 `launch`（匿名文件写入，再对 `/proc/self/mem` 做 `seek`/`write`），分数不变，flag 相同。

## 这次真正的收获

5 条规则不是「每出现一个 API 加一条」。一组高信号名字会一次打满。`memfd_create` 和 `/proc/self/mem` 各自留下 2 条未命中，合在同一个函数里变成只命中 2 条，刚好过 60% 的线。flag 由 `/check` 在 `bypassed` 时返回。
