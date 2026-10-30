# 模型特点与对策

每条对策都是从模型的工作方式推出来的。遇到新情况时回到左列想一想。

## 1 上下文有限，越满越差

会话里的每条消息、每个读过的文件、每段命令输出都占上下文。填得越满，越容易忘记早期指令、越容易出错。

对策：
- 规划文件分层。CLAUDE.md 每次必读所以要短；SPEC、ARCHITECTURE 按需读；任务描述里指明这次要读哪几节。
- 任务切到一个会话能做完的大小。
- 规划会话和执行会话分开。规划时读了大量材料，上下文已经很脏，写完文件后换新会话执行。
- 大范围调查交给子代理，主会话只拿结论。

## 2 用训练数据的平均值填空

人类工程师遇到没写清的地方会凭对业务的理解补上。模型补上的是最常见的做法，它能运行，但不一定是你要的。

对策：
- 重点写与默认不同的决定。模型本来就会做对的事不用写。
- 写清目的。知道为什么做，模型才不会优化错方向。
- 写「不做」清单，否则模型会顺手加功能。
- 给真实例子，一个输入输出样例胜过一段描述。

## 3 看起来完成就停

没有可运行的检查时，「看起来对」是模型唯一的停止信号，人就成了唯一的验证环节。

对策：
- 每条验收配一个能返回通过或失败的检查：测试、构建、类型检查、截图对照。
- 先写测试并看到它失败，再实现。这能防止写出永远通过的空测试。
- 规格以一个端到端验证步骤结尾。

## 4 倾向让人满意

模型会在功能没通的时候说通了，也会在缺数据时编数据。

对策：
- 完成时要求贴出命令和真实输出。
- 复审用独立会话或子代理。它只看改动和标准，看不到实现时的推理，所以不会被带偏。
- 给复审划范围：只报影响正确性和既定需求的问题。不划范围的话它总能找出点什么，追着改会过度设计。

## 5 跨会话没有记忆

对策：
- 每个决定写进决策记录，带日期和原因。
- 规格是活文档。发现规格错了，先改规格再改代码。
- 纳入版本控制，每个任务通过后提交一次，形成回滚点。

## 6 主流技术写得最好

对策：
- 选成熟、文档多的技术栈。
- 冷门库或新版本接口单独标风险，任务里附上官方文档链接让模型先读。
- 平台限制类的事实标「需核实」，由人或由联网查证确认，不靠模型记忆。

## 7 指令多了会互相稀释

同时给很多要求时，每条被遵守的概率都下降。规则文件越长，关键规则越容易被淹没。

对策：
- 规则文件逐行自问：删掉它模型会不会犯错。
- 必须零例外执行的事用钩子或脚本做，不靠文字叮嘱。
- 强调只给一两条，全都强调等于没强调。

## 8 文件冲突时随机选

对策：
- 定权威顺序，写在规格开头。
- 同一事实只写一处，别处用「见某文件某节」引用。
- 旧文件被取代时明确标注，不要让两个版本并存又不说明。

## 出处

- Anthropic，Best practices for Claude Code：https://code.claude.com/docs/en/best-practices
- Addy Osmani，How to write a good spec for AI agents：https://dev.to/addyosmani/how-to-write-a-good-spec-for-ai-agents-31i9
- GitHub，Spec-driven development with AI：https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/
- Simon Willison，Agentic Engineering Patterns：https://simonwillison.net/2026/Feb/23/agentic-engineering-patterns/
- SaaStr，做了五个生产应用后的复盘：https://www.saastr.com/the-live-complete-guide-to-vibe-coding-without-a-developer-what-we-actually-learned-after-building-5-production-apps
