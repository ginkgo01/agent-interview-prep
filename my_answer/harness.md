# agent harness是什么？
harness 是包裹在LLM外面的运行时基础设施。
模型本身只会think和续写；harness则赋予其做事的能力：调用工具、管理上下文、管理长任务的状态、以及验证

## 核心模块
- 【工具集成】： harness负责拦截模型输出的工具调用指令。然后执行。
- 【上下文和记忆管理】: LLM context window有限，且没有跨session的记忆。Harness把记忆分成三层：
    - 工作记忆：当前prompt
    - 会话状态：当前任务进度
    - 长期记忆：跨session持久化的知识

- 【用户意图识别和任务编排】：harness负责识别用户意图，然后拆解为子任务，决定执行顺序。其实就是Orchestrator。

- 【观测、验证、安全护栏】： Harness 安排每一步输出后的格式校验、逻辑检查、安全过滤甚至自动化测试。让agent从"能跑"变成"能可靠地跑"。

- 【完成和交接】：在跨session的长任务中，在session之间进行任务交接。

## 为什么要harness：
2025年agent能工作。2026年要agent【可靠地】工作。

而且harness天生是模型无关的。

# harness、framework 区别：
 
- Framework
比如说LangChain。（注：langchain 1.0 之后就默认底层是langgraph了）
是SDK，提供构建agent的基础抽象功能：比如agent loop、 tool call的封装、 中间件机制等。
如何组装就得自己写。

- Runtime
比如说LangGraph。提供图结构流程控制、状态持久化、human-in-the-loop、流式输出等功能
感觉和framework是一个层级的，都不是很清楚。差不多知道有就行了。

- Harness
在Framework之上的一整套开箱即用的运行环境。
例如 claude code cli等。
Deep Agents SDK也是。（langchain公司整的harness。） 


## agent 和 workflow的区别
workflow 是预设的指定流程，每一步都稳定可控。
agent 是根据输入灵活的自适应处理问题。但存在不可控的风险。


# 企业级agent的架构
一个agent系统必须完成以下几件事：
1. 用户交互接入

2. 任务拆解和任务编排，多agent协作系统

3. 记忆和上下文

4. 工具与执行

5. 观测与反馈
    指完整的推理路径、token消耗、行为漂移检测和幻觉告警等。

6. 安全和兜底。
    输入验证、输出校验、数据脱敏合规等，必须by design嵌入agent系统中。








