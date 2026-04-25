# MCP
## 是什么
Model Context Protocol是一种 LLM 和外部系统交互的通信协议。
目标是打通 ai应用 和数据源/工具 之间的通信问题。

告诉 host 【能怎么做】

## 架构组成
Client-Server架构

- Host: 
    【就是那个应用】运行LLM的应用本体。
    持有会话和用户授权。    e.g. claude code、cursor
- Client：
    【Host内部的一个进程/模块】
    Host内部的协议客户端，与单个server 维持1：1连接
    SDK封装层
- Server：
    【外部一个按协议收发JSON-PRC消息的进程】
    xx MCP server

交互：按协议收发 JSON-RPC 消息

- 注 MCP官方的sdk其实管的主要是server这一端。host和client各家有各家的实现

### JSON-RPC 2.0
就是一个远程过程调用协议，不用管太细。
用json说 要调哪个函数，传什么参数
规定消息必须有 method、params、id、result 这几个字段

对方用json回 结果。

### Server
一个独立的程序/进程，可以用任何语言写，只要能按协议收发 JSON-RPC 消息。
本地命令、本地可执行文件、远程http连 都ok。

向Client【暴露】以下三项【原语】：
    指 Server 能提供的能力就这三种基本形状。

- Tool
就是LLM能够调用的函数。
例如：
```
  @server.tool()
  async def send_email(to: str, body: str) -> str:
      //执行
```
- Resources
只读数据源
```
  @server.resource("file://projects/{name}/README.md")
  async def get_readme(name: str) -> str:
      return open(f"projects/{name}/README.md").read()
```
- Prompt
存储一堆提示词模板，调用后返回指定提示词模板给Host。
e.g.
```
  你（用户）在界面上会看到一个可选列表，类似：
  可用模板：
    📋 summarize-pr  —— 总结一个 PR 的改动
    📋 review-code   —— 审查一段代码

  你点击 summarize-pr，它让你填个 PR 号，然后 自动生成一段精心设计的提示词 塞给 LLM：

  请分析 PR #142 的所有改动，从以下维度总结：
  1. 核心变更
  2. 潜在风险
  3. 测试覆盖情况
  ...
```


### Client
Client 侧也能提供能力（反向）：
- Sampling — Server 反过来让 Host 的 LLM 推理
- Roots — 告诉 Server 有权访问哪些文件根
- Elicitation（2025-06 新增）— Server 会话中途向用户请求输入


### 【暴露】的具体实现
连接 → 握手（能力协商）→ 拉清单（tools/list）→ 塞进 LLM 上下文 → LLM 看到后决定调用 → tools/call。

```
  第一阶段：握手（Initialize）

  Client 连上 Server 后，双方先交换"我能干什么"：

  // Client → Server："我支持什么"
  {
    "method": "initialize",
    "params": {
      "capabilities": {
        "sampling": {}    // 我支持你反过来让我推理
      }
    }
  }

  // Server → Client："我有什么"
  {
    "result": {
      "capabilities": {
        "tools": {},      // 我有 Tools
        "resources": {},  // 我有 Resources
        "prompts": {}     // 我有 Prompts
      }
    }
  }

  这一步只是说"我有这几类能力"，还没有给出具体清单。

  第二阶段：发现（List）

  Host 根据握手结果，按需拉取具体清单：

  // "你有哪些 Tools？"
  { "method": "tools/list" }

  // Server 返回完整清单（JSON Schema 描述每个 Tool）
  {
    "result": {
      "tools": [
        {
          "name": "send_email",
          "description": "发送邮件",
          "inputSchema": {
            "type": "object",
            "properties": {
              "to": { "type": "string" },
              "body": { "type": "string" }
            },
            "required": ["to", "body"]
          }
        },
        {
          "name": "create_issue",
          "description": "创建 GitHub Issue",
          "inputSchema": { ... }
        }
      ]
    }
  }

  Resources 和 Prompts 同理：resources/list、prompts/list。

    然后呢：注入 LLM

  Host 拿到清单后，把它塞进 LLM 的上下文（system prompt 或 tool
  definitions）。

  这就是之前说的 token 膨胀问题的根源——tools/list 返回多少，就往 LLM
  上下文里塞多少。GitHub MCP Server 43 个 Tool 的 schema
  全部注入，每轮对话都带着。
```
  Client          Server
    │── initialize ──→│
    │←── capabilities ─│   "我有 tools, resources, prompts"
    │── tools/list ──→│
    │←── 43个tool ────│   完整 schema
    │                  │
    │  (Host 把清单塞进 LLM 上下文)
    │                  │
    │── tools/call ──→│   LLM 决定调 send_email
    │←── result ──────│   执行结果

# Skills

## 是什么
本质是一个文件夹。里面装着完成某类任务所需的指令、脚本、参考资料、模板。运行时按需加载。

【教Host怎么用工具/做事】

## 架构组成
目录结构：
my-skill/
├── SKILL.md          ← 核心
├── scripts/          ← 可选，可执行脚本（Python、Bash 等）
└── templates/        ← 可选，模板文件

SKILL.md 两部分：

YAML frontmatter
```
name: pdf-form-filler
description: Fill out PDF forms using field data from a JSON input.
                  Use when the user provides a PDF with form fields.
```

