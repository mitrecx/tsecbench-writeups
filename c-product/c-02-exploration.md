# c-02 · 探索记录

对应题解：[`c-02.md`](./c-02.md)

## 目标与起点

题目只说对 AI 推理服务做安全测试。容器地址上的 8188 在服务起来之后返回 ComfyUI 页面。`GET /system_stats` 写明版本是 `0.9.2`，前端是 `1.36.14`，启动参数是 `main.py --listen 0.0.0.0 --cpu`。同进程还加载了 ComfyUI-Manager `V3.37`，以及 `/ComfyUI/custom_nodes/websocket_image_save.py`。

这次窗口里这份服务中途不再监听。下面除了最后一条节点名单，都来自那一份已经关掉的实例。新实例只重新确认了自定义节点，flag 还没有拿到。

## 第一阶段：文件接口和数据节点

`/view` 不接受直接指向 `/challenge/flag.txt` 的文件名。带 `../` 的 `subfolder` 是 403，把点号写成 `....//` 之后变成 404，响应体是空的。

`LoadTrainingDataset` 的 `folder_name` 可以写成 `/challenge`。把它接到 `SaveLatent` 上之后，执行错误是 `No shard files found in /challenge`。`LoadImageTextDataSetFromFolder` 的 `folder` 不是自由字符串，下拉框里只有 `3d`；选 `3d` 时错误是 `No valid images found in input`。`Preview3D` 接受路径 `/challenge/flag.txt`，输出里只把这个路径原样放进 `result`，没有文件内容。

## 第二阶段：用户目录和 Manager

`GET /api/userdata` 必须带 `dir`。`dir=..` 和 `dir=/challenge` 都是 `Invalid directory`。`dir=workflows/..` 可以列出用户目录里的 `ComfyUI-Manager/config.ini`。用百分号编码的路径去读 `workflows/../../../challenge/flag.txt`，响应是 403，正文为空。

读出来的 `config.ini` 里，`security_level` 是 `normal`，`network_mode` 是 `public`。通过 userdata 把 `security_level` 改成 `weak` 之后，当时的进程仍然按旧配置拒绝 `POST /api/customnode/install/pip`，响应是 403。`GET /api/manager/reboot` 会让进程重新拉起，第一次重启后 8188 很快恢复。随后把一次本地安装放进启动脚本再重启，8188 在大约 40 秒内没有重新监听。那份实例被 close，平台重新 start 了一份。新实例的 `object_info` 里，自定义节点仍然只有 `SaveImageWebsocket` 和 ComfyUI-Manager 自带的节点。flag 没有出现在这些响应里。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/view` 用 `../` 或绝对路径读 flag | 读不到 | `filename` 带 `../` 是 400；`subfolder=/challenge` 是 403；响应体为空 |
| `LoadTrainingDataset` 指向 `/challenge` | 没有文件内容 | 执行错误是 `No shard files found in /challenge` |
| `LoadImageTextDataSetFromFolder` | 不能选挑战目录 | `folder: '/challenge' not in ['3d']`；`3d` 的错误是 `No valid images found in input` |
| `Preview3D` 的 `model_file` 写成 flag 路径 | 只回显路径 | `outputs` 是 `["/challenge/flag.txt", null, null]` |
| userdata 直接跳出用户目录 | 被拒绝 | `dir=..` 与编码后的 `../../../challenge/flag.txt` 都是 403 |
| 改 `security_level` 后立刻装包 | 仍被拒绝 | 配置文件已经写成 `weak`，`/api/customnode/install/pip` 仍是 403 |
| 把安装放进启动脚本再重启 | 服务没有回来 | 重启请求得到空响应；之后约 40 秒内 8188 连接失败 |

## 这次真正的收获

这是 ComfyUI 0.9.2，flag 不在页面和 `/view` 允许的目录里。用户目录可以读到 Manager 配置，但按当时进程里的安全级别，安装接口不接受额外的包。把安装推迟到重启之后，进程没有再把 8188 打开。题解仍然待补充。
