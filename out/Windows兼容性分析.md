# huashu-chrome Windows 兼容性分析

> 分析日期：2026-09-15 · 版本：v1.2.0
> 数据来源：`windows-compat-checker.sh` 自动扫描 + 逐处源码核验 + 本机实测（Windows 11 · Node v24.14.0）

## 总评

**Windows 是一等公民，整体兼容性良好。** `src/install.js` 全篇显式做了 `WIN = process.platform === 'win32'` 三分支（打开浏览器、找 node、启动命令），配置路径支持 `$APPDATA` 展开与 `/` → `path.sep` 转换，`npx.cmd` 是 Windows 下 spawn 的正确写法。自动扫描未发现无守卫的 Unix 专用调用、硬编码 Unix 路径或裸 `$HOME` 依赖。

| 维度 | 评级 | 说明 |
|---|---|---|
| 安装器（install.js） | ✅ 良好 | 三平台分支齐全，实测语义正确 |
| 进程管理 | ⚠️ 有差异 | SIGTERM 在 Windows 是无条件终止，优雅退出不生效（见 §3.1） |
| 文件系统 | ⚠️ 一处风险 | 跨盘符 `fs.renameSync` 会 EXDEV 失败（见 §3.2） |
| 路径处理 | ✅ 良好 | 全部 `path.join` / `os.homedir()` / `APPDATA` |
| 文件权限 | ℹ️ 语义降级 | 0600/0700 在 Windows 上是 no-op，靠 per-user 目录 ACL 兜底 |
| 终端输出 | ℹ️ 外观问题 | 中文/emoji 在旧 conhost 可能乱码 |
| CI 覆盖 | ❌ 缺失 | 仅 ubuntu-latest，Windows 路径无回归保护 |

---

## 1. 自动扫描结果摘要（脚本原始输出见 `out/05-Windows兼容性-raw.md`）

### 1.1 spawn 调用（7 处，全部核验为安全）

| 位置 | 代码 | Windows 判定 |
|---|---|---|
| `src/cli.js:351` | `spawn('explorer', ['/select,' + dir])` | ✅ Windows 专用分支（`--reveal` 用），标准资源管理器选中语法 |
| `src/install.js:315` | `spawn('cmd', ['/c', 'start', '', target])` | ✅ Windows 专用分支。**实测验证**：Node 对含空格参数自动加引号（本机 echo 试验确认），最终命令行为 `start "" "路径"` —— 恰是「带标题占位的 start」这一正确惯用法，含空格的用户名路径（`C:\Users\John Smith\...`）不会出错 |
| `src/install.js:316-317` | `open` / `xdg-open` | ✅ 仅 macOS/Linux 分支 |
| `src/lib/rpc.js:187` | `spawn(process.execPath, [CLI, 'bridge'])` | ✅ `node.exe` 是真可执行文件，detached + 文件重定向 stdio，无窗口闪烁问题 |

**关键正确适配**（扫描脚本未标出但值得记录）：