markdown body
```
主体
```

# LLM从用户输入到使用MCP返回结果的全过程
1. 用户发送消息
2. Host组装请求。把用户消息 + tools清单一起发送给LLM

3. LLM决定调用工具

4. Host转发给MCP Server执行

5. Host拿到结果，将结果返回给LLM

6. LLM生成最终回答。



# MCP Server 如何开发？

三个步骤：
创建实例
用装饰器注册原语
选传输方式启动

### 第一步：初始化一个 Server 实例。
告诉 FastMCP "我要创建一个叫 xxx 的 Server"。
- mcp: 一个SDK，是MCP官方提供的开发库
- 可理解为一种封装，把Server的创建和注册原语的功能封装起来，方便开发者使用
- fastmcp： 这个SDK中的一个类，封装了【自动生成 JSON Schema 和 自动处理请求路由 的功能。】
  - 自动生成 JSON Schema。
  底层 Server 你得自己手写每个 tool 的参数描述（类型、必填、默认值）。FastMCP 直接从你的 Python 函数签名和类型标注自动推导，你写 def add(a: int, b: str) 它就知道了。
  - 自动处理请求路由。
  底层 Server 你得自己监听 tools/list、tools/call 等请求，手动判断调哪个函数。FastMCP 用 @mcp.tool() 装饰器自动注册，请求进来时自动匹配到对应函数。

```
  from mcp.server.fastmcp import FastMCP
  server = FastMCP("my-server")
```
- 注：
  - 底层 API：mcp.server.Server —— 你得自己手动处理请求路由、手写 JSON Schema
  - 高层 API：mcp.server.fastmcp.FastMCP —— 用装饰器 + 类型标注自动搞定一切

### 第二步：往空的Server上注册原语。
写需要用的tools、resources和prompts。
然后使用python装饰器，把你写的函数"挂"到 Server 上。（即@mcp.tool()等）

```
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("my-server")       # 第一步：空壳

# ---- 第二步：注册原语 ----

@mcp.tool()                       # 这个装饰器 = "把下面的函数注册为 Tool"
async def search_user(name: str) -> str:
    """根据姓名搜索用户"""
    result = db.query(name)       # 你的业务逻辑
    return result

@mcp.tool() 做了什么：
1. 把 search_user 这个函数记录到 Server 的内部注册表里
2. 读取函数名 → tool 名叫 search_user
3. 读取参数 name: str → 生成 inputSchema
4. 读取 docstring → 生成 description

之后 Host 发 tools/list 时，Server 就从这个注册表里把所有注册过的函数信息返回出去。
```

### 第三步：选择传输方式，并启动。
选择本地通信方式还是http方式。
本地：mcp.run(transport="stdio")
之后server会作为host的子进程开始运行。

远程：mcp.run(transport="http")


## 开发完之后，需要调试和接入

### 调试
  npx @modelcontextprotocol/inspector python server.py

  它会在浏览器打开一个 Web 面板，让你不接入任何 LLM 的情况下手动测试 Server：

  - 验证注册是否正确：在 Inspector 里点 tools/list，看返回的清单——tool 名字对不对、参数 schema 对不对、description 有没有缺。
  - 验证调用能不能跑通：手动填参数调用某个 Tool → 检查结果对不对。
  - 验证通信：看调用时的 JSON-RPC 消息收发记录。


### 接入
在 Host 的配置文件里声明 Server 怎么启动就行。
- 比如说 openclaw mcp add --transport stdio serverA python /绝对路径/server.py
- 其实就是 修改了 ~/.openclaw/openclaw.json：
- claude cli中就是改 ~/.claude/settings.json

# token膨胀问题
## 问题原因
Host 从 Server 拉到 tools 清单后，每轮对话都要把完整清单塞进 LLM API 的 tools 参数里。
```
一个 GitHub MCP Server 暴露 43 个 tool，每个 tool 的 schema（名字、描述、参数定义）可能占几百 token。43 个加起来几千 token。你聊 100 轮，这几千 token 就重复传了 100
次——即使你一次都没调用过其中任何一个。
```

这就是skills为什么好：每个 skill 启动时只占几十 token 的 metadata，主体按需加载。

## claude是怎么解决的【渐进式披露】
claude code cli 在他们的host 和 client上做了如下处理：
```
● Claude Code 的做法分三步：

  第一步：启动时只存名字，不存完整定义。
  所有 MCP tool 标记为"延迟加载"，上下文里只保留 tool 名字和简短描述，不放完整的参数 schema。同时自动注入一个内置工具叫 ToolSearch。

  第二步：需要时，LLM 先搜索再使用。
  当用户说"帮我发个邮件"，Claude 发现自己手头没有 send_email 的完整定义，就先调用 ToolSearch("发邮件")。系统用关键词+语义匹配，从所有延迟加载的 tool 里找到最相关的 3-5
  个，把它们的完整 schema 返回给 Claude。

  第三步：激活后记住，后续轮次复用。
  一旦某个 tool 被搜索发现过，它的完整定义就会留在对话上下文里，后续轮次不用再搜。

  ---
  触发条件：当 MCP tool 的总定义超过上下文窗口的 10% 时，Claude Code 自动启用这个机制。tool 少的时候还是直接全部加载。
```


