# 依赖关系分析

## npm/yarn/pnpm 依赖

### 生产依赖

### 开发依赖

## 依赖统计

- 生产依赖: 
- 开发依赖: 

## 模块导入分析

### ES Modules 导入
      9 import fs from 'node:fs';
      8 import path from 'node:path';
      6 import { fileURLToPath } from 'node:url';
      2 import { spawn } from 'node:child_process';
      2 import { VERSION } from './version.js';
      2 import { VERSION } from './lib/version.js';
      2 import os from 'node:os';
      2 import crypto from 'node:crypto';
      1 import { validateScript } from '../../extension/script.js';
      1 import { spawn, execFileSync } from 'node:child_process';
      1 import { scrubProse } from '../extension/redact.js';
      1 import { resolveHost } from './lib/host.js';
      1 import { readBridgeInfo, DEFAULT_PORT, HOME, AUDIT_FILE, LOG_FILE } from './lib/paths.js';
      1 import { getLearnings, saveLearnings } from './lib/learnings.js';
      1 import { flatCount } from '../extension/script.js';
      1 import { audit } from './lib/paths.js';
      1 import { WebSocketServer } from 'ws';
      1 import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
      1 import { Server } from '@modelcontextprotocol/sdk/server/index.js';
      1 import { HOME } from './paths.js';

### Python 导入

