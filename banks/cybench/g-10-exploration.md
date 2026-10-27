# g-10 · 探索记录

对应题解：[`g-10.md`](./g-10.md)

## 目标与起点

菜单在 1337。80 的 zip 给出 `sandbox.py` 和 `app.py`，注释写着 Python 3.8。查看会员时会 `print(module, name)`，失败信息可以直接看到是哪一个 `find_class` 被拒绝。

## 过程

1. 过滤条件是 `name.startswith("__builtins__.")` 或 `name.startswith("random.")`，不是 `startswith("__builtins__")`。所以名字本身叫 `__builtins__` 或 `random` 能过，只有再往下点一层才被挡。协议 4 又允许名字里带点。
2. 直接 `app.random._os.system` 和 `__main__.random._os` 都被这个前缀拒绝，异常信息是空的 `UnpicklingError`。
3. `__main__.Phreaks.__init__.__globals__` 能拿到模块全局字典（查看时抱怨 dict 没有 `display_info`，说明对象已经还原出来了）。`__main__.__builtins__` 是 builtins 模块。函数对象上的 `__builtins__` 属性在这台服务上不存在。
4. 字典的 `__getitem__` 以 `Phreaks.__init__.__globals__.__getitem__` 取到，前缀检查过得去。`Phreaks.__getattribute__` 是 `object.__getattribute__`。
5. 栈序按 pickle 对 `getattr(obj, name)` 的习惯：先放可调用对象，再放参数。先 `REDUCE` 出 builtins 模块，再 `__getattribute__(module, "eval")`，再 `eval` 读文件。
6. 第一次 `find` 把 `/challenge/flag.txt` 列出来。第二次只打印这个文件。提交通过。

## 排除项

没有把 `os.system` 编成 `GLOBAL os system`，模块 `os` 不在白名单里。判据是服务端打印的 `FIND` 行：被拒的名字以 `random.` 开头；能还原的名字不以这两个前缀开头，并且 `_getattribute` 走得通。
