# learnings.js 模块逻辑与提示词分析

> 分析日期：2026-09-15 · 对象：`src/lib/learnings.js`（145 行）+ 三处相关提示词
> 定位：经验回流的全部读写逻辑。纯 Node 模块，不碰 chrome API、不经桥，被 `mcp-server.js` 直接调用

## 1. 模块在系统中的位置

```
agent ──tools/call learnings──> src/mcp-server.js（本地分支，不连桥）
                                   ├── getLearnings(domain)      ← 读
                                   └── saveLearnings(domain, save) ← 写
                                          ↓
                          src/lib/learnings.js（本文件）
                          ├── SEED_DIR      = <包内>/docs/经验/        （出厂层，只读）
                          └── LEARNINGS_DIR = ~/.huashu-chrome/learnings/（本机层，读写）
```

关键前提：learnings **不经过桥和浏览器**（`mcp-server.js:558-564` 特设本地分支），所以它是全链路里唯一一个 agent 断网（扩展离线）也能用的工具——查经验不该被「扩展没连上」阻塞。

导出面：`getLearnings` / `saveLearnings` / `lintPlaybooks` / `LEARNINGS_DIR` 四个，`HUASHU_CHROME_LEARNINGS_DIR` 环境变量仅供测试隔离（learnings.js:13 注释明示）。

## 2. 主逻辑逐段拆解

### 2.1 域名归一化（normalize，learnings.js:26-30）

```js
d = d.replace(/^[a-z]+:\/\//, '').split(/[/?#]/)[0].split(':')[0];
return d.replace(/^www\./, '');
```

一条链做四件事：去协议头 → 截到主机名（丢路径/查询/锚点）→ 去端口 → 去 `www.`。输入容错宽：完整 URL、裸域名都收（agent 传什么都有结果）。

### 2.2 归属解析（resolveKey，learnings.js:40-52）——本文件最核心的算法

问题：`my.feishu.cn` 的经验存在 `feishu.cn.md` 里，怎么找到？——**逐级去子域，在「已知键集合」里从长到短匹配**：

```js
const known = new Set([...mdFiles(SEED_DIR), ...mdFiles(LEARNINGS_DIR)]);  // 两层目录的文件名并集
for (let i = 0; i < parts.length - 1; i++) {          // 遍历所有后缀，但不含裸 TLD
  const cand = parts.slice(i).join('.');
  const key = ALIAS[cand] || cand;                    // 每级先过别名表
  if (known.has(key)) return key;
}
return ALIAS[d] || d;                                 // 全没命中 → 全名（过别名）当新键
```

四个设计细节：

1. **已知集合是两层目录的并集**——本机新建的站也能被当作归属锚点（我在本机存过 `foo.com.md`，下次查 `bar.foo.com` 就能归进去）
2. **循环上界 `parts.length - 1`**——永远不把裸 TLD（`cn`、`com`）当候选键，防止一个 `cn.md` 吞掉所有 .cn 站点
3. **ALIAS 在循环内逐级应用**——`sub.twitter.com` 逐级走到 `twitter.com` 时经别名映射到 `x.com`，与已有笔记对上
4. **兜底返回全名**——全新站点 `sub.newsite.com` 找不到锚点时，以全名为新键建文件（不会自作聪明归到 `newsite.com`——那可能是个不属于用户的域）。副作用：同一新站的子域先后来存，会散落成多个文件，直到某天有人 PR 一个父域笔记才收敛

### 2.3 读主流程（getLearnings，learnings.js:102-131）——四条分支

```
getLearnings(domain)
 ├─ 无 domain      → 列出所有站名，教 agent 带域名再调一次
 ├─ resolveKey 后两层都读不到 → 「还没有记录」+ 已有站点清单 + 用回显的 key 教它存
 └─ 有内容（出厂/本机/两者）→ [提示头 HINT] + [## 出厂经验（v1.2.0）] + [## 本机经验]
                              + （有剧本时）[剧本使用说明]
```

分支三的组装细节：

- **版本戳**：出厂层标题带 `huashu-chrome v${VERSION}`——agent 不用猜这段知识多新，与包版本绑定
- **次序**：HINT 最前、出厂居中、本机最后——本机经验是最新修正，放最靠近阅读末端的位置，也是「补充修正出厂」的语义
- **零噪音**：剧本提示只在 `countPlaybooks > 0` 时附加（learnings.js:123 注释：「没有就零噪音（经验不阻塞，也不添乱）」）；HINT 只在有内容时出现

### 2.4 剧本体检（lintPlaybooks，learnings.js:79-98）

```
提取 ```act 块 → {{占位符}} 全部替换成 'X' → JSON.parse
  → 必须是非空数组 → validateScript（extension/script.js 的全套规则）
