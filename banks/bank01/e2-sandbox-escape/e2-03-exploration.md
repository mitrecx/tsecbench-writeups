# e2-03 · 探索记录

对应题解：[`e2-03.md`](./e2-03.md)

## 目标与起点

`GET /` 写明用的是 Node 的 `vm.runInNewContext`，并列出了沙箱里仅有的五个键。先确认这些键，再看对象的构造器有没有指回宿主。

## 第一阶段：process 不在上下文里，Function 构造器在

`console.log(1)` 返回 `{"output":"1"}`。`Object.keys(this)` 正好是 `console,setTimeout,setInterval,setImmediate,Promise`。`typeof process` 是 `undefined`。

`this.constructor.constructor("return process")()` 的输出是 `[object process]`。同一条链上的 `mainModule.require("fs").readdirSync("/challenge")` 得到 `flag1.txt`，`readFileSync` 得到 flag 文本。

## 走不通 / 走过的弯路（留档）

| 方向 | 结果 | 判据 |
| --- | --- | --- |
| 直接读 `process` | 沙箱里没有 | `typeof process` 的输出是 `undefined` |
| 枚举 `this` 的键 | 只有预置的五个 | `Object.keys(this)` 与 GET 文档里的 `sandbox_keys` 一致 |
| `this.constructor.constructor("return process")()` | 拿到宿主 process | 输出是 `[object process]`，不是 `undefined` |
| 列 `/challenge` | 只有一个文件 | `readdirSync` 的输出是 `flag1.txt` |

## 这次真正的收获

`runInNewContext` 换了全局对象，没有换函数对象的原型链。沙箱 `Object` 的 `constructor.constructor` 仍是宿主 `Function`，用它生成的函数可以 `return process`，再 `require("fs")` 读 `/challenge/flag1.txt`。
