
# 07. 数据模型、API 与本地架构

## 1. 本地优先架构

第一阶段应该坚持本地优先。

```text
桌面客户端 UI
  ↓ Tauri Commands
Rust 本地核心
  ↓
SQLite 数据库
  ↓
本地文件系统
  ↓
可选本地 API / CLI / 插件
```

---

## 2. 本地目录结构

跨平台目录：

```text
macOS:    ~/Library/Application Support/AI Knowledge Workspace/
Windows:  %APPDATA%/AI Knowledge Workspace/
Linux:    ~/.local/share/ai-knowledge-workspace/
```

内部结构：

```text
AI Knowledge Workspace/
  database/
    app.db
    app.db-wal
    app.db-shm
  notes/
    markdown/
  raw/
    conversations/
  attachments/
    images/
    files/
  exports/
  vectors/
  backups/
  logs/
  config/
    app.toml
    providers.toml
```

---

## 3. 核心数据模型

### 3.1 Note

```ts
type Note = {
  id: string
  title: string
  type: 'qa' | 'bugfix' | 'learning' | 'prd' | 'prompt' | 'work' | 'creative' | 'decision'
  categoryId?: string
  summary?: string
  bodyMd: string
  status: 'draft' | 'pending_confirm' | 'confirmed' | 'archived' | 'deleted'
  favorite: boolean
  createdAt: string
  updatedAt: string
}
```

### 3.2 Category

```ts
type Category = {
  id: string
  name: string
  parentId?: string
  sortOrder: number
}
```

### 3.3 Tag

```ts
type Tag = {
  id: string
  name: string
  type: 'user' | 'auto' | 'system'
}
```

### 3.4 Conversation

```ts
type Conversation = {
  id: string
  sourceType: 'manual' | 'browser' | 'cli' | 'ide' | 'file'
  sourceName?: string
  title?: string
  url?: string
  status: 'raw' | 'processed' | 'converted' | 'discarded'
  rawHash?: string
  metadata?: Record<string, unknown>
  createdAt: string
  updatedAt: string
}
```

### 3.5 Message

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

### 3.6 AI Chat Session

右侧 AI 对话也要保存。

```ts
type AiChatSession = {
  id: string
  title: string
  scope: 'none' | 'current_note' | 'category' | 'tag' | 'all'
  scopeRefId?: string
  modelProviderId: string
  createdAt: string
  updatedAt: string
}
```

```ts
type AiChatMessage = {
  id: string
  sessionId: string
  role: 'user' | 'assistant'
  content: string
  sources?: SourceRef[]
  createdAt: string
}
```

---

## 4. SQLite 表设计

### 4.1 notes

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  type TEXT NOT NULL,
  category_id TEXT,
  summary TEXT,
  body_md TEXT NOT NULL,
  status TEXT NOT NULL,
  favorite INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 4.2 categories

```sql
CREATE TABLE categories (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0
);
```

### 4.3 tags / note_tags

```sql
CREATE TABLE tags (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  type TEXT NOT NULL
);

CREATE TABLE note_tags (
  note_id TEXT NOT NULL,
  tag_id TEXT NOT NULL,
  PRIMARY KEY(note_id, tag_id)
);
```

### 4.4 conversations / messages

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

CREATE TABLE messages (
  id TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL,
  seq INTEGER NOT NULL,
  metadata_json TEXT,
  created_at TEXT NOT NULL
);
```

### 4.5 note_sources

```sql
CREATE TABLE note_sources (
  id TEXT PRIMARY KEY,
  note_id TEXT NOT NULL,
  conversation_id TEXT NOT NULL,
  message_id TEXT
);
```

### 4.6 model_providers

```sql
CREATE TABLE model_providers (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  provider_type TEXT NOT NULL,
  base_url TEXT,
  api_key_encrypted TEXT,
  chat_model TEXT,
  embedding_model TEXT,
  rerank_model TEXT,
  enabled INTEGER NOT NULL DEFAULT 1,
  default_for_chat INTEGER NOT NULL DEFAULT 0,
  default_for_organize INTEGER NOT NULL DEFAULT 0,
  default_for_embedding INTEGER NOT NULL DEFAULT 0,
  metadata_json TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 4.7 ai_chat_sessions / ai_chat_messages

```sql
CREATE TABLE ai_chat_sessions (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  scope TEXT NOT NULL,
  scope_ref_id TEXT,
  model_provider_id TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE ai_chat_messages (
  id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL,
  sources_json TEXT,
  created_at TEXT NOT NULL
);
```

---

## 5. Tauri Command API

### 5.1 笔记

```text
list_notes(filter)
get_note(id)
create_note(input)
update_note(input)
delete_note(id)
favorite_note(id, favorite)
search_notes(query, filters)
```

### 5.2 分类和标签

```text
list_categories()
create_category(input)
update_category(input)
delete_category(id)

list_tags()
create_tag(input)
rename_tag(id, name)
merge_tags(sourceId, targetId)
```

### 5.3 AI 对话

```text
list_model_providers()
create_model_provider(input)
update_model_provider(input)
test_model_provider(id)

chat_with_ai(input)
summarize_note(noteId)
append_ai_answer_to_note(noteId, answer)
```

### 5.4 导入

```text
create_conversation(input)
list_conversations(status)
generate_note_from_conversation(conversationId)
convert_conversation_to_note(conversationId, draft)
```

### 5.5 导出

```text
export_note(noteId, format)
export_notes(filter, format)
```

---

## 6. 本地 HTTP API，供插件和 CLI 使用

桌面客户端可启动本地 API。

地址：

```text
http://127.0.0.1:37891
```

### 6.1 健康检查

```http
GET /api/health
```

### 6.2 搜索

```http
GET /api/search?q=登录失败
```

### 6.3 获取笔记

```http
GET /api/notes/{id}
```

### 6.4 AI 问答

```http
POST /api/ask

{
  "query": "我之前怎么解决登录失败的问题？",
  "scope": "all",
  "topK": 8
}
```

### 6.5 插件导入

```http
POST /api/ingest/browser-conversation
```

---

## 7. 安全设计

### 7.1 本地 API 安全

- 只监听 127.0.0.1；
- 插件首次连接需要配对；
- 使用本地 token；
- 用户可以关闭本地 API；
- 所有调用记录日志。

### 7.2 API Key 安全

- 不明文写入普通配置文件；
- macOS 使用 Keychain；
- Windows 使用 Credential Manager；
- Linux 使用 Secret Service，可后期；
- MVP 可以先加密保存，但要明确风险。

### 7.3 敏感信息检测

检测：

```text
手机号
邮箱
身份证
API Key
Token
私钥
数据库连接串
URL query 中的 token
```

---

## 8. 后续 RAG 扩展

V0.2 新增：

```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  note_id TEXT NOT NULL,
  content TEXT NOT NULL,
  chunk_index INTEGER NOT NULL,
  hash TEXT,
  vector_id TEXT,
  token_count INTEGER,
  created_at TEXT NOT NULL
);
```

流程：

```text
笔记保存
  ↓
切 chunk
  ↓
生成 embedding
  ↓
写入向量索引
  ↓
AI 问答时检索相关 chunk
  ↓
带来源回答
```
