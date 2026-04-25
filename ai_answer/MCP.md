  一、MCP（Model Context Protocol）

  1. 是什么

  Anthropic 于 2024 年 11 月 开源的「LLM ↔ 外部系统」通信协议，目标是把 AI 应用与数据源/工具之间"一对一自定义集成"的
  M×N 问题变成 M+N 问题——大家都实现同一套协议即可互通。被称为"AI 应用界的 USB-C"。

  2025 年 12 月，Anthropic 已将 MCP 捐赠给 Agentic AI Foundation（Linux Foundation 旗下），目前
  OpenAI、Google、Microsoft 等都已原生支持。

  2. 架构组成（面试重点）

  Client-Server 架构，三个角色：

  ┌────────┬────────────────────────────────────────────────────┬─────────────────────────────────────┐
  │  角色  │                        说明                        │                例子                 │
  ├────────┼────────────────────────────────────────────────────┼─────────────────────────────────────┤
  │ Host   │ 运行 LLM 的应用本体，持有会话和用户授权            │ Claude Desktop、Claude Code、Cursor │
  ├────────┼────────────────────────────────────────────────────┼─────────────────────────────────────┤
  │ Client │ Host 内部的协议客户端，与单个 Server 维持 1:1 连接 │ SDK 封装层                          │
  ├────────┼────────────────────────────────────────────────────┼─────────────────────────────────────┤
  │ Server │ 暴露能力的外部进程，可独立部署                     │ GitHub MCP Server、Slack MCP Server │
  └────────┴────────────────────────────────────────────────────┴─────────────────────────────────────┘

  通信层：基于 JSON-RPC 2.0，有状态会话（stateful），支持两种传输：
  - stdio（本地子进程）
  - Streamable HTTP（远程，支持 SSE 推送）

  Server 可暴露的三类原语（Primitives）：
  1. Tools — 可调用的函数（类似 function calling），有副作用
  2. Resources — 可读取的数据（文件、DB 记录等），只读
  3. Prompts — 预定义的提示模板，由用户主动触发

  Client 侧也能提供能力（反向）：
  - Sampling — Server 反过来让 Host 的 LLM 推理
  - Roots — 告诉 Server 有权访问哪些文件根
  - Elicitation（2025-06 新增）— Server 会话中途向用户请求输入

  2025 年关键更新：
  - OAuth 2.0 正式作为授权标准（Server 作为 Resource Server）
  - 结构化 JSON 输出
  - Tasks 抽象：异步长任务追踪
  - 加强了 Confused Deputy 和 token 盗用的安全考量

  3. 典型应用场景

  - 连外部 SaaS（GitHub、Jira、Notion、Slack）
  - 连企业内部数据库、知识库、监控系统
  - 连本地文件系统、浏览器、IDE
  - 跨 AI 产品复用同一套工具集成

  ---
  二、Skills（Agent Skills）

  1. 是什么

  Anthropic 于 2025 年 10 月 16 日 推出的功能，本质是 "一个带元数据的文件夹"，里面装着完成某类任务所需的
  指令、脚本、参考资料、模板。Claude 在运行时按需加载。

  一句话定位：MCP 是"给工具"，Skills 是"教 Claude 怎么用工具、怎么做事"。

  2. 构成

  目录结构：
  my-skill/
  ├── SKILL.md          ← 必需，入口文件
  ├── reference.md      ← 可选，详细参考
  ├── examples/         ← 可选，示例
  ├── scripts/          ← 可选，可执行脚本（Python、Bash 等）
  └── templates/        ← 可选，模板文件

  SKILL.md 两部分：
  ---
  name: pdf-form-filler
  description: Fill out PDF forms using field data from a JSON input.
                    Use when the user provides a PDF with form fields.
  ---

  # 主体：Markdown 指令
  ## 步骤
  1. ...
  2. 调用 scripts/fill.py ...

  - YAML frontmatter：仅 name + description，启动时全部加载进系统提示（非常轻量）
  - Markdown 主体：只有 Claude 判断"这个 skill 用得上"时才读取

  3. 核心设计：渐进式披露（Progressive Disclosure）

  三层加载（面试高频考点）：

  ┌──────────────────┬───────────────────────┬──────────────────────────────────┬─────────────────────────────────┐
  │       层级       │       加载时机        │               内容               │           Token 成本            │
  ├──────────────────┼───────────────────────┼──────────────────────────────────┼─────────────────────────────────┤
  │ L1 Metadata      │ 会话启动              │ 所有 skill 的 name + description │ 极低（每个 skill 仅几十 token） │
  ├──────────────────┼───────────────────────┼──────────────────────────────────┼─────────────────────────────────┤
  │ L2 SKILL.md 主体 │ Claude 判定相关后读取 │ 完整工作流指令                   │ 中等（建议 < 500 行）           │
  ├──────────────────┼───────────────────────┼──────────────────────────────────┼─────────────────────────────────┤
  │ L3 附加文件/脚本 │ 按需打开/执行         │ reference.md、scripts/*.py 等    │ 按需付费                        │
  └──────────────────┴───────────────────────┴──────────────────────────────────┴─────────────────────────────────┘

  关键前提：Claude 必须有 文件系统访问 + 代码执行 能力（因此在 Claude Code、Claude Desktop with file access
  里才完整工作）。

  4. 典型应用场景

  - 企业 SOP、内部写作/代码风格
  - 文档处理（PDF 填表、PPTX 生成、XLSX 清洗）
  - 固化工作流（代码评审流程、安全审计流程）
  - 打包团队专家知识供所有人复用

  ---
  三、MCP vs Skills：核心区别

  ┌────────────┬────────────────────────────────────────────────────────────────┬───────────────────────────────────┐
  │    维度    │                              MCP                               │              Skills               │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 本质       │ 通信协议（connection）                                         │ 打包的指令+资源（instruction）    │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 解决的问题 │ Claude 如何访问外部系统                                        │ Claude 如何正确做事               │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 类比       │ 给你一把锤子                                                   │ 教你怎么用这把锤子钉钉子          │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 运行载体   │ 独立进程/远程服务                                              │ 文件夹（本地或打包分发）          │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ Token 开销 │ 高：所有 tool schema 启动时全量注入（典型如 GitHub MCP 43      │ 低：仅 metadata                   │
  │            │ 个工具全部加载）                                               │ 驻留，主体按需加载                │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 更新/分发  │ 有协议版本化、跨厂商标准                                       │ 本地文件，暂无官方中心化注册表    │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 厂商中立性 │ 开放标准，多家支持                                             │ 目前 Anthropic 私有（Claude       │
  │            │                                                                │ 专属）                            │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 状态       │ 有状态会话                                                     │ 无状态文档/脚本                   │
  ├────────────┼────────────────────────────────────────────────────────────────┼───────────────────────────────────┤
  │ 安全边界   │ OAuth、独立进程沙箱                                            │ 依赖宿主的代码执行权限            │
  └────────────┴────────────────────────────────────────────────────────────────┴───────────────────────────────────┘

  最佳实践：二者结合使用。
  - MCP 提供"能做什么"（连上了 Jira）
  - Skills 提供"该怎么做"（我们公司提 bug 的模板、字段填写规则、流转流程）

  ---
  四、面试时建议注意的点

  1. MCP 的 token 膨胀问题 是当前最常被吐槽的痛点——每开一个 MCP Server，即使只用一个工具也会把全部 schema 灌进
  context。这也是 Skills 被称为"可能比 MCP 更有意义的抽象"的原因（Simon Willison、Armin Ronacher 等都撰文讨论过）。
  2. Skills 并非 MCP 的替代品，而是互补层。有观点认为 Anthropic 正在用 Skills 在"提示层"做它当年用 MCP
  在"协议层"做的事——标准化 + 可组合。
  3. 渐进式披露是现代 Agent 的通用设计范式，不只 Skills，LangGraph、AutoGen
  里的"条件化工具暴露"本质也是这个思路，可类比操作系统的"按需换页"。
  4. MCP 的三大原语（Tools/Resources/Prompts） 经常被拿出来单独问，尤其是 Resources 和 Tools 的区别：Resources
  只读、由模型/用户拉取，Tools 有副作用、由模型决策调用。
  5. 安全性：MCP 的 Confused Deputy 攻击、Prompt Injection 通过 Resource 进入、Tool 毒化（tool poisoning） 是 2025
  年学术界和工业界热议的话题，面试可能深挖。
  6. Skills 的"自动触发"机制 依赖 description 的措辞质量——写 skill 最大的技巧就是让 description
  既精准描述触发条件又简短，这是 prompt engineering for meta-level prompting。

  ---
  Sources

  - https://www.anthropic.com/news/model-context-protocol
  - https://modelcontextprotocol.io/specification/2025-11-25
  - https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/
  - https://forgecode.dev/blog/mcp-spec-updates/
  - https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
  - https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
  - https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices
  - https://claude.com/blog/skills-explained
  - https://simonwillison.net/2025/Oct/16/claude-skills/
  - https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/
  - https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/
  - https://www.mcpjam.com/blog/claude-agent-skills
  - https://intuitionlabs.ai/articles/claude-skills-vs-mcp
