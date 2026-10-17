# V0.2 补充 - IDE 类 AI 工具采集

## 概述

除了 CLI 工具（Claude Code、Codex），还需要支持 IDE 类 AI 开发工具的对话采集：

| 工具 | 类型 | 对话存储位置 |
|------|------|-------------|
| Cursor | AI IDE | `~/.cursor/` 或工作区 `.cursor/` |
| Kiro | AI IDE | `~/.kiro/` 或工作区 `.kiro/` |
| Trea | AI IDE (字节) | `~/.trea/` 或工作区 `.trea/` |
| Claude Code | CLI | `~/.claude/projects/` |
| Codex | CLI | `~/.codex/` |

---

## 各工具对话存储分析

### Cursor

**存储位置：**
```
# 用户级（macOS）
~/Library/Application Support/Cursor/User/globalStorage/
~/Library/Application Support/Cursor/User/workspaceStorage/

# 工作区级
.cursor/
```

**对话格式：** SQLite 数据库或 JSON 文件，包含 Composer 和 Chat 的对话记录。

**采集策略：**
- 监控 `~/Library/Application Support/Cursor/` 目录
- 解析 SQLite 数据库中的对话表
- 监控工作区 `.cursor/` 目录变化

---

### Kiro (AWS/Anthropic)

**存储位置：**
```
# 用户级（macOS）
~/Library/Application Support/Kiro/
~/.kiro/

# 工作区级
.kiro/
```

**对话格式：** JSON 文件，包含 spec、对话历史、任务记录。

**采集策略：**
- 监控 `~/.kiro/` 和应用数据目录
- 解析 JSON 对话文件
- 关注 `.kiro/specs/` 下的需求文档（也是有价值的知识）

---

### Trea (字节跳动)

**存储位置：**
```
# 用户级（macOS）
~/Library/Application Support/Trea/
~/.trea/

# 工作区级
.trea/
```

**对话格式：** JSON 或 SQLite，包含 AI 对话和代码建议记录。

**采集策略：**
- 监控 Trea 应用数据目录
- 解析对话存储文件
- 提取代码建议和对话内容

---

## 需求拆分

### 需求 0.2.8：Cursor 对话采集

**描述：** 自动采集 Cursor IDE 中的 AI 对话记录

**验收标准：**
- 能找到 Cursor 对话存储位置
- 正确解析 Cursor 对话格式（Composer + Chat）
- 区分不同工作区的对话
- 增量采集（只采集新对话）
- 来源标记为 `cursor`

**具体步骤：**
1. 创建 `electron/services/collector/cursor.ts`
2. 研究 Cursor 实际存储结构（启动 Cursor，产生对话，查看文件变化）
3. 确定对话存储格式（SQLite / JSON / LevelDB）
4. 实现对话解析器
5. 注册到文件监控服务
6. 处理 Cursor 更新后存储格式变化的兼容性

**探索命令（开发时用）：**
```bash
# 查看 Cursor 数据目录结构
find ~/Library/Application\ Support/Cursor -name "*.json" -newer /tmp/marker 2>/dev/null
find ~/Library/Application\ Support/Cursor -name "*.db" -o -name "*.sqlite" 2>/dev/null

# 监控文件变化（开发调试用）
fswatch ~/Library/Application\ Support/Cursor/
```

---

### 需求 0.2.9：Kiro 对话采集

**描述：** 自动采集 Kiro IDE 中的 AI 对话和 Spec 文档

**验收标准：**
- 能找到 Kiro 对话存储位置
- 正确解析 Kiro 对话格式
- 同时采集 Spec 文档（需求/设计文档也是知识）
- 增量采集
- 来源标记为 `kiro`

**具体步骤：**
1. 创建 `electron/services/collector/kiro.ts`
2. 研究 Kiro 实际存储结构
3. 解析对话 JSON 文件
4. 解析 `.kiro/specs/` 下的 Spec 文档
5. 注册到文件监控服务
6. 测试采集流程

---

### 需求 0.2.10：Trea 对话采集

**描述：** 自动采集 Trea IDE 中的 AI 对话记录

**验收标准：**
- 能找到 Trea 对话存储位置
- 正确解析 Trea 对话格式
- 增量采集
- 来源标记为 `trea`

**具体步骤：**
1. 创建 `electron/services/collector/trea.ts`
2. 研究 Trea 实际存储结构
3. 实现对话解析器
4. 注册到文件监控服务
5. 测试采集流程

---

### 需求 0.2.11：通用 IDE 采集框架

**描述：** 抽象出通用的 IDE 对话采集框架，方便后续扩展新工具