- `src/install.js:123`：`command: WIN ? 'npx.cmd' : 'npx'` —— Windows 上 spawn 不带 `shell:true` 找不到 `npx`（只有 `npx.cmd`，没有 `npx.exe`），这一字之差决定写进 agent 配置的命令能否被拉起
- `src/install.js:114`：`nodeBin()` 用 `where node`（而非 `which`），且过滤掉路径中带版本号的 nvm/homebrew 安装位（正则同时匹配 `\` 与 `/`，覆盖 nvm-windows 的 `AppData\Roaming\nvm\v22.x.x` 布局）

### 1.2 扫描误报澄清

| 报告项 | 核验结论 |
|---|---|
| `src/lib/learnings.js:28` 疑似硬编码 `:` 分隔符 | **误报**。`split(':')[0]` 是 URL 归一化里剥离端口（`example.com:8080` → `example.com`），与 `path.delimiter` 无关 |
| HOME 环境变量无 USERPROFILE 回退 | 全库统一用 `os.homedir()`（`src/lib/paths.js:7`），Windows 上正确返回 `%USERPROFILE%` |

## 2. 已验证的 Windows 适配清单

| 适配点 | 位置 |
|---|---|
| 配置目录 = `os.homedir()/.huashu-chrome` | `src/lib/paths.js:7` |
| agent 配置路径：`~` 与 `$APPDATA` 双格式，`expand()` 统一转 `path.sep`（20 家 agent 中 4 家用 `$APPDATA`，其余 `~/` 在 Windows 同样有效） | `src/install.js:34-37, 24-28` |
| Windows 探测命令 `where` | `src/install.js:114` |
| 打开引导页 `cmd /c start` | `src/install.js:315` |
| 资源管理器选中扩展目录 `explorer /select,` | `src/cli.js:351` |
| 单例文件锁 `O_EXCL`（`fs.openSync(lock, 'wx')`） | `src/lib/rpc.js:179` —— Windows 上语义一致 |
| `process.kill(pid, 0)` 探活 | `src/cli.js:150` —— Windows 支持 signal 0 探测 |
| 引导页写 `os.tmpdir()` | `src/install.js:278` |
| 路径分隔符全部经 `path.join` / `path.extname` / `path.basename` | 全库 |

## 3. 发现的真实差异与风险

### 3.1 ⚠️ SIGTERM 在 Windows 上不触发优雅退出（中等严重度）

**涉及**：`src/bridge.js:425-437`（SIGTERM/SIGINT 处理器：先将在途命令回成「桥正在重启」再退出）、`src/lib/rpc.js:164-173`（`stopBridge` 用 SIGTERM 换代旧桥）

**机制**：Windows 上 Node 的 `process.kill(pid, 'SIGTERM')` 是无条件终止（TerminateProcess），目标进程的 `process.on('SIGTERM')` 处理器**不会执行**。只有控制台事件（前台 Ctrl+C → SIGINT）能走处理器。

**Windows 上的实际后果**：
1. 版本换代（`stopBridge`）变成硬杀——在途命令收不到「桥正在重启，重试一次即可」的错误回执，agent 会干等到 35s 超时（`TIMEOUT`），再重连时发现新桥已在——**功能可恢复，但换代瞬间多等最多 35 秒且错误信息有误导性**
2. 桥退出前不会 `wss.close()`，扩展侧靠看门狗（45s 无回音）发现连接已死——比 macOS 慢约 45 秒
3. 用户手动 kill 桥进程同理

**建议修复**（按侵入性从低到高）：
- 换代前先通过 WS 发一条控制消息让桥自退（桥已有 token 鉴权的 agent 通道）：
  ```js
  // stopBridge 里，SIGTERM 之前：
  const ws = new WebSocket(`ws://127.0.0.1:${info.port}`);
  ws.onopen = () => ws.send(JSON.stringify({ type: 'hello', role: 'agent', token: info.token, ... }));
  // 桥侧加一条 shutdown 控制命令，收到后走既有的优雅退出路径
  ```
- 或在文档中如实声明 Windows 下的行为差异（换代慢 35s），接受降级

### 3.2 ⚠️ 跨盘符 `fs.renameSync` 抛 EXDEV（低频但静默失败）

**涉及**：`src/cli.js:63`、`src/mcp-server.js:636-638`（download 完成后把文件从 Chrome 下载目录挪到 `savePath`）

**机制**：Windows 多盘符环境（`C:\Users\...\Downloads` → `D:\data\out.csv`）远比 macOS 常见（macOS 单一文件系统几乎不触发）。`fs.renameSync` 跨卷抛 `EXDEV`，download 工具报错——文件其实已完整落在 Downloads 里，但 agent 收到的是失败，可能重试造成重复下载。

**建议修复**：
```js
try {
  fs.renameSync(data.path, args.savePath);
} catch (e) {
  if (e.code === 'EXDEV') {
    fs.copyFileSync(data.path, args.savePath);
    fs.unlinkSync(data.path);
  } else throw e;
}
```
（两处调用可提成 `lib/paths.js` 的 `moveFile(from, to)` 复用。）

### 3.3 ❌ CI 无 Windows runner（测试保护缺口）

`.github/workflows/publish.yml` 仅 `ubuntu-latest`。本项目的 `npm test`（16 个文件）**不需要浏览器**（协议/脱敏/风控判定都是纯函数），加一个 Windows 矩阵几乎零成本，却能把 §3.1/§3.2 这类平台差异挡在发布前：

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
runs-on: ${{ matrix.os }}
```