```

三处工程细节：

- **占位符先换假值再解析**（learnings.js:88）——`{{关键词}}` 本身不是 JSON 错误，体检的是「填完之后的形状」
- **全局正则状态复位**——`PLAYBOOK_RE` 带 `g` 标志是有状态的，函数开头 `lastIndex = 0` 显式归零（learnings.js:83），否则同一进程里第二次调用会从上次断点继续、漏检前几个块。这类 bug 静默且难查，作者显式处理了
- **校验规则不在这里**——`validateScript` 从 `extension/script.js` import，与 background 执行器、mcp-server 超时预算**三方同源**（script.js 头注释：「各抄一份的下场是静默分叉」）

### 2.5 写主流程（saveLearnings，learnings.js:133-145）——守卫链

```
resolveKey(domain)
 ├─ 无 key（空输入）      → '缺 domain，没存。'
 ├─ body 为空             → '内容为空，没存。要清掉本机经验请直接说明再人工删。'  ← 拒绝用空存当删除
 ├─ 通过                  → mkdir(0700) → writeFileSync（整文件覆盖）→ lintPlaybooks(body)
 └─ 返回 '已存 → learnings/<key>.md（N 字）。本机经验整文件覆盖：下次保存前先 get 合并旧内容。'
        + 体检不合格时追加 ⚠️ 警告（照存，但说清坏剧本下个会话会炸）
```

值得注意的两点：

- **「空存 ≠ 删除」是刻意设计**——agent 不能用一个空 save 把本机经验抹掉；清经验必须人出手。这是防 agent 误操作/被注入后的破坏半径控制
- **先写后检**——writeFileSync 在 lintPlaybooks 之前：体检不合格**照存**，只在返回里警告。次序不是疏忽，是「经验不阻塞」原则的落实（存本身永远成功，坏剧本的代价被限制为「下个会话跑不动」而不是「这轮任务卡住」）

## 3. 相关提示词全清单

learnings 的「提示词」分三层：**握手注入一次**（STRATEGY）、**工具描述常驻**、**返回文本每次**。逐条列出并标注设计意图：

### 3.1 握手层——STRATEGY 的 LEARNINGS FIRST 段（mcp-server.js:494-497，每次会话注入一次）

```
LEARNINGS FIRST. Before acting on a site, call `learnings` with its domain — past sessions
may have mapped its APIs, walls and pitfalls (some as runnable ```act playbooks).
Notes are hints, not rules — trust the page when they disagree, then save corrections back.
```

三句话干三件事：**行为规定**（先查）、**价值主张**（past sessions 可能已铺好路）、**防腐原则**（页面说了算 + 改回来）。它必须短——STRATEGY 有 2000 字符硬预算（Claude Code 会截断，mcp-server.js:485-488 注释），所以细节全部下放到工具描述和返回文本。

### 3.2 工具描述层（mcp-server.js:456-460，常驻 agent context，每次工具列表都在）

```
Site notes: APIs, walls, pitfalls from past sessions. Call {domain} before first acting on a site
(no args = list sites). Notes may embed runnable ```act scripts — fill {{placeholders}} and run
them instead of rediscovering. Learned something non-obvious or mapped a flow? Save the full note
back via {domain, save}. Hints, never rules.
```

压缩到六行的完整行为闭环：是什么 → 何时查（含无参用法）→ 剧本怎么用 → 何时存（**non-obvious** 是质量闸：显而易见的不值得存）→ 最后一句防腐定调。注释（mcp-server.js:481-488）明说这套分层的成本逻辑：instructions 一份只发一次但会被截断，工具描述不截断但每处引用都在付 context——所以宪法放 STRATEGY、细节放这里。

### 3.3 返回文本层（learnings.js 内嵌，六条）——最密集的提示词工程

| # | 触发 | 文本 | 设计意图 |
|---|---|---|---|
| 1 | 库为空 | `经验库是空的。做完任务学到非显而易见的规律时，用 {domain, save} 存下来。` | 空库不是死路，顺势播种写回路 |
| 2 | 站点无记录 | `「{key}」还没有经验记录——按通用流程干就行，别让查询空手而归拖慢任务。` + 已有站点清单 + `摸清这个站后用 {domain: "{key}", save} 把规律存下来` | **三连**：不阻塞（按通用流程干）+ 可视（有哪些站）+ 教学回显 **canonical key**——agent 传的是子域/URL，回显归一化后的键，存的时候自然落到对的文件 |
| 3 | 有内容，头部 | `[经验仅供参考，不是规则。它记录的是过去某个时点的实况——站点会改版、环境各不相同。与页面实际不符时，以你观察到的实际为准，并在收工时用 learnings(domain, save) 改写。]`（HINT，learnings.js:62-64） | 防腐三件套：免责 + 冲突裁决规则（页面赢）+ 冲突时的动作（改写）。**每次读经验都带**，不是一次性声明 |
| 4 | 分节标题 | `## 出厂经验（huashu-chrome v1.2.0）` / `## 本机经验` | 来源与新鲜度信号：出厂的跟包版本走，本机的是本地实况 |
| 5 | 有剧本时尾部 | `[上面有 N 份可执行剧本（```act 块）：内容就是 act 的 steps，把 {{占位符}} 填成实值后可直接运行——一条 act 顶过去几十轮试错。剧本可能过时：assert/until 会在页面不符时停下，停了就按现场重走，收工时把新流程写回来。]` | 用法 + 收益量化 + 失效行为预告（剧本会自己停）+ 停了怎么办——把「过时」的失败模式提前写进使用说明 |
| 6 | 保存回执 | `已存 → learnings/{key}.md（N 字）。本机经验整文件覆盖：下次保存前先 get 合并旧内容。`（+ 体检警告 `照存了（经验不阻塞），但这样的剧本下个会话直接跑会失败——修好再 save 一次`） | 回显落点（agent 知道存哪了）+ **覆盖语义警告**（逼先读后写）+ 坏剧本不静默 |

### 3.4 外围——docs/双脑.md 的 Driver 提示词模板（第 1、6 条规矩）

```
1. 开工先 learnings(domain) 查经验；有 ```act 剧本就填好占位符直接跑。
6. 收工把学到的规律（含跑通的剧本）save 回 learnings。
```

