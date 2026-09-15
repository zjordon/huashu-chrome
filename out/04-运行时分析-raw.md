# 运行机制分析

## 入口点分析

### 主入口
**文件**: 

### CLI 入口

## 主函数位置


## 初始化流程

### 初始化相关文件

## 中间件/拦截器

  const liveSessions = () => [...new Set([...agents].map((a) => a.sid).filter(Boolean))];
  const liveExtensions = () => [...extensions].filter((e) => e.readyState === 1);
    const headed = live.filter((e) => !e.headless);
    const first = out.split(/\r?\n/).map((s) => s.trim()).filter(Boolean)[0];
  const all = knownAgents().filter((a) => a.file);
    found = found.filter((a) => a.client === only);
  try { sites = fs.readdirSync(path.join(ROOT, 'docs', '经验')).filter((f) => f.endsWith('.md')).length; } catch { /* 没带 docs 就不报数 */ }
  const ok = rows.filter((r) => r.ok).length;
  const written = rows.filter((r) => r.written).length;
    return fs.readdirSync(dir).filter((f) => f.endsWith('.md')).map((f) => f.slice(0, -3));

## 事件处理

### 事件监听器
// 桥 Daemon —— 全机单例，坐在 agent 和 Chrome 扩展中间
// 网页的 Origin 是自己的域名，扩展的是 chrome-extension://<id>。只放后者进来。
import { VERSION } from './lib/version.js';
import { scrubProse } from '../extension/redact.js';
const PROTOCOL = 1;
const HELLO_TIMEOUT = 5000;
const CMD_TIMEOUT = 30000;
const IDLE_EXIT_MS = 30 * 60 * 1000; // 半小时没人用就自己退，别留僵尸进程
const EXT_SILENCE_MS = 50000;        // 扩展每 15 秒一次心跳，连着三拍没到就先探一下
const EXT_PROBE_MS = 15000;          // 探了还不回，再等这么久才判死
