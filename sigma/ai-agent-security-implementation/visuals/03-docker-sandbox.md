# 03. 基于 Docker 的沙箱执行环境构建

## 核心需求：安全地执行不受信任的代码

当大模型为你生成了一段 Python 代码（比如用于数据分析），这段代码是**不受信任**的。它可能包含恶意逻辑，或者只是单纯的死循环。
我们不能直接在宿主机（Host）上运行 `python script.py`，而是要把它塞进一个 Docker 容器里。

## TypeScript 实现：调用 Docker

在 Node.js 中，我们通常通过 `child_process` 模块来调用系统的 `docker` 命令，或者使用 `dockerode` 这样的 API 库。

### 方案 A：使用 `child_process.exec` (最简单直接)

```typescript
import { exec } from 'child_process';
import { promisify } from 'util';
import * as fs from 'fs/promises';
import * as path from 'path';

const execAsync = promisify(exec);

async function runPythonInDocker(code: string): Promise<string> {
  // 1. 将 LLM 生成的代码写入一个临时文件
  const tmpDir = path.resolve('./tmp');
  await fs.mkdir(tmpDir, { recursive: true });
  const scriptPath = path.join(tmpDir, 'script.py');
  await fs.writeFile(scriptPath, code);

  // 2. 构造 Docker run 命令
  // --rm: 运行完毕后立刻删除容器，保持环境干净
  // -v: 将临时目录挂载到容器内的 /app 目录（只读或读写）
  // --network none: (可选) 禁用容器的网络，防止代码向外发送数据
  // --memory 512m: (可选) 限制内存，防止 OOM 攻击
  const dockerCmd = `docker run --rm -v ${tmpDir}:/app --network none --memory 512m python:3.9-slim python /app/script.py`;

  try {
    // 3. 执行容器并获取输出
    const { stdout, stderr } = await execAsync(dockerCmd, { timeout: 10000 }); // 限制 10 秒超时
    if (stderr) {
      console.warn("Docker stderr:", stderr);
    }
    return stdout;
  } catch (error) {
    // 处理超时或执行错误
    return `Execution failed: ${error.message}`;
  } finally {
    // 4. 清理临时文件
    await fs.unlink(scriptPath).catch(() => {});
  }
}
```

## 进阶思考：Docker 沙箱的防御纵深

仅仅使用 `docker run` 并不绝对安全，专业的 Agent 平台（如 E2B, CodeSandbox）会在 Docker 层面做更极端的限制：
1. **Drop Capabilities**：使用 `--cap-drop=ALL` 剥夺容器内 root 用户的几乎所有 Linux 内核特权。
2. **无网络或代理网络**：`--network none`，或者强制所有流量走一个监控代理。
3. **资源配额 (Cgroups)**：限制 CPU 核心数、内存大小（`--memory`）、甚至磁盘写入速度，防止资源耗尽攻击（如 Fork 炸弹）。
4. **只读文件系统**：`--read-only`，只允许在特定的 `/tmp` 挂载点写入。
