# 01. 总体技术架构、项目结构、数据模型和 API

## 1. 总体架构

```text
采集端
  ├─ 手动粘贴
  ├─ 浏览器插件
  ├─ CLI Wrapper
  ├─ IDE 插件
  ├─ 文件导入
  └─ 手机分享导入

        ↓

本地 Ingestion Service
  ├─ 接收采集内容
  ├─ 标准化格式
  ├─ 敏感信息检测
  ├─ 去重
  └─ 保存原始记录

        ↓

本地知识处理引擎
  ├─ 摘要
  ├─ 分类
  ├─ 标签
  ├─ 模板选择
  ├─ 结构化笔记生成
  ├─ chunk 切分
  └─ embedding 向量化

        ↓

本地知识库
  ├─ SQLite 元数据
  ├─ SQLite FTS 全文搜索
  ├─ LanceDB / sqlite-vec 向量索引
  ├─ Markdown 文件
  ├─ 附件目录
  └─ 备份目录

        ↓

使用端
  ├─ 桌面客户端
  ├─ 命令行 aimem
  ├─ 本地 HTTP API
  ├─ MCP Server
  ├─ IDE 插件
  └─ 后期 Web / Mobile
```

---

## 2. 推荐技术选型

### 2.1 第一阶段推荐栈

| 模块 | 推荐技术 |
|---|---|
| 桌面端 | Tauri 2 |
| 前端 | React + TypeScript + Vite |
| UI | Tailwind CSS + shadcn/ui |
| 本地后端 | Rust |
| 数据库 | SQLite |
| ORM/SQL | sqlx 或 rusqlite |
| 全文搜索 | SQLite FTS5 |
| Markdown 渲染 | react-markdown / markdown-it |
| 模型接口 | OpenAI-compatible API + Ollama |
| 配置 | JSON/TOML |

### 2.2 第二阶段增加

| 模块 | 推荐技术 |
|---|---|
| 浏览器插件 | WXT + TypeScript |
| 插件标准 | WebExtensions / Manifest V3 |
| 插件通信 | localhost HTTP 或 Native Messaging |

### 2.3 第三阶段增加

| 模块 | 推荐技术 |
|---|---|
| 向量库 | LanceDB 或 sqlite-vec |
| RAG | 本地混合检索：FTS + Vector |
| Embedding | OpenAI-compatible / Ollama |

### 2.4 第四阶段增加

| 模块 | 推荐技术 |
|---|---|
| CLI | Rust clap |
| 终端交互 | portable-pty |
| MCP | Rust/Node MCP server |
| IDE 插件 | VS Code Extension API |

### 2.5 云端阶段增加

| 模块 | 推荐技术 |
|---|---|
| 后端 API | NestJS / FastAPI / Rust Axum |
| 数据库 | PostgreSQL |
| 向量 | pgvector / Qdrant |
| 对象存储 | S3 兼容存储 |
| 队列 | Redis + BullMQ / NATS |
| 身份认证 | Auth.js / Keycloak / 自研 OAuth |
| 后台 | Next.js Admin |

---

## 3. Monorepo 项目结构

建议从第一天就使用 monorepo，后面方便扩展插件、CLI、共享类型。

```text
ai-memory-hub/
  apps/
    desktop/                  # Tauri 桌面客户端
    browser-extension/        # 浏览器插件，V0.2 开始
    web-admin/                # 管理后台，V2/V3 开始
    mobile/                   # 手机端，后期

  packages/
    types/                    # 共享 TypeScript 类型
    core/                     # 前端可复用业务逻辑
    prompts/                  # 整理、分类、摘要 Prompt
    exporters/                # Markdown/HTML/PDF/Word 导出
    browser-adapters/         # ChatGPT/Claude/Gemini DOM 适配器
    sdk/                      # JS SDK，供插件/脚本调用本地 API

  crates/
    local-core/               # Rust 本地核心：数据库、搜索、文件、API
    aimem-cli/                # CLI 工具
    mcp-server/               # MCP 服务

  migrations/                 # SQLite 数据库迁移
  docs/                       # 产品、接口、开发文档
  scripts/                    # 构建、发布、调试脚本
  .github/workflows/          # CI/CD
  package.json
  pnpm-workspace.yaml
  README.md
```

---

## 4. 本地数据目录

跨平台数据目录建议：

```text
macOS:    ~/Library/Application Support/AI Memory Hub/
Windows:  %APPDATA%/AI Memory Hub/
Linux:    ~/.local/share/ai-memory-hub/
```

目录结构：

```text
AI Memory Hub/
  database/
    app.db
    app.db-wal
    app.db-shm
  vectors/
    lancedb/
  raw/
    conversations/
    imports/
  notes/
    markdown/
  attachments/
    images/
    files/
  exports/
  backups/
  logs/
  config/
    app.toml
    providers.toml
```

---

## 5. 核心数据模型

### 5.1 Conversation

```ts
type Conversation = {
  id: string
  sourceType: 'manual' | 'browser' | 'cli' | 'ide' | 'file' | 'mobile'
  sourceName?: string
  title?: string
  url?: string
  status: 'raw' | 'processing' | 'processed' | 'confirmed' | 'archived'
  rawHash?: string
  metadata?: Record<string, unknown>
  createdAt: string
  updatedAt: string
}
```

### 5.2 Message