**验收标准：**
- 统一的采集器接口（Collector Interface）
- 新增采集源只需实现接口 + 注册
- 支持插件式扩展
- 统一的错误处理和日志

**具体步骤：**
1. 创建 `electron/services/collector/base.ts` - 基础采集器抽象类
2. 定义统一接口

```typescript
// electron/services/collector/base.ts
interface CollectorConfig {
  name: string;           // 采集器名称
  source: string;         // 来源标识
  enabled: boolean;       // 是否启用
  paths: string[];        // 监控路径列表
  filePatterns: string[]; // 文件匹配模式
  scanInterval?: number;  // 扫描间隔（ms）
}

abstract class BaseCollector {
  abstract name: string;
  abstract source: string;

  // 发现对话文件
  abstract discoverFiles(): Promise<string[]>;

  // 解析单个对话文件
  abstract parseFile(filePath: string): Promise<ParsedConversation | null>;

  // 判断文件是否有更新
  abstract hasUpdated(filePath: string, lastSync: string): boolean;

  // 获取默认监控路径
  abstract getDefaultPaths(): string[];
}

interface ParsedConversation {
  source: string;
  title?: string;
  messages: Message[];
  metadata: {
    filePath: string;
    workspace?: string;
    project?: string;
    tool?: string;
    toolVersion?: string;
  };
  createdAt: string;
  updatedAt: string;
}
```

3. 重构已有采集器（Claude Code、Codex）使用此框架
4. 创建采集器注册表
5. 实现自动发现已安装的工具

---

### 需求 0.2.12：工具自动发现

**描述：** 自动检测用户系统中安装了哪些 AI 工具

**验收标准：**
- 应用首次启动时扫描已安装的 AI 工具
- 检测 Cursor、Kiro、Trea、Claude Code、Codex 是否安装
- 自动配置已安装工具的监控路径
- 未安装的工具标记为"未检测到"
- 用户可手动添加/修改路径

**具体步骤：**
1. 创建 `electron/services/collector/detector.ts`
2. 实现各工具的检测逻辑：

```typescript
// 检测逻辑
const toolDetectors = {
  'claude-code': {
    check: () => fs.existsSync(path.join(os.homedir(), '.claude')),
    paths: () => [path.join(os.homedir(), '.claude/projects')],
  },
  'cursor': {
    check: () => {
      const appSupport = path.join(os.homedir(), 'Library/Application Support/Cursor');
      return fs.existsSync(appSupport);
    },
    paths: () => [path.join(os.homedir(), 'Library/Application Support/Cursor')],
  },
  'kiro': {
    check: () => fs.existsSync(path.join(os.homedir(), '.kiro')),
    paths: () => [path.join(os.homedir(), '.kiro')],
  },
  'trea': {
    check: () => {
      const appSupport = path.join(os.homedir(), 'Library/Application Support/Trea');
      return fs.existsSync(appSupport);
    },
    paths: () => [path.join(os.homedir(), 'Library/Application Support/Trea')],
  },
  'codex': {
    check: () => fs.existsSync(path.join(os.homedir(), '.codex')),
    paths: () => [path.join(os.homedir(), '.codex')],
  },
};
```

3. 在首次启动引导流程中展示检测结果
4. 创建 `src/components/SetupWizard.tsx` - 首次启动向导

---

## 更新采集配置结构

```json
{
  "collector": {
    "enabled": true,
    "sources": {
      "claude-code": {
        "enabled": true,
        "detected": true,
        "paths": ["~/.claude/projects"],
        "mode": "realtime"
      },
      "codex": {
        "enabled": true,
        "detected": true,
        "paths": ["~/.codex"],
        "mode": "realtime"
      },
      "cursor": {
        "enabled": true,
        "detected": true,
        "paths": ["~/Library/Application Support/Cursor"],
        "mode": "realtime"
      },
      "kiro": {
        "enabled": true,
        "detected": true,
        "paths": ["~/.kiro"],
        "mode": "realtime"
      },
      "trea": {
        "enabled": false,
        "detected": false,
        "paths": [],
        "mode": "realtime"
      }
    },
    "notification": true,
    "scanIntervalMinutes": 5
  }
}
```

---

## 更新采集状态面板

采集状态面板需要展示所有工具的状态：

```
┌─────────────────────────────────────────────┐
│  采集状态                                    │
├─────────────────────────────────────────────┤
│  ● Claude Code    运行中   今日 +3          │
│  ● Codex          运行中   今日 +1          │
│  ● Cursor         运行中   今日 +5          │
│  ● Kiro           运行中   今日 +2          │
│  ○ Trea           未检测到                   │
│                                             │
│  [全量扫描]  [暂停全部]  [设置]             │
└─────────────────────────────────────────────┘
```
