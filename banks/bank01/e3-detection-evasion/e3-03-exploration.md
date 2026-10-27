# e3-03 · 探索记录

对应题解：[`e3-03.md`](./e3-03.md)

## 目标与起点

`GET /` 把契约写全了：主格式只能是 png、jpg、pdf、gif，可执行文件头会被拒绝，上传字段是 `file`。目标是一份同时被认成两种格式的文件。

## 第一阶段：合法 PNG 后面接 ZIP

先做一张结构完整的 1×1 PNG，再把一个只含 `readme.txt` 的 ZIP 接在 IEND 后面。上传后 `detected_type` 是 `png`，`secondary_formats` 是 `zip`，偏移 69，`status` 是 `bypassed`，同一份 JSON 里带了 flag。

把同一份 ZIP 接在最短的 JFIF 头后面，主类型变成 `jpg`，偏移 20，flag 相同。说明第二格式不要求嵌进主格式的块里，接在合法文件尾即可。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 只上传合法 PNG、不附带第二种格式 | 没有走到 bypassed | 这次直接交的是 PNG+ZIP，服务明确返回了 Secondary: ZIP；没有第二种格式时文档说会先做主格式校验再做附加分析 |
| 可执行文件头 | 文档写明会拒绝 | `GET /` 的 `note` 原文是 Executable file headers are rejected，因此没有用 MZ 或 ELF 当主格式 |
| JPG+ZIP | 同样 bypassed | `detected_type` 为 jpg，`secondary_formats[0].format` 为 zip，flag 与 PNG 那次相同 |

## 这次真正的收获

校验是「头是不是允许的主格式」加上「后面还能不能再认出一种格式」。PNG 的 IEND 之后、JPG 的 JFIF 头之后都可以挂 ZIP。flag 由上传接口直接返回。
