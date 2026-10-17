
# 03. CLI：aimem 命令设计与 Codex 工作流

## 1. CLI 是第一阶段核心，不是后期附属

因为你经常在命令行使用 Codex，所以 V0.1 就要有 CLI。

CLI 的目标：

```text
在不离开终端的情况下：
搜索知识、查看笔记、问知识库、生成 Codex 上下文、保存 Codex 输出。
```

---

## 2. CLI 名称

```bash
aimem
```

含义：AI Memory。

---

## 3. V0.1 必做命令

```bash
aimem status

aimem search <query>
aimem show <note_id>
aimem ask <query>
aimem context <query>
aimem import <file>
aimem save --stdin
aimem new
```

### 3.1 status

```bash
aimem status
```

输出：

```text
AI Memory Local Service: Online
API: http://127.0.0.1:37891
Database: ~/Library/Application Support/AI Knowledge Workspace/database/app.db
Default chat model: gpt-4.1
Default embedding model: not configured
MCP: Off
```

### 3.2 search

```bash
aimem search "登录失败"
```

输出：

```text
找到 3 条相关知识：

[1] note_01 登录失败排查流程
    类型：bugfix
    项目：project-a
    标签：#auth #cookie #bugfix
    摘要：记录 token、cookie、401 的排查方式。

[2] note_02 OAuth 回调失败问题
    类型：bugfix
    标签：#oauth #callback

使用 aimem show note_01 查看详情
```

支持：

```bash
aimem search "cookie" --type bugfix
aimem search "auth" --project my-app
aimem search "react" --tag frontend
aimem search "登录" --json
```

### 3.3 show

```bash
aimem show note_01
```

输出 Markdown。

支持：

```bash
aimem show note_01 --plain
aimem show note_01 --json
aimem show note_01 --copy
aimem show note_01 --out note.md
```

### 3.4 ask

```bash
aimem ask "我之前怎么解决 Next.js hydration 错误？"
```

V0.1 可以先用关键词搜索 + LLM 总结。  
V0.2 再升级为 RAG。

支持：

```bash
aimem ask "登录失败排查清单" --project my-app
aimem ask "总结我所有 auth 相关经验" --format md
aimem ask "找出最近一周的 bugfix" --from 2026-05-01
```

### 3.5 context

这是给 Codex 用的核心命令。

```bash
aimem context "登录失败 cookie token" --out AIMEM_CONTEXT.md
```

生成文件：

```md
# AI Memory Context

## 用户问题
登录失败 cookie token

## 相关历史经验

### 1. 登录失败排查流程
...

### 2. OAuth 回调失败问题
...

## 建议给 Codex 的任务说明
请结合以上历史经验，优先检查 cookie domain、SameSite、token 过期、401 处理。
```

然后：

```bash
codex "参考 AIMEM_CONTEXT.md，帮我排查当前项目登录失败" < AIMEM_CONTEXT.md
```

### 3.6 save --stdin

保存命令输出或 Codex 输出。

```bash
pbpaste | aimem save --stdin --type codex --tag codex --title "Codex 登录失败排查"
```

或者：

```bash
codex "解释这个错误" | tee codex.log
cat codex.log | aimem save --stdin --type codex
```

### 3.7 import

```bash
aimem import ./codex.log --type codex --project my-app
aimem import ./debug.md --type bugfix --tag auth
```

### 3.8 new

快速创建笔记。

```bash
aimem new --title "SQLite FTS5 查询经验" --type note --tag sqlite
```

---

## 4. V0.2 增强命令

```bash
aimem ask <query> --rag

aimem context <query> --top-k 8 --format codex

aimem codex <prompt>

aimem project index

aimem project context
```

### 4.1 aimem codex

一条命令完成知识检索 + Codex 调用：

```bash
aimem codex "帮我排查当前项目登录失败"
```

内部流程：

```text
1. 读取当前目录和 git branch
2. 根据 prompt 搜索知识库
3. 生成上下文文件
4. 调用 codex
5. 显示 Codex 输出
6. 询问是否保存本次会话
```

### 4.2 project context

```bash
aimem project context
```

生成当前项目上下文：

```text
项目路径
Git 分支
最近修改文件
README 摘要
package.json / pyproject.toml
相关历史笔记
```

---

## 5. V0.3 增强命令

```bash
aimem run codex

aimem mcp serve

aimem session list

aimem session summarize <session_id>
```

### 5.1 run codex

```bash
aimem run codex
```

作用：包装 Codex，自动记录完整会话。

记录：

```text
stdin/stdout/stderr
当前目录
Git branch
Git diff before
Git diff after
修改文件列表
用户确认的最终方案
```

### 5.2 mcp serve

```bash
aimem mcp serve
```

让 AI 工具调用知识库。

工具：

```text
knowledge_search
knowledge_get
knowledge_ask
knowledge_create_note
project_context_get
recent_bugfix_notes
```

---

## 6. 推荐 shell alias

等 CLI 稳定后，可以设置：

```bash
alias am='aimem'
alias ams='aimem search'
alias ama='aimem ask'
alias amc='aimem context'
```

可选：

```bash
alias codexm='aimem codex'
```

不要一开始强行 alias `codex=aimem run codex`，避免影响原始 Codex 使用。

---

## 7. CLI 和桌面客户端的关系

V0.1：CLI 调用桌面客户端本地 API。

```text
要求桌面客户端启动
CLI → http://127.0.0.1:37891 → SQLite
```

V0.2：增加 headless local service。

```text
即使桌面客户端没打开，CLI 也能启动本地服务或直连数据库
```

V0.3：CLI 可以启动 MCP。

---

## 8. CLI 实现建议

推荐 Rust：

```text
clap：命令行解析
reqwest：调用本地 API
serde：JSON
comfy-table：表格输出
arboard：复制剪贴板
portable-pty：后期包装 Codex
```

V0.1 可以先不做 PTY，只调用本地 API。
