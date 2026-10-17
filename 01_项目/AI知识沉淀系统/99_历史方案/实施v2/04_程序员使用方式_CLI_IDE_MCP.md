
# 04. 程序员使用方式：CLI、IDE、MCP

## 1. 程序员为什么需要额外入口

程序员不会一直打开桌面客户端。

很多时候他们在：

```text
终端
VS Code / Cursor
Codex
Claude Code
Git 仓库
CI 日志
报错堆栈
```

所以程序员需要：

```text
命令行里能查知识库
IDE 里能保存代码和报错
AI 编程工具能通过 MCP 调用知识库
```

---

## 2. CLI 工具：aimem

命令名称：

```bash
aimem
```

它连接本地桌面客户端的数据库或本地 API。

---

## 3. 常用命令设计

### 3.1 搜索知识库

```bash
aimem search "登录失败"
```

输出：

```text
找到 3 条相关知识：

[1] 登录失败排查流程
    ID: note_abc123
    类型：Bug 修复
    标签：#auth #cookie #bugfix
    摘要：记录 token、cookie、401 的排查方式。

[2] OAuth 回调失败问题
    ID: note_def456
    类型：Bug 修复
    标签：#OAuth #callback

使用 aimem show note_abc123 查看详情
```

### 3.2 查看某条笔记

```bash
aimem show note_abc123
```

输出完整 Markdown。

```bash
aimem show note_abc123 --plain
aimem show note_abc123 --json
aimem show note_abc123 --copy
```

### 3.3 对知识库提问

```bash
aimem ask "我之前怎么解决登录失败的问题？"
```

输出：

```text
根据你的知识库，之前主要从以下方向排查：

1. Cookie 是否写入成功
2. SameSite 是否配置正确
3. Token 是否过期
4. 后端是否返回 401
5. 前端路由守卫是否误判

来源：
- 登录失败排查流程
- OAuth 回调失败问题
```

### 3.4 把知识库内容给其他 AI CLI

```bash
aimem ask "整理我之前所有登录失败的排查经验" --format md > context.md
codex "参考 context.md，帮我排查当前项目登录失败" < context.md
```

或者：

```bash
aimem show note_abc123 --plain | claude "把这条经验应用到当前项目"
```

### 3.5 导入文件

```bash
aimem import ./debug-notes.md --type bugfix --tag auth
```

### 3.6 导出笔记

```bash
aimem export note_abc123 --format md --out ./login-debug.md
```

---

## 4. CLI 完整命令清单

```text
aimem search <query>
aimem show <note_id>
aimem ask <query>
aimem import <file>
aimem export <note_id>
aimem new
aimem list
aimem tag
aimem category
aimem run <command>
aimem mcp serve
aimem status
aimem config
aimem doctor
```

---

## 5. aimem run：采集终端 AI 会话

程序员使用 Codex / Claude Code / Gemini CLI 时，可以通过 wrapper 启动：

```bash
aimem run codex
aimem run claude
aimem run gemini
```

系统记录：

```text
命令
当前目录
Git 分支
Git diff before
用户输入
AI 输出
终端输出
Git diff after
开始时间
结束时间
```

生成技术笔记：

```md
# 问题名称

## 背景

## 问题现象

## 排查过程

## 最终解决方案

## 代码变更

## 验证方式

## 经验沉淀

## 相关文件
```

---

## 6. IDE 插件

支持 VS Code / Cursor。

命令：

```text
AI Memory: Search Knowledge Base
AI Memory: Save Selected Code
AI Memory: Save Current Error
AI Memory: Generate Bugfix Note
AI Memory: Attach Git Diff
AI Memory: Ask With Current File
```

### 6.1 保存选中代码

采集：

```text
文件路径
语言
选中代码
当前项目路径
Git branch
Git diff
用户补充说明
```

### 6.2 保存报错

采集：

```text
错误信息
堆栈
当前文件
相关代码
终端输出
用户描述
```

---

## 7. MCP Server

MCP 的作用：

> 让 Codex、Claude Code、Cursor 等 AI 编程工具主动调用你的知识库。

启动：

```bash
aimem mcp serve
```

提供工具：

```text
knowledge_search(query, filters, limit)
knowledge_get(note_id)
knowledge_ask(query, scope)
knowledge_create_note(title, markdown, tags)
project_context_save(cwd, git_diff, problem)
recent_notes(limit)
```

### 7.1 使用场景

用户在 AI 编程工具里说：

```text
先查一下我的知识库里有没有类似登录失败的问题。
```

AI 调用：

```text
knowledge_search("登录失败")
```

然后结合历史经验排查当前项目。

---

## 8. 程序员版开发优先级

### V0.4 必做

- [ ] aimem search；
- [ ] aimem show；
- [ ] aimem ask；
- [ ] aimem import；
- [ ] aimem export；
- [ ] `--json` 输出；
- [ ] `--copy` 复制；
- [ ] 本地 API 连接；
- [ ] MCP Server 初版；
- [ ] VS Code 插件搜索和保存选中代码。

### V1.0 增强

- [ ] aimem run；
- [ ] Git diff 采集；
- [ ] 自动生成 Bug 修复笔记；
- [ ] Cursor 插件；
- [ ] AI 编程工具自动调用知识库。