```ts
type Message = {
  id: string
  conversationId: string
  role: 'user' | 'assistant' | 'system' | 'tool'
  content: string
  seq: number
  metadata?: Record<string, unknown>
  createdAt: string
}
```

### 5.3 Note

```ts
type Note = {
  id: string
  title: string
  type: 'qa' | 'bugfix' | 'prd' | 'learning' | 'prompt' | 'workflow' | 'decision' | 'creative'
  category?: string
  summary?: string
  bodyMd: string
  status: 'draft' | 'confirmed' | 'archived'
  createdAt: string
  updatedAt: string
}
```

### 5.4 Tag

```ts
type Tag = {
  id: string
  name: string
  type: 'user' | 'auto' | 'system'
}
```

### 5.5 Chunk

```ts
type Chunk = {
  id: string
  noteId: string
  content: string
  chunkIndex: number
  hash: string
  vectorId?: string
  createdAt: string
}
```

---

## 6. SQLite 表设计

### 6.1 conversations

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  source_type TEXT NOT NULL,
  source_name TEXT,
  title TEXT,
  url TEXT,
  status TEXT NOT NULL,
  raw_hash TEXT,
  metadata_json TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 6.2 messages

```sql
CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL,
  seq INTEGER NOT NULL,
  metadata_json TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY (conversation_id) REFERENCES conversations(id)
);
```

### 6.3 notes

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  type TEXT NOT NULL,
  category TEXT,
  summary TEXT,
  body_md TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 6.4 note_sources

```sql
CREATE TABLE note_sources (
  id TEXT PRIMARY KEY,
  note_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  message_id TEXT,
  FOREIGN KEY (note_id) REFERENCES notes(id),
  FOREIGN KEY (conversation_id) REFERENCES conversations(id),
  FOREIGN KEY (message_id) REFERENCES messages(id)
);
```

### 6.5 tags / note_tags

```sql
CREATE TABLE tags (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  type TEXT NOT NULL
);

CREATE TABLE note_tags (
  note_id TEXT NOT NULL,
  tag_id TEXT NOT NULL,
  PRIMARY KEY (note_id, tag_id),
  FOREIGN KEY (note_id) REFERENCES notes(id),
  FOREIGN KEY (tag_id) REFERENCES tags(id)
);
```

### 6.6 chunks

```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  note_id TEXT NOT NULL,
  content TEXT NOT NULL,
  chunk_index INTEGER NOT NULL,
  hash TEXT,
  vector_id TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY (note_id) REFERENCES notes(id)
);
```

### 6.7 FTS 表

```sql
CREATE VIRTUAL TABLE notes_fts USING fts5(
  title,
  summary,
  body_md,
  content='notes',
  content_rowid='rowid'
);
```

---

## 7. 本地 API 设计

本地客户端可以启动一个只监听 `127.0.0.1` 的 API 服务。

默认端口：

```text
127.0.0.1:37891
```

### 7.1 导入对话

```http
POST /api/conversations
Content-Type: application/json

{
  "sourceType": "manual",
  "sourceName": "ChatGPT",
  "title": "用户调研报告讨论",
  "messages": [
    {"role": "user", "content": "怎么写用户调研报告？"},
    {"role": "assistant", "content": "可以按以下步骤..."}
  ]
}
```

### 7.2 生成笔记

```http
POST /api/conversations/{id}/generate-note

{
  "template": "auto",
  "language": "zh-CN"
}
```

### 7.3 搜索

```http
GET /api/search?q=登录失败&type=bugfix&limit=10
```

### 7.4 获取笔记

```http
GET /api/notes/{id}
```

### 7.5 知识库问答

```http
POST /api/ask

{
  "query": "我之前怎么解决登录失败的问题？",
  "topK": 8,
  "withSources": true
}
```

---

## 8. 安全设计

### 8.1 本地 API 安全

- 默认只监听 `127.0.0.1`；
- 首次连接需要 token；
- 浏览器插件需要配对；
- 所有请求加本地授权头：`Authorization: Bearer local_xxx`；
- 用户可以关闭本地 API。

### 8.2 敏感信息检测

第一版可以检测：

- 邮箱；
- 手机号；
- 身份证；
- API Key；
- Token；
- 私钥；
- URL 中的敏感 query。

### 8.3 数据加密

路线：

```text
V0.1: 不加密，但明确数据在本地
V1.0: 支持笔记库加密
V2.0: 云同步端到端加密
```

---

## 9. Prompt 输出协议

让模型输出 JSON，不要直接输出 Markdown 字符串后再猜。

```json
{
  "title": "string",
  "type": "qa | bugfix | prd | learning | prompt | workflow | decision | creative",
  "category": "string",
  "summary": "string",
  "tags": ["string"],
  "confidence": 0.92,
  "markdown": "# 标题

## 问题
...",
  "sourceMessageIds": ["msg_1", "msg_2"],
  "needUserConfirm": true
}
```

---

## 10. 模块边界

| 模块 | 职责 |
|---|---|
| Ingestion | 接收原始内容 |
| Parser | 解析对话结构 |
| Sanitizer | 脱敏与隐私处理 |
| NoteGenerator | 调模型生成笔记 |
| Classifier | 分类和标签 |
| Storage | SQLite / 文件保存 |
| Search | 关键词和全文搜索 |
| RAG | 向量检索和问答 |
| Exporter | 导出 Markdown/HTML/PDF/Word |
| Sync | 后期多端同步 |
| Admin | 后期后台管理 |
