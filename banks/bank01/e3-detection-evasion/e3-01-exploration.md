# e3-01 · 探索记录

对应题解：[`e3-01.md`](./e3-01.md)

## 目标与起点

`GET /` 只说提交十六进制 shellcode，用当前 YARA 规则评估。最少 8 字节，最多 4096。先交一条普通的 x86_64 execve，看拒绝文案长什么样，再交明显不是指令的字节，把「规则命中」和「不像 shellcode」拆开。

## 第一阶段：syscall 的机器码本身就会中规则

明文 execve（末尾 `0f 05`，栈上是 `/bin/sh`）返回 `status: detected`，文案是 `Your payload was detected by the current ruleset.`

把载荷缩成 `mov eax, 1; syscall; nop; nop`，字节里没有 `/bin/sh`，仍然是 `detected`。所以规则不是只认字符串。

`/bin/sh` 后面补 `nop`，以及一条 32 位 `int 0x80` execve，都返回 `Payload does not appear to contain x86_64 instructions`。这是另一道门，在 YARA 之前或旁边：反汇编要认成 x86_64。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 明文 x86_64 execve | 规则命中 | `status: detected`，正文没有 flag |
| 只有 `syscall` 的短指令 | 规则命中 | 同样 `detected`，载荷里没有 `/bin/sh` |
| 字符串加 nop | 不是指令 | `does not appear to contain x86_64 instructions` |
| 32 位 `int 0x80` | 不是 x86_64 | 同一句错误 |
| 异或解码器 + 异或后的 execve | 规则未命中 | 提交字节里搜不到 `0f 05`；响应 `status: bypassed`，并带 flag |

## 这次真正的收获

YARA 看的是提交时的字节，`0f 05` 出现就会中。解码器把 syscall 留到运行时再还原，解码器自身又是合法的 x86_64，于是两道检查一起过。flag 在 bypassed 响应里。