### 3.4 ℹ️ 文件权限模式在 Windows 上是 no-op（可接受）

`src/lib/paths.js`：`ensureHome` 的 `mode: 0o700`、`writeBridgeInfo` 的 `mode: 0o600` 在 Windows 上被忽略——`bridge.json` 里的 token 文件不设 ACL。

**实际风险有限**：文件位于 `%USERPROFILE%\.huashu-chrome\`，该目录默认继承 per-user ACL，同机其他标准用户读不到；暴露面与 Unix 下「同 uid 进程可读」相当。如要严格对齐，可用 `icacls` 收紧，优先级低。

### 3.5 ℹ️ 中文/emoji 控制台输出在旧终端可能乱码（外观）

CLI 大量输出中文与 ✅❌⚠️（install / doctor / audit），文件按 UTF-8 读写。Windows Terminal 与 PowerShell 7 下正常；**旧 conhost（默认代码页 GBK=936）下会乱码**。`chcp 65001` 或使用 Windows Terminal 可解。属已知 Windows 生态问题，不构成功能缺陷，但面向中文用户的排错文档可加一句提示。

### 3.6 ℹ️ `engines: node >=20` 与全局 WebSocket 的错位（跨平台问题，附带记录）

`src/lib/rpc.js:88`、`src/cli.js:313` 直接使用全局 `new WebSocket(...)`。本机 Node v24 实测可用（`typeof WebSocket === 'function'`）；但全局 WebSocket 在 Node 21 才默认启用、22 转稳定——**Node 20.x 上裸调会 `ReferenceError`**（除非 `--experimental-websocket`）。`engines` 声明与实际不符，Windows/macOS/Linux 一视同仁。建议：要么把 engines 提到 `>=22`，要么在 README 标注。

## 4. 逐项核对结论表

| 检查维度（对照 WINDOWS_COMPAT 清单） | 结论 |
|---|---|
| package.json scripts 含 bash 语法 | ✅ 无（全部 `node src/cli.js ...` 形式） |
| 生命周期 hooks | ✅ 无 |
| spawn 缺 shell:true / .cmd 回退 | ✅ 全部平台守卫 + `npx.cmd` 显式适配 |
| 平台守卫覆盖 | ✅ cli.js(2) / install.js(2)，另有路径层 `path.sep` 转换 |
| Unix 专用工具（grep/sed/ln 等）无守卫 | ✅ 未发现 |
| 硬编码 Unix 路径（/usr、/tmp、/var） | ✅ 未发现（临时文件走 `os.tmpdir()`） |
| `$HOME` 无 USERPROFILE 回退 | ✅ 统一 `os.homedir()` |
| PATH 分隔符硬编码 `:` | ✅ 仅 URL 端口剥离误报一处 |
| .sh / .ps1 / .cmd 脚本 | ✅ 仓库无依赖 |
| 信号处理 | ⚠️ 见 §3.1 |
| 符号链接操作 | ✅ 无 |
| CI/CD 平台覆盖 | ❌ 仅 Linux，见 §3.3 |
| Docker | ✅ 不涉及 |
| 大小写敏感文件名 | ✅ 全部代码内引用，无跨文件大小写漂移 |

## 5. 修复优先级建议

| 优先级 | 项 | 工作量 |
|---|---|---|
| P1 | CI 加 `windows-latest` 矩阵（§3.3） | 一行 YAML |
| P2 | download 挪文件加 EXDEV 回退（§3.2，两处） | ~10 行 |
| P2 | 换代走 WS 控制消息替代 SIGTERM，或在文档声明 Windows 差异（§3.1） | ~20 行 / 0 行 |
| P3 | engines 与全局 WebSocket 对齐（§3.6） | 一行 |
| P4 | 文档注明旧终端乱码解法（§3.5）；可选 icacls 收紧（§3.4） | 文档/可选 |
