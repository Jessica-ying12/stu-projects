
# 01. 最终产品形态：桌面客户端 + CLI + Codex + 插件

## 1. 最终不是一个 App，而是一套本地知识系统

最终产品形态：

```text
桌面客户端：知识仓库主界面
CLI：开发者终端入口
本地 API：连接客户端、CLI、插件、MCP
MCP Server：让 AI 编程工具主动调用知识库
浏览器插件：采集网页 AI 对话
IDE 插件：采集代码上下文、错误、diff
后期云端：同步、团队、分享、后台
```

---

## 2. 桌面客户端负责什么

桌面客户端是核心中枢。

它负责：

```text
查看知识
编辑知识
分类标签
管理原始记录
右侧 AI 对话
模型配置
搜索
RAG 问答
导出
备份
插件连接状态
本地 API 开关
```

桌面客户端是长期打开的知识工作台，不是导入工具。

---

## 3. CLI 负责什么

CLI 名称：

```bash
aimem
```

CLI 负责开发者终端工作流：

```text
搜索知识库
查看笔记
向知识库提问
生成 Codex 上下文
保存终端输出为笔记
包装运行 Codex 并采集会话
启动 MCP Server
导入/导出笔记
```

CLI 不替代客户端。

关系是：

```text
客户端负责管理和编辑
CLI 负责快速调用和开发流集成
```

---

## 4. Codex 工作流中怎么用

### 4.1 查询经验

```bash
aimem search "登录失败"
```

### 4.2 让知识库回答

```bash
aimem ask "我之前怎么解决登录失败的问题？"
```

### 4.3 生成上下文给 Codex

```bash
aimem context "登录失败 cookie token" --out AIMEM_CONTEXT.md
codex "参考 AIMEM_CONTEXT.md，排查当前项目" < AIMEM_CONTEXT.md
```

### 4.4 一条命令增强 Codex

后续可以做：

```bash
aimem codex "帮我排查当前项目登录失败"
```

内部执行：

```text
1. 分析用户问题
2. 搜索知识库
3. 获取相关笔记
4. 生成上下文 Prompt
5. 调用 codex
6. 可选保存 Codex 对话和 git diff
```

### 4.5 包装运行 Codex 并自动保存

```bash
aimem run codex
```

它会记录：

```text
当前目录
Git 分支
Git diff before
Codex 输入输出
Git diff after
最终摘要
可选生成 Bug 修复笔记
```

---

## 5. 本地 API 负责连接所有入口

本地客户端启动时可以开启服务：

```text
http://127.0.0.1:37891
```

连接对象：

```text
CLI aimem
浏览器插件
IDE 插件
MCP Server
脚本
```

注意：

```text
默认只监听 127.0.0.1
需要本地 token
用户可以关闭
```

---

## 6. MCP Server 负责什么

MCP 是后期核心能力。

启动：

```bash
aimem mcp serve
```

AI 编程工具可以调用：

```text
knowledge_search
knowledge_get
knowledge_ask
knowledge_create_note
project_context_save
recent_bugfix_notes
```

这样 Codex / Claude Code / Cursor 就可以主动查你的知识库。

---

## 7. 浏览器插件负责什么

浏览器插件后置到 V0.4。

原因：

```text
你的当前高频场景是命令行 Codex
网页采集重要，但不是第一需求
```

插件后续负责：

```text
采集 ChatGPT / Claude / Gemini 网页对话
保存选中内容
发送到本地客户端
进入待整理区
```

---

## 8. IDE 插件负责什么

IDE 插件适合 V0.3。

支持：

```text
保存选中代码
保存当前报错
保存终端输出
附带当前文件路径
附带 Git diff
搜索知识库
让当前文件带着知识库上下文问 AI
```

---

## 9. 一句话架构

```text
Desktop App 是脑子
SQLite 是记忆
aimem CLI 是手
Local API 是神经
Codex/MCP/插件是外部感官和工具
```
