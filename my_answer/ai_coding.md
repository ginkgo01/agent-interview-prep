
# karpathy

## AI-assisted coding
1. 把所有东西丢进上下文。
2. 描述需要实现的next single, concrete incremental change。
don't ask for code, ask for a few high-level approaches. 和这些方法的利弊。
这里需要人类来选择其中的一种方法。

3. 选择一种方法让AI写好。

4. Review 阶段：调出所有不熟悉的函数和API的问问你当，询问explanations、clarifications and changes。
需要回滚时回滚。

5. 测试
6. git提交


## karpathy skills
name: karpathy-guidelines
description: Behavioral guidelines to reduce common LLM coding mistakes. Use when writing, reviewing, or refactoring code to avoid overcomplication, make surgical changes, surface assumptions, and define verifiable success criteria.


1. Think Before Coding
2. Simplicity First
3. Surgical Changes
4. Goal-Driven Execution


1. Think Before Coding
Don't assume. Don't hide confusion. Surface tradeoffs.
别妄下定论。不要掩饰困惑。表面权衡。

Before implementing:  在实施之前：

State your assumptions explicitly. If uncertain, ask.
明确表达你的假设。如果不确定，可以问。
If multiple interpretations exist, present them - don't pick silently.
如果存在多种解读，就提出来——不要默默选择。
If a simpler approach exists, say so. Push back when warranted.
如果有更简单的方法，请说明。必要时反驳。
If something is unclear, stop. Name what's confusing. Ask.
如果有什么不清楚的地方，就停止。说出什么让人困惑。问吧。

2. Simplicity First

Minimum code that solves the problem. Nothing speculative.
只需最小限度的代码来解决问题。没有任何猜测性内容。

No features beyond what was asked.
除了被要求的部分，没有其他特征。
No abstractions for single-use code.
一次性代码不做抽象。
No "flexibility" or "configurability" that wasn't requested.
没有未被要求的“灵活性”或“可配置性”
No error handling for impossible scenarios.
不可能的情景不做错误处理。
If you write 200 lines and it could be 50, rewrite it.

3. Surgical Changes
Touch only what you must touch. Clean up only your own mess.

When editing existing code:
编辑现有代码时：
Don't "improve" adjacent code, comments, or formatting.
不要“改进”相邻的代码、注释或格式。
Don't refactor things that aren't broken.
不要重构那些没有坏掉的东西。
Match existing style, even if you'd do it differently.
即使你会用不同的方式，也要匹配现有的风格。
If you notice unrelated dead code, mention it - don't delete it.
如果你发现了无关的死代码，要提一提——不要删除。

When your changes create orphans:
当你的更改产生孤儿时：
Remove imports/variables/functions that YOUR changes made unused.
移除那些是你自己改动导致没用到的导入/变量/函数。
Don't remove pre-existing dead code unless asked.
除非被要求，不要删除已有的死代码。
The test: Every changed line should trace directly to the user's request.
测试：每一行更改的线条都应直接追踪到用户的请求。


4. Goal-Driven Execution

Define success criteria. Loop until verified.
定义成功标准。循环直到确认。

Transform tasks into verifiable goals:
将任务转化为可验证的目标：

"Add validation" → "Write tests for invalid inputs, then make them pass"
“添加验证”→“为无效输入写测试，然后让它们通过”
"Fix the bug" → "Write a test that reproduces it, then make it pass"
“修复 bug”→“写一个复现它的测试，然后让它通过”。
"Refactor X" → "Ensure tests pass before and after"
“重构 X”→“确保测试在前后通过”
For multi-step tasks, state a brief plan:
对于多步骤任务，请提出简要计划：

1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.
强有力的成功标准让你可以独立循环。薄弱的标准（“让它奏效”）需要不断澄清。




# ai coding 的技巧和心得
需要将LLM视为强大的“实习生”，需要明确的方向、背景和监督

### start with a clear plan
spec.md
先和AIbrainstrom 一个详细的方案，制定逐步计划。

- 制定spec.md ：描述想法并让LLM从各种角度反复提问：
主要是针对【需求、架构、数据结构和边缘案例的测试】。

- 生成项目计划plan.md ：将这个spec.md输入LLM让他生成项目计划。“将实现拆分为logical small tasks or milestones”。
做计划的时候，设计好goal driven的目标。和ai沟通定义好每个step的成功标准，设定“可验证的目标”

一般会生成一个design doc 或者project plan。

### break work into small, iterative chunks
Scope management。避免一次给LLM太多内容

避免向AI要求一次性大量输出。
把项目拆分为计划（前文提到）

### provide correct background context manually

模型在运行时需要知道 要看的代码，项目技术限制， 偏好方法等。

- 使用Context7 这样的MCP。
Context7可以为host提供实时、最新的技术栈的API和框架官方文档。

就是在编码前让模型对该了解的内容都了解一遍。
有诸如gitingest或repo2txt这样的上下文打包工具，可以把代码库中的相关部分转储到文本文件中，供llm读取。

选择性的给AI “只包含当前任务相关的上下文”。如果某事超出范围，就不要关注（为了节省token）


### 经常commit，实现（超细粒度的）版本控制。


