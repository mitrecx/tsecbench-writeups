# 题库索引

这里登记所有已接触过的 TSecBench 题库。**一套题库一个目录**（`banks/<slug>/`），
因为不同题库的 `unique_code` 会重合，混放会互相覆盖。

## 现有题库

| slug | 题数 | 总分 | 分类前缀 | 首次记录 | 状态 | 总览 |
| --- | ---: | ---: | --- | --- | --- | --- |
| `bank01` | 63 | 23000 | `a` `b` `c` `d` `e1` `e2` `e3` `f1` `f2` | 2026-08 | 63/63 通关 | [banks/bank01/README.md](./banks/bank01/README.md) |
<<<<<<< HEAD
| `xbow-validation` | 104 | 28300 | `xben-*`（无分类前缀） | 2026-10-27 | 70/104 进行中 | [banks/xbow-validation/README.md](./banks/xbow-validation/README.md) |
| `cybench` | 40 | 20800 | `g-*`（无分类前缀） | 2026-10-27 | 24/40 进行中 | [banks/cybench/README.md](./banks/cybench/README.md) |
=======
| `xbow-validation` | 104 | 28300 | `xben-*`（无分类前缀） | 2026-10-27 | 72/104 进行中 | [banks/xbow-validation/README.md](./banks/xbow-validation/README.md) |
>>>>>>> 7c7ee2d (xben-010-24: solve JSFuck char blacklist (71/104))

## 判别指纹

新 token 拿到后，先 `GET /openapi/v1/challenges` 取三项：**题数**、**总分**、
**`unique_code` 排序后的集合**。三项都与某库一致，才判为同一题库。

| slug | 题数 | 总分 | unique_code 形态 |
| --- | ---: | ---: | --- |
| `bank01` | 63 | 23000 | `a-01`–`a-18`、`b-01`–`b-03`、`c-01`–`c-09`、`d-01`–`d-06`、`e1-01`–`e1-06`、`e2-01`–`e2-04`、`e3-01`–`e3-04`、`f1-01`–`f1-05`、`f2-01`–`f2-08` |
| `xbow-validation` | 104 | 28300 | `xben-001-24`–`xben-104-24`（补零三位 + `-24`）；全部 `level=1`、`flag_count=1` |
| `cybench` | 40 | 20800 | `g-01`–`g-40`；全部 `level=1`、`flag_count=1`。easy 9 题 / 2200，medium 20 题 / 9600，hard 11 题 / 9000。flag 格式不统一，题面会写明要保留 `HTB{}`、`gctf{}`、`SEKAI{}` 或 `hkcert22{}` |

```bash
curl --noproxy '*' -sS -H "BENCHMARK_TOKEN: $BENCHMARK_TOKEN" \
  -H 'Accept: application/json' https://tsecbench.zc.tencent.com/openapi/v1/challenges \
| python3 -c 'import json,sys; cs=json.load(sys.stdin); \
print("count", len(cs)); \
print("score", sum(c["total_score"] for c in cs)); \
print("codes", " ".join(sorted(c["unique_code"] for c in cs)))'
```

## 判定规则

| 情况 | 处置 |
| --- | --- |
| 题数、总分、`unique_code` 集合三者全同 | **同一题库的另一个场次**。续写该库，不要新建目录——flag 按实例重新生成，题解和探索记录都可以直接复用（但**提交前必须先 `start`**，见根 README 第 2 节） |
| 任一项不同 | **新题库**。新建 `banks/bankNN/`，复制 `bank01/` 的目录骨架与 README 模板，然后在本文件两张表里各加一行 |
| 题数总分相同但 `unique_code` 不同 | 仍判为新题库。编号重合只是巧合，按集合判 |

## 新增题库的步骤

1. `mkdir -p banks/bankNN`，按新库实际出现的分类前缀建子目录（不必凑齐 9 个）。
2. 写 `banks/bankNN/README.md`，从 `banks/bank01/README.md` 复制顶部元数据表 + 进度表骨架，
   填本库的题数、总分、分类、指纹。
3. 在上面的「现有题库」与「判别指纹」两张表各加一行。
4. 单题文件按 `<code>.md` + `<code>-exploration.md` 命名，约定见根 README 第 4 节。
