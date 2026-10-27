# g-21 · 探索记录

对应题解：[`g-21.md`](./g-21.md)

## 目标与起点

题面只说这是一个 Go 文字冒险，走出迷宫就能看到 flag。附件 `rpgo.zip` 里是 1.4MB 的静态二进制，strip 过，没有源码。本机跑不了 Linux ELF，所以全程静态看 pclntab 和 `objdump`。

## 做了什么

1. Go 1.20+ pclntab 魔数 `0xfffffff1` 在文件偏移 `0xd7c40`。函数名表里有 `main.move`、`main.checkMove`、`main.win`、`main.decrypt`，入口分别在 `0x491bc0`、`0x491920`、`0x491540`、`0x4913c0`。
2. `checkMove` 用 `rax * 160 + rbx * 16` 索引 VMA `0x553820` 的 10×10 字符串表。格子是单字节：`#` 墙、`M` 怪、`W` 可挖的墙、`X` 陷阱、`P` 镐、`S` 玩家、`F` 终点，其余是空格。返回值非 0 才允许移动；`#` / `M` / `W` 返回 0。
3. `main` 的字符串比较（带 `ReadString` 留下的 `\n`）对应九个命令。长度小于 8 又不是 `go up` 会走到 `panicSlice`，因为编译器把 `go right` 的第 8 个字节检查提前了。`go down\n` 刚好 8 字节，所以带换行时 `down` / `left` / `mine` 都能过。
4. 地图左下角 `(6,3)` 的 `M` 挡住去右半边的路，生命值在 `0x54a220`，初值 3。先 `brace` 再 `fight` 三次，怪被清成空格，且不会被打向左边的 `X`。镐在 `(4,4)`，捡到之后到 `(7,7)` 用 `mine` 打掉右边的 `W`，再向右两步进 `F`。
5. `main.win` 的常量是 MD5 初始向量 `67452301 efcdab89 98badcfe 10325476`，随后 `hex.Encode` 和 `aes.NewCipher` + `NewCBCDecrypter`。密文 96 个 hex 字符在 `0x4bb8fe`。用日志的 MD5 hex 当 32 字节密钥、解码后的前 16 字节当 IV，后 32 字节解出 flag。最后一步 `right` 发生在 `win` 返回之前，不能算进日志；带上它就解不出这句明文。

## 走不通 / 走过的弯路

- `strings` 里的 `fight` 和 `You win!` 跟运行时字符串粘在一起，不能靠它还原命令表。判据是 `main` 里对 `go ` 之后 4 字节的立即数比较。
- 曾把解密当成 CTR。pclntab 对上的是 `crypto/cipher.NewCBCDecrypter`，CTR 解出来不是文本。
- 密钥不是 MD5 的 16 字节原始摘要。`win` 先把摘要编成小写 hex，再把这 32 个 ASCII 字节交给 `aes.NewCipher`，所以是 AES-256。用 16 字节原始摘要解出来是乱码。
