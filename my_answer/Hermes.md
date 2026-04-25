# 记忆和上下文管理机制
分为四层。

## Session




## 第一层：MEMORY.md（Agent 的笔记本） 和 USER.md（用户画像）

- 位置：~/.hermes/memories/MEMORY.md
- 容量：~2,200 字符 / ~800 tokens
- 内容：
    - 环境
    - 经验
    - 项目已经做了的工作/项目约定

- 典型条目数：8-15 条


- 位置：~/.hermes/memories/USER.md
- 容量：~1,375 字符 / ~500 tokens
- 内容：【与用户相关的情报】用户偏好、沟通风格、工作习惯、技术水平
- 典型条目数：5-10 条


### 如何触发存储

- 第一层：被动。上下文中写了要保存。
memory 工具注册在 Agent 的工具列表中，和 terminal、browser 等工具并列。
【add、replace、remove】三个工具。

LLM在任意一轮推理中都可以选择调用它。同时，系统提示中有一段行为指导，大意是：
 "Save durable facts using the memory tool: user preferences, environment details, tool quirks, and stable conventions."

- 第二层：主动提醒——Nudge 机制：每隔一定对话轮次提醒。
Hermes 有一个 nudge_interval 配置项，实现了定期提醒：

Agent 内部维护一个计数器: _turns_since_memory

每轮对话后: _turns_since_memory += 1

当 _turns_since_memory >= nudge_interval 时:
    → 注入一条系统级消息，提醒 Agent 回顾并保存记忆
    → 计数器归零
也就是说，每隔 N 轮对话，系统会主动往对话中插入一条系统消息，问 Agent："回顾一下刚才发生了什么，有没有值得持久化的知识？"

注： nudge定期检查是会spawn一个独立的后台agent来执行的。不影响主对话流程。拿到主对话的消息快照和memory store。

- 三、条件触发
会话结束时、每15次工具调用时、模式检测到3+次类似请求时、用户主动要求时

### 格式：
字符串形式。用 §（section sign）作为条目分隔符
User's project is a Rust web service at ~/code/myapi using Axum + SQLx
§
This machine runs Ubuntu 22.04, has Docker and Podman installed
§

### 达到上限怎么办？
memory tool里面没有主动处理的机制。超限只会返回“爆了”。
整理工作又交回给agent判断了。
他会自己“判断”怎么合并...直接在主agent里面做。

只有nudge和 会话结束时会主动spawn一个子agent做记忆存储。


## 第二层：Session History（会话历史）

### 存储什么
（当前这个session）所有对话的完整记录。

### 格式
双线并行：
~/.hermes/sessions/*.jsonl 
jsonl文件。读写比较快，给运行时用的。

SQLite 数据库(~/.hermes/state.db)
用于检索。提供了结构化查询，全文索引，统计聚合等能力。
三个核心结构：
- sessions 表：会话元数据
    元信息 + 统计数据
- messages 表：所有消息记录
    逐条记录
- FTS5 虚拟表：全文搜索索引（建立在 messages 的 content 上）
    是一个虚拟表。
    其实是SQLite内建的一个全文搜索引擎，维护的是一个倒排索引：
```
  CREATE VIRTUAL TABLE messages_fts USING fts5(
      content,                -- 索引 messages.content 字段
      content=messages,       -- 数据来源是 messages 表（外部内容模式）
      content_rowid=id        -- 用 messages.id 做 rowid 映射
  );
```
普通表:  消息ID → 消息内容
FTS5:    关键词 → [消息ID1, 消息ID2, 消息ID3, ...]

### 什么时候触发存储
每轮对话时自动触发存储。


### 检索！
只有唯一一个session_search 工具。
参数：
query
role_filter
limit ：返回会话（摘要）数限制

具体执行： 直接传给SQLite FTS5执行。
FTS5 维护一个倒排索引。其内置排名算法会考虑：
词频、文档频率、文档长度。
同时Hermes的FTS5 还用了porter tokenizer，做词干提取，增加英文上下文的命中程度。

支持：
docker deployment              → 包含这些词的消息
"docker networking"            → 精确短语匹配
elevenlabs OR baseten          → 布尔 OR
python NOT java                → 排除
deploy*                        → 前缀匹配


详细流程：
① FTS5 搜索
    query 直接扔给 SQLite FTS5，取 top 50 条匹配消息
        │
② 去重 & 分组
    按 session_id 分组，沿 parent_session_id 链解析委托关系
    排除当前会话，取 top N 个不重复的会话（N = limit）【返回的是一个message id 在哪个session以及在这个session中的message id】
        │
③ 加载对话文本
    每个命中的 session，从 DB 取完整消息列表，格式化为可读文本
        │
④ 截断到 ~10 万字符
    以匹配位置为中心截取窗口（25% 在前，75% 在后）
    匹配策略按优先级：
    1. 完整短语命中
    2. 所有词在 200 字符内共现
    3. 任意单词命中
    选覆盖最多匹配点的窗口位置
        │
⑤ 并行 LLM 摘要（最多 3 路并发）
    每个 session 的截断文本 + 搜索主题 → 发给辅助模型
    temperature=0.1，max_tokens=10,000
    提示词要求：总结做了什么、结论是什么、保留具体命令/路径/错误信息
        │
⑥ 组装返回
    每个 session 返回: session_id, when, source, model, summary
    摘要失败时降级为原始文本前 500 字符预览



### 一些有意思的机制
resume时 recap会自动精简
  恢复时不是把全部历史灌进来，而是显示一个精简回顾面板：
  - 用户消息截断到 300 字符，Assistant 截断到 200 字符或 3 行
  - 工具调用折叠为计数：[3 tool calls: terminal, web_search]
  - 隐藏 system 消息和工具原始结果
  - 最多显示最近 10 轮对话

压缩时自动创建一个新session。并且标题按继承链连接住，存在sessions.json中。




## 第四层：Memory Providers（外部记忆插件）
同一时间只能激活一个外部memory provider
每个provider 注册自己的工具到agent的工具列表里，这样就可以主动调用了。


- 可选插件：Honcho、OpenViking、Mem0、Hindsight、Holographic、RetainDB、ByteRover、Su
permemory（共 8 种）
- 运行方式：与内置记忆并行运行，不替换
- 增强能力：知识图谱、语义搜索、自动事实抽取、跨会话用户建模

类比：这是可插拔的外挂大脑。内置记忆是基础款，外部插件提供高级检索能力。




┌─────────────────────────────────────────────┐
│           System Prompt（系统提示）            │
│  ┌───────────┐  ┌──────────┐                │
│  │ MEMORY.md │  │ USER.md  │  ← 冻结快照注入  │
│  │  ~800 tok │  │ ~500 tok │    每次会话固定   │
│  └───────────┘  └──────────┘                │
├─────────────────────────────────────────────┤
│         Session History (SQLite)             │
│         按需搜索，LLM 摘要召回                 │
├─────────────────────────────────────────────┤
│         Memory Providers (可选插件)           │
│         自动预取 + 会话同步 + 增强检索          │
└─────────────────────────────────────────────┘

关键设计：
- 第一、二层是常驻记忆，每次会话都注入，成本固定（~1,300 tokens）
- 第三层是按需召回，不占系统提示空间，搜索时才消耗 tokens
- 第四层是增强层，在前三层基础上叠加，永远不替代内置机制

这种分层的核心思路是：高频信息常驻（但有严格容量限制），低频信息按需检索（但无容量限
制）。想深入哪一层的细节？
