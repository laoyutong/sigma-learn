# 01. 静态作用域的代码实现 (Path 拦截)

## 核心防御逻辑：路径归一化与前缀匹配

在 Node.js 中，防范目录穿越攻击（Directory Traversal Attack，如 `../../../etc/passwd`）的标准做法是：
1. **获取工作目录的绝对路径**
2. **将用户传入的路径解析为绝对路径**
3. **检查解析后的路径是否以工作目录开头**

## TypeScript 完整实现示例

```typescript
import * as path from 'path';
import * as fs from 'fs/promises';

class SecureFileSystemTool {
  private readonly workspaceDir: string;

  constructor(workspaceDir: string) {
    // 1. 初始化时，确保 workspaceDir 是绝对路径且已规范化
    this.workspaceDir = path.resolve(workspaceDir);
  }

  /**
   * 安全地解析路径，如果越界则抛出异常
   */
  private getSecurePath(targetPath: string): string {
    // 2. path.resolve 会处理掉所有的 '..' 和 '.'，并结合当前工作目录返回绝对路径
    // 如果 targetPath 已经是绝对路径（如 /etc/passwd），resolve 会直接返回它
    const resolvedPath = path.resolve(this.workspaceDir, targetPath);

    // 3. 关键防御：检查解析后的路径是否以 workspaceDir 开头
    // 注意：为了防止类似 /workspace-hacked 匹配 /workspace 的情况，
    // 最好在 workspaceDir 后面加上 path.sep (即 '/')
    const prefix = this.workspaceDir + path.sep;
    
    // 如果 resolvedPath 就是 workspaceDir 本身，也是允许的
    if (resolvedPath !== this.workspaceDir && !resolvedPath.startsWith(prefix)) {
      throw new Error(`Security Error: Access denied. Path '${targetPath}' is outside the workspace.`);
    }

    return resolvedPath;
  }

  // 供 LLM 调用的工具函数
  async readFile(args: { path: string }): Promise<string> {
    try {
      const securePath = this.getSecurePath(args.path);
      const content = await fs.readFile(securePath, 'utf-8');
      return content;
    } catch (error) {
      // 将错误信息返回给 LLM，LLM 可能会尝试修正路径
      return `Error reading file: ${error.message}`;
    }
  }
}

// 测试用例
const tool = new SecureFileSystemTool('/Users/didi/my-project');

// ✅ 合法访问
// tool.getSecurePath('src/index.ts') -> /Users/didi/my-project/src/index.ts

// ❌ 恶意访问：相对路径穿越
// tool.getSecurePath('../../../etc/passwd') -> /etc/passwd (抛出异常)

// ❌ 恶意访问：绝对路径
// tool.getSecurePath('/etc/passwd') -> /etc/passwd (抛出异常)
```

## 网络请求的静态作用域 (URL 拦截)

同样的逻辑适用于网络请求。如果你有一个 `fetchUrl` 工具，你需要用 `URL` 对象来解析并校验 `hostname`。

```typescript
function isAllowedUrl(targetUrl: string, allowedDomains: string[]): boolean {
  try {
    const url = new URL(targetUrl);
    // 检查协议
    if (url.protocol !== 'http:' && url.protocol !== 'https:') return false;
    
    // 检查域名是否在白名单中（或者以白名单域名结尾，如 .github.com）
    return allowedDomains.some(domain => 
      url.hostname === domain || url.hostname.endsWith(`.${domain}`)
    );
  } catch {
    return false; // URL 解析失败直接拒绝
  }
}
```
