# g-29 · 探索记录

对应题解：[`g-29.md`](./g-29.md)

## 目标与起点

题面说 flag 分三段藏在流量里。文件魔数是 `0A0D0D0A`，是 pcapng，不是经典 pcap。`tcpdump -A` 能看文本，但要重组 TCP 得自己解析 Enhanced Packet Block。

## 过程

1. 先看明文。61616 是 ActiveMQ 5.18.2，后面跟着一次 Spring XML 的 XXE/反序列化式投递：`ProcessBuilder` 拉 `aQ4caZ.exe`。8080 的 XML 和 8000 的 exe 里都没有直接的 `HTB{`。
2. exe 是 EZRAT。用户字符串里有服务器 `10.10.10.21`、口令样子的 `VYAemVeO3zUDTL6N62kVA`，分隔符是 `§`。`.text` 填充区还有 ASCII `Very_S3cr3t_S`，一开始当成 flag 碎片，后来对上了 IL。
3. `Encrypt` 的 RVA 是 `0x2104`。IL 先取 `_encryptKey`，再 `newarr 13` + `InitializeArray`。唯一的 FieldRVA 指向那 13 个字节，所以盐是 `Very_S3cr3t_S`，口令是 `VYAem...`。`GetBytes(32)` 然后 `GetBytes(16)`。
4. 用这个 AES-256-CBC 解 1234 端口。控制端命令里有：
   - `echo ssh-rsa ... HTB{c0mmun1c4710n5 >> authorized_keys`。`>>` 是重定向，不是 flag 的一部分。
   - `type credentials.txt` 的回包写着 `2nd flag part: _h45_b33n_r357`。
5. 同一条流后面还有一段没加密的 `powershell.exe -encoded`。UTF-16LE 解开后，注释写 “3th flag part”，任务名是 `0r3d_1n_7h3_h34dqu4r73r5}`。
6. 三段连起来读成 “communications has been restored in the headquarters”。提交通过。

## 排除项

- 没有把 `Very_S3cr3t_S` 或 bean id（`WHgLtpJX`、`fgCggZBK`）当成 flag 段。它们对不上 “第几段” 的标注，而且盐的用途在 IL 里是确定的。
- 服务端回包没有 `§`，是多段 Base64 直接接在一起。按 `=` 填充切开再解密，第一段 `infoback` 的 PKCS7 填充（`0x0f`）能对上，说明切分没把两段密文焊死。

## 判据

三段都有独立出处：SSH 注释、`credentials.txt` 的 “2nd flag part”、计划任务注释的 “3th flag part”。拼出来的字符串以 `HTB{` 开头、以 `}` 结尾，并且被平台接受。
