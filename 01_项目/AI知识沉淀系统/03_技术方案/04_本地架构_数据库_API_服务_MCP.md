
# 04. 本地架构：数据库、API、服务、MCP

## 1. 总体架构

```text
Desktop UI
  ↓ Tauri command
Rust Local Core
  ↓
SQLite + Files
  ↑
Local HTTP API 127.0.0.1:37891
  ↑        ↑          ↑
aimem CLI  Browser    IDE Plugin
           Extension

MCP Server 可由 aimem 启动，调用同一套 Local API / Local Core
```

---

## 2. 第一阶段为什么需要本地 API

因为 CLI、浏览器插件、IDE 插件都需要和知识库通信。

如果只有 Tauri command，只有桌面 UI 能用。  
所以 V0.1 就建议做一个本地 API。

V0.1 简化：

```text
桌面客户端启动时启动 Local API
CLI 必须在客户端启动时使用
```

V0.2 增强：

```text
独立 background service
CLI 可以拉起服务
```

---

## 3. 本地目录结构

```text
~/Library/Application Support/AI Knowledge Workspace/
  database/
    app.db
  notes/
    markdown/
  raw/
    conversations/
    codex-sessions/
  projects/
    project-index.json
  attachments/
  exports/
  vectors/
  backups/
  logs/
  config/
    app.toml
```

---

## 4. SQLite 核心表

### 4.1 notes

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  type TEXT NOT NULL,
  project TEXT,
  category_id TEXT,
  summary TEXT,
  body_md TEXT NOT NULL,
  status TEXT NOT NULL,
  favorite INTEGER NOT NULL DEFAULT 0,
  source_type TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

### 4.2 tags / note_tags

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

### 4.3 categories

```sql
CREATE TABLE categories (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0
);
```

### 4.4 conversations / messages

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,
  source_type TEXT NOT NULL, -- manual, codex, browser, ide, cli
  source_name TEXT,
  title TEXT,
  url TEXT,
  project TEXT,
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

### 4.5 codex_sessions

```sql
CREATE TABLE codex_sessions (
  id TEXT PRIMARY KEY,
  conversation_id TEXT,
  project TEXT,
  cwd TEXT,
  git_branch TEXT,
  git_diff_before TEXT,
  git_diff_after TEXT,
  command TEXT,
  status TEXT NOT NULL,
  summary TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
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
  enabled INTEGER NOT NULL DEFAULT 1,
  default_for_chat INTEGER NOT NULL DEFAULT 0,
  default_for_organize INTEGER NOT NULL DEFAULT 0,
  default_for_embedding INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

---

## 5. V0.2 RAG 表

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

CREATE TABLE embedding_jobs (
  id TEXT PRIMARY KEY,
  chunk_id TEXT NOT NULL,
  status TEXT NOT NULL,
  error TEXT,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

---

## 6. 本地 API 设计

### 6.1 Health

```http
GET /api/health
```

返回：

```json
{
  "ok": true,
  "version": "0.1.0",
  "database": "connected"
}
```

### 6.2 Notes

```http
GET /api/notes
GET /api/notes/{id}
POST /api/notes
PATCH /api/notes/{id}
DELETE /api/notes/{id}
GET /api/search?q=xxx
```

### 6.3 Ask

```http
POST /api/ask

{
  "query": "我之前怎么解决登录失败？",
  "scope": "all",
  "project": "my-app",
  "topK": 8
}
```

### 6.4 Context for Codex

```http
POST /api/context

{
  "query": "登录失败 cookie token",
  "project": "my-app",
  "format": "codex"
}
```

返回 Markdown。

### 6.5 Import

```http
POST /api/import/text
POST /api/import/file
POST /api/import/codex-session
```

---

## 7. 本地 API 安全

必须做：

```text
只监听 127.0.0.1
首次 CLI 使用生成 local token
token 存在用户目录
插件需要配对
允许用户关闭 API
```

V0.1 可以简化，但不能完全裸奔。

---

## 8. MCP 设计

V0.3 做。

MCP 工具：

```text
knowledge_search(query, project, tags, limit)
knowledge_get(note_id)
knowledge_ask(query, scope, project)
knowledge_create_note(title, body_md, tags, project)
project_context_get(cwd)
codex_session_save(transcript, metadata)
```

MCP Server 通过 Local Core 或 Local API 访问数据库。

---

## 9. 技术选型

```text
Desktop: Tauri 2 + React + TypeScript
Rust local core: database, file, API, model calls
SQLite: metadata + FTS
Vector: V0.2 可先 sqlite-vec 或 LanceDB
CLI: Rust clap
Local API: Rust axum
MCP: Rust or Node，建议先 Rust 集成 CLI
```
