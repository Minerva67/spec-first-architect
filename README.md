# spec-first-architect

写代码之前的架构师流程：把 PRD、原型、设计稿、现有代码和口头想法，变成一套模型能执行也能自查的开工文件（SPEC.md 规格 + ARCHITECTURE.md 架构 + TASKS.md 任务 + DECISIONS.md 决策记录 + 精简的 CLAUDE.md）。凡是用户说「做个架构规划 / 出技术方案 / 写 spec / 写项目规格 / 先架构后写码 / 开工前准备什么 / 帮我规划怎么做这个产品 / 这个 PRD 怎么落地 / 拆一下任务 / vibe coding 怎么起步」，或者丢来 PRD、原型、设计稿压缩包要求「按这个做」「评估一下怎么实现」时，务必先用本 skill，再动手写任何业务代码，即使用户没说「架构」或「spec」。也适用于已有代码库上加大功能、换版本、重构前的规划。不用于一句话能说清的小改动（直接改），不用于写产品 PRD 本身（那是产品文档），不用于竞品拆解（competitor-teardown）。

一个 Claude Code skill。来自合集 [claude-skills](https://github.com/Minerva67/claude-skills)。

## 安装

```bash
git clone https://github.com/Minerva67/spec-first-architect.git ~/.claude/skills/spec-first-architect
```

装好后在 Claude Code 里输入 `/spec-first-architect`，或者直接描述任务，符合条件时会自动触发。

## 文件

```
SKILL.md
assets/ARCHITECTURE.template.md
assets/CLAUDE.template.md
assets/DECISIONS.template.md
assets/SPEC.template.md
assets/TASKS.template.md
references/interview.md
references/irreversible-checklist.md
references/model-traits.md
references/platform-wechat-miniprogram.md
```
