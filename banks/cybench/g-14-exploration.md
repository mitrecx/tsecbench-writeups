# g-14 · 探索记录

对应题解：[`g-14.md`](./g-14.md)

## 目标与起点

题面说运行程序没有输出，flag 被拆进 flecs 世界。附件只有 `flecks` 这一个 ELF，本机不能执行 Linux 动态链接的程序，所以只做静态还原。

## 做了什么

1. `strings` 能看到 `Explorer `、` has found a flag part` 和人名 `Armanix`、`Ayaan`、`Brensom`、`Jaggex`、`Zavis`、`Austim`、`Randis`，没有完整的 `HTB{`。
2. `main` 在 `0x3af0`。移动回调在 `0x52e0`：把人的坐标加上 `rand_range`，再和每个 `FlagPart` 的坐标做 `cvttsd2si` 后比较。重合才把碎片字符写进缓冲并打印。`srand(time)` 加上随机游走，解释了「跑起来什么都没有」。
3. 碎片不是运行时算出来的。`0x43b4` 从 `r12` 读一个 `word` 写进组件。`r12` 来自 `completed.0+0x18` 指向的数组，长度在 `completed.0+0x10`。
4. 这两个 BSS 字段只在 `_Z1_v`（`0xc3fc0`）里赋值：长度立即数 `0x1d`，指针是 `malloc(0x3a)`。内容来自四段 rodata，不是连续的 `0x3a` 字节。最后 5 个 `(index, char)` 在 `0xc6dc8` 的 8 字节和 `0xc6e30` 的一个 word 里。
5. 按 index 排好就是 `HTB{br1ng_th3_p4rt5_t0g3th3r}`。提交被接受。

## 走不通 / 走过的弯路

- 把 `0xc6e00` 起连续 `0x3a` 字节都当成碎片。第 25 个 word 开始读到了类型名里的 `E`、`O`、`T`，拼出来的字符串在中间缺字、末尾是垃圾。判据是 `_Z1_v` 的拷贝源地址：偏移 `0x30` 的 8 字节来自 `0xc6dc8`，不是 `0xc6e30` 前面的字节。
- 想靠多跑几次撞坐标。动态链接的 Linux ELF 在这台机器上跑不起来，而且种子是 `time(NULL)`，不是还原碎片所必需的。