主脑派 subagent 时的岗前须知，把同一闭环再钉一遍——这是第四个触点，覆盖「快脑」跑循环时主脑忘了交代的场景。

## 4. 提示词工程的设计规律（四条）

**① 每条返回都可行动，没有死路。** 空库→教存；无记录→教查清单+存；有坏剧本→照存+教修。任何分支都不会让 agent 停下来问人。

**② 防腐思想出现在四个触点，措辞各不同但裁决规则唯一。** STRATEGY：`trust the page when they disagree`；工具描述：`Hints, never rules`；HINT：`以你观察到的实际为准，并……改写`；剧本提示：`停了就按现场重走，收工时把新流程写回来`。同一原则在「会话开头、选工具时、每次读、用剧本时」四个时机重复强化——单点声明会被上下文冲淡。

**③ 语言的分工是刻意的。** 面向 agent 的协议文本（工具描述、STRATEGY）是英文——token 效率高、且这是 MCP 生态惯例；返回的**数据**（经验内容、HINT）是中文——内容层跟着作者和用户群走。HINT 夹在数据里所以也是中文，正好充当「数据自带的免责声明」。

**④ 长度即预算。** STRATEGY 的 LEARNINGS FIRST 段只占 ~230 字符（全段预算 2000）；工具描述六行；返回提示全部一句话级别。注释里反复出现「描述写长一个字，agent 的 context 就多付 N 份」的成本意识。

## 5. 与安全链的交叉点

- **审计脱敏**：learnings 调用单独记审计（mcp-server.js:562），`save` 内容**不落审计**——只记 `<N 字>`。经验内容可能含敏感实况（订单号、内部 URL），审计日志是明文 JSONL，所以按长度脱敏
- **一个值得注意的缺口**：learnings 返回**不裹** `<page-content untrusted>` 边界（与 read_text 等页面数据不同，mcp-server.js:563 直接 `return {content:[{type:'text', text}]}`）。出厂层有 PR 人审兜底，但本机经验是 agent 从页面世界总结写回的——一个恶意页面理论上可以诱导 agent 把「指令文本」存成经验，让注入**跨会话持久化**（下个会话读经验时无降权边界）。当前无实测案例，属设计层面值得留意的一环；对策也简单——getLearnings 返回也过一遍 wrapUntrusted
- **写回人的内容有闸**：空存拒绝（§2.5）；PR 前脱敏靠 README 人工提醒（无工具兜底）

## 6. 模块总评

145 行做了「一个双层知识库的完整 CRUD + 体检 + 提示词系统」。最出色的一点是**返回文本被当成提示词来设计**——六条返回语每条都同时是数据（告诉 agent 结果）和指令（规定下一步行为），并且与 STRATEGY、工具描述构成四触点的行为闭环。算法上 resolveKey 的逐级归属 + 别名归并 + 裸 TLD 防护，用二十行解决了「子域/别名/新站」三类真实脏数据。可以挑剔之处：新站子域散文件问题（§2.2）、learnings 返回无降权边界（§5）、lint 只查形状不查语义（占位符换 X 后 validateScript 通过不代表填实值后仍合法）。
