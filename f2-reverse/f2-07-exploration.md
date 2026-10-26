# f2-07 · 探索记录

对应题解：[`f2-07.md`](./f2-07.md)

## 目标与起点

页面同时给了 ELF 和 `/check`。字符串里已经有 `/challenge/flag1.txt` 以及接受、拒绝两句提示。先用 `/check` 看什么样的序列号会让程序去读那个文件，再回头看二进制里那串异或数据是不是 flag。

## 第一阶段：末尾空格决定校验，文件内容才是 flag

`POST /check`，`key=Eube3`，结果页是 `Invalid license key.`。同一个值后面加一个空格，`key=Eube3 `，结果页变成 `License accepted.`，并且多出一行 `flag{2f61e2b0-66a6-438b-87d0-80e1c2876e32}`。`key=test` 同样是拒绝。

把文件里 `bOHNIbNIDQXb[SKbU` 附近的字节异或 `0x3d`，得到的可读尾巴是 `_rust_style_fnv_hash}`。这和 `/check` 打出来的 `flag{uuid}` 不是同一串。平台接受的是页面上的 uuid。

空格前后的 FNV-1a 不同：`Eube3 ` 是 `0xe868c44d`，`Eube3` 是 `0x1b7f4ebf`。程序里的乘子没有以立即数 `0x01000193` 出现，常量先和 `0x3d` 异或过。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `key=Eube3` | 校验失败 | 结果页只有 `Invalid license key.` |
| `key=test`、`key=123456` | 校验失败 | 同样一句拒绝 |
| 把异或 `0x3d` 得到的 `...rust_style_fnv_hash}` 当成 flag | 不是这一题的提交值 | `/check` 通过后打印的是 `flag{uuid}`，和这段诱饵不同 |
| `key=Eube3 ` | 校验通过并读出文件 | 结果页有 `License accepted.` 和 `flag{2f61e2b0-66a6-438b-87d0-80e1c2876e32}`，平台接受 |

## 这次真正的收获

序列号比的是哈希，所以末尾一个空格就足以变成另一个被接受的输入。通过之后程序读 `/challenge/flag1.txt`。二进制里能解出来的那句 FNV 口诀是诱饵。
