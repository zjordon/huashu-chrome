# Windows 兼容性检查结果

## 1. package.json Scripts 检查

### 可疑 bash 语法

| Script | 匹配项 | 完整命令 |
|--------|--------|----------|
| (无) | - | - |

### 生命周期 Hooks

未发现生命周期 hooks。

## 2. child_process 调用检查

### spawn() 调用 (可能缺少 shell:true 或 .cmd 回退)

| 文件 | 行号 | 代码 |
|------|------|------|
| `src/cli.js` | 350 | `if (process.platform === 'darwin') spawn('open', ['-R', dir], { detached: true, ` |
| `src/cli.js` | 351 | `else if (process.platform === 'win32') spawn('explorer', ['/select,' + dir], { d` |
| `src/cli.js` | 352 | `else spawn('xdg-open', [path.dirname(dir)], { detached: true, stdio: 'ignore' })` |
| `src/install.js` | 315 | `if (WIN) spawn('cmd', ['/c', 'start', '', target], { detached: true, stdio: 'ign` |
| `src/install.js` | 316 | `else if (MAC) spawn('open', [target], { detached: true, stdio: 'ignore' }).unref` |
| `src/install.js` | 317 | `else spawn('xdg-open', [target], { detached: true, stdio: 'ignore' }).unref();` |
| `src/lib/rpc.js` | 187 | `const child = spawn(process.execPath, [CLI, 'bridge'], {` |

未发现可疑的 exec 系列调用。

## 3. 平台守卫检查

### 已有平台守卫的文件

- `src/cli.js` (2 处平台检查)
- `src/install.js` (2 处平台检查)

### Unix 专用工具使用 (无平台守卫)

未发现无守卫的 Unix 专用工具调用。

## 4. 路径和文件系统检查

### 硬编码 Unix 路径

未发现硬编码的 Unix 路径。

### 环境变量 HOME 使用 (无 USERPROFILE 回退)

未发现 HOME 环境变量使用。

### PATH 分隔符 (硬编码 ':' 而非 path.delimiter)

| 文件 | 行号 | 代码 |
|------|------|------|
| `src/lib/learnings.js` | 28 | `d = d.replace(/^[a-z]+:\/\//, '').split(/[/?#]/)[0].split(':')[0];` |

## 5. Shell 脚本检查

### .sh 文件

未发现 .sh 文件。

### .ps1 / .cmd 文件

未发现 .ps1 或 .cmd 文件。

## 6. 信号处理检查

| 文件 | 行号 | 信号 |
|------|------|------|
| `src/bridge.js` | 426 | `SIGTERM
SIGINT` |
| `src/lib/rpc.js` | 162 | `SIGTERM` |
| `src/lib/rpc.js` | 167 | `SIGTERM` |

> 注意: Windows 对 Unix 信号的支持有限。SIGINT 可用，但 SIGTERM/SIGUSR 等行为不同。

## 7. 符号链接检查

未发现符号链接操作。

## 8. CI/CD 配置检查

### GitHub Actions

| 工作流 | Runner | 排除 Windows? |
|--------|--------|--------------|
| `publish.yml` |     runs-on: ubuntu-latest | — 仅 Linux/macOS |

## 9. Docker / 容器检查

未发现 Docker 配置文件。

## 检查总结

以上为自动化扫描结果。AI 应结合脚本输出和源代码分析，生成完整的 Windows 兼容性报告。
