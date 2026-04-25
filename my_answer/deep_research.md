# Agent 是怎么检索的？
两阶段。
先websearch，给查询词，返回一组搜索结果（标题、链接、简短摘要）
再webfetch， 打开具体的URL，获取内容。

## 生成查询词
agent常见的问题是查询词写的太长太具体。
因此应该采取 先board后concrete：先看信息分布怎么样，再逐步收窄。

具体有几种技术

- 【关键词提取和扩展】：
  LLM 从用户的自然语言问题中抽出核心关键词，然后用同义词、缩写、相关术语做扩展。比如用户问"Claude 的记忆怎么实现的"，Agent 可能生成 Claude memory implementation、Claude context persistence、Claude session state 多组查询。

- 【问题拆解】：
  把一个复合问题拆成多个独立子问题。Chroma 的 Context-1 系统明确要求 Agent "break down the query into its key concepts and information needs, list each one explicitly"，然后对每个概念制定单独的搜索策略。

- 【伪答案生成（Pseudo-Answer / HyDE）】——不搜问题本身，而是让 LLM 先想象"如果有一篇文章回答了这个问题，它的标题和摘要大概长什么样"，然后用这个假想的文档描述去做语义检索。

- 【并行查询】。一次websearch多项。

## 筛选结果
- 【websearch返回了一堆链接，如何确定要不要读？】
  - 先看来源域名。筛选出像官方博客/权威网站的文章。
  - 再看标题和意图是否匹配。
  - 尽量挑不同来源的文章，
  - 数量控制

## 要不要继续搜索
采用Interleaved Thinking 交错思考：
整个检索过程是一个 observe → reason → act 的循环，让模型思考当前检索结果质量如何、缺什么信息、下一步该搜什么。


### 局限性
获得的所有信息都是“搜索引擎返回的消息”。
即：如果一个帖子质量很高，但搜索引擎没有检索回来的话，那..永远没有办法检索回来了。

可能的解法：
1. 同时搜索不同的来源，然后交叉比对。
2. 不同领域用不同的检索引擎/工具。
3. 换掉传统搜索引擎：用语义搜索替代关键词匹配
4. 主动爬取：Agent直接去爬取和解析网页，就像一个人类那样——这能找到搜索引擎无法返回的内容。【关于特定APP封闭的生态也能起到作用！特定APP的某些帖子、内容，有时在传统搜索引擎中排名偏后或无法检出】




## claude Research 的 harness

【Orchestrator-Worker】
一个 Lead Agent（Claude Opus） 当指挥官，多个 Subagent（Claude
  Sonnet） 当兵。

用户提交一个研究查询后，Lead Agent 用 extended thinking 分析意图、制定策略，然后 spawn 出 3-5 个子 Agent 并行出去搜索不同方向的信息。子Agent 完成后把压缩过的发现返回给 LeadAgent，由它判断够不够——不够就再派一轮，够了就综合成最终答案。最后还有一个专门的CitationAgent 负责给每个结论标注引用来源。

### 上下文管理
- 每个子agent 当然有自己独立的context window
他们在自己的窗口里工作，最后
把【研究结论】，即浓缩过的摘要传回给lead agent。
把大体积的原始材料写到外部存储系统里面，并传回相应的轻量引用。
- 外部记忆：
    - session log： 每个agent 有一个session log。所有事件都这样持久化存储起来。如果崩溃了能恢复。
- Compaction
    - lead agent 自身上下文快满了的时候会压缩


# Deep Research
## Deep Research 要配备哪些工具？
必备：
- 搜索工具
- 网页抓取工具
- 记忆管理工具（文件读写）

高价值：
- 结构化数据提取： 按schema从页面中抽取结构化的数据
- 多模态感知： 处理网页中的图表

- 子agent派生

- 代码执行沙箱
- 学术检索引擎

## Deep Research 基础设施怎么搭？
“基础设施”指的是让工具和agent本身能跑起来的底层支撑系统：
计算资源、存储、网络、部署、监控等。

- 算力
- 存储：agent的log，搜索/抓取结果，子agent阶段性成果(长期记忆)等。
- 搜索和抓取服务
- 代码执行环境
- 监控和追踪