# e2-04 · 探索记录

对应题解：[`e2-04.md`](./e2-04.md)

## 目标与起点

`GET /` 标明是 vm2，不是上一题那个裸的 `vm.runInNewContext`。先把 e2-03 的构造器链原样打一次，看 vm2 拦在哪一层。

## 第一阶段：构造器和调用帧都不是出口

`console.log(1)` 正常。`typeof process`、`typeof require` 都是 `undefined`，`Buffer`、`Function`、`eval`、`Proxy` 都是 `function`。

`this.constructor.constructor("return process")()` 报 `process is not defined`。同一句从 `Buffer.from("").constructor.constructor` 出发，还是这句错误。vm2 把这条链上的 `Function` 换成了沙箱里的版本。

异常栈是字符串，里面有 `at vm.js:2:13` 和 `/app/node_modules/vm2/lib/vm.js`。把 `Error.prepareStackTrace` 改成返回 `frames` 之后，`e.stack` 的长度是 10，类型是 object。第一帧的原型上有 `getThis`、`getFunction`、`getFileName`。`getFileName()` 是 `vm.js`，`getFunction()` 和 `getThis()` 都是 `undefined`。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| `this.constructor.constructor("return process")()` | 仍在沙箱里 | `ReferenceError: process is not defined` |
| 从 `Buffer` 的构造器再走一次 | 同样失败 | 输出是 `E process is not defined` |
| `CallSite.getThis` / `getFunction` | 没有宿主对象 | 两者都是 `undefined`，文件名只有 `vm.js` |
| `err.name.toString` 换成 Proxy，在 `apply` 里用 `args.constructor.constructor` | 读到文件 | `console` 输出就是 `flag{...}`，平台接受 |

## 这次真正的收获

vm2 拦的是脚本里直接拿到的 `Function`，拦不住宿主在格式化异常时对 `name.toString` 的调用。那个调用的参数对象来自宿主，它的 `constructor.constructor` 可以 `return process`，再用 `fs.readFileSync` 读 `/challenge/flag1.txt`。文件内容是用 `throw` 送出来的，因为陷阱本身的返回值不会出现在页面上。
