# 04. 基于 Node.js vm / WASM 的轻量级隔离

## Docker 的痛点：太重了

虽然 Docker 隔离性很好，但如果我们只是想让大模型执行一段简单的 `1 + 1` 或者处理一下 JSON 字符串，每次都启动一个 Docker 容器（即使是预热好的）也会带来几百毫秒到几秒的延迟，并且消耗大量内存。

对于高频、轻量级的代码执行，我们需要更快的沙箱。

## 方案 A：Node.js 内置的 `vm` 模块 (轻量，但有逃逸风险)

Node.js 提供了一个内置的 `vm` 模块，可以在当前进程中创建一个 V8 虚拟机的上下文。

```typescript
import * as vm from 'vm';

function runUntrustedJS(code: string) {
  // 1. 创建一个完全干净的上下文（没有 require，没有 fs，没有 process）
  const sandbox = {
    console: { log: (...args: any[]) => { /* 收集日志 */ } },
    Math: Math,
    // 只能传入你明确允许的变量
  };
  
  vm.createContext(sandbox);

  try {
    // 2. 在沙箱中执行代码
    // timeout: 1000 防止死循环 (while(true))
    const result = vm.runInContext(code, sandbox, { timeout: 1000 });
    return result;
  } catch (err) {
    return `Execution error: ${err.message}`;
  }
}
```

**⚠️ 致命缺陷**：Node.js 的 `vm` 模块**不是为了安全隔离设计的**。黑客可以通过原型链污染（Prototype Pollution）或 `this.constructor.constructor('return process')()` 逃逸出沙箱，拿到宿主机的 `process` 对象，进而执行 `fs.rmSync`。因此，`vm` 只能防君子，不能防小人。

## 方案 B：WASM (WebAssembly) 运行时 (轻量且安全)

目前业界最前沿的轻量级沙箱方案是 **WASM**。
我们可以把 Python 解释器（如 Pyodide）或者 QuickJS 编译成 WASM 字节码，然后在 Node.js 中运行。

**WASM 的安全优势：**
1. **内存隔离**：WASM 运行在一个线性的、完全隔离的内存空间中，它绝对无法读取宿主机的内存。
2. **能力默认拒绝 (Default Deny)**：WASM 代码默认没有任何系统权限（不能读文件、不能发网络请求）。如果它需要这些能力，必须通过 WASI (WebAssembly System Interface) 显式地由宿主机注入。
3. **启动极快**：毫秒级启动，几乎没有冷启动延迟。

### 架构对比

| 特性 | Docker / Firecracker | Node.js `vm` | WASM (Wasmtime / Pyodide) |
|------|----------------------|--------------|---------------------------|
| **启动速度** | 慢 (数百毫秒~秒级) | 极快 (微秒级) | 极快 (毫秒级) |
| **内存消耗** | 大 (几十~几百MB) | 极小 | 小 (几MB) |
| **安全性** | 高 (OS级别隔离) | 低 (容易逃逸) | 极高 (内存与指令级隔离) |
| **适用场景** | 运行复杂的系统命令、构建项目 | 仅限完全信任的内部逻辑 | 运行大模型生成的纯计算/数据处理代码 |
