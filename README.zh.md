# GroundCheck

<p align="center">
  <img src="assets/banner.svg" alt="GroundCheck — 单 agent、证据接地的声明核查,专治幻觉" width="100%">
</p>

<p align="center">
  <a href="README.md">English</a> · <strong>中文</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/skill-groundcheck-blue" alt="skill">
  <img src="https://img.shields.io/badge/version-0.1.1-informational" alt="version">
  <a href="https://github.com/zhjai/agent-arena"><img src="https://img.shields.io/badge/companion-agent--arena-6b46c1" alt="agent-arena"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="MIT License"></a>
</p>

> **单 agent、证据接地的声明核查** —— 把每条声明对照真实证据核实来抓出编造的事实,而不是靠"多叫几个模型来点头同意"。

GroundCheck 接收**已经存在的内容**——一个生成的答案、一份报告、RAG 输出、文档或代码——提取其中的可验证声明,把每条声明接地到真实证据,返回逐条的判定(verdict)+ 引用 + 一个动作:**保留 / 修订 / 撤回 / 打回**。

## 为什么是单 agent(以及为什么这很关键)

多 agent 辩论擅长治**过度自信**——但它会**强化共享的幻觉**,因为多个用相似数据训练的模型会一起确认同一个错误事实。GroundCheck 刻意**不用**多 agent panel:它把每条声明接地到确定性的外部证据(测试、源码、文档、web、检索上下文)。

- **GroundCheck** → 治**幻觉**(单 agent + 证据接地)
- [**agent-arena**](https://github.com/zhjai/agent-arena) → 治**过度自信**(多 agent 辩论)

两者是**同一验证栈的两个深度**,通过共享的 [Claim Ledger](interop/claim-ledger.md) 互操作。

## 它产出什么

每条声明:一个原子化、已分类、带日期的判定 ——
`supported · partially_supported · refuted · unverifiable · outdated · needs_qualification` ——
附引用证据 + 推荐动作。**复合声明会被原子分解**,这样一个为真的子声明无法"洗白"一个错误的捆绑结论。

ledger 是**可争议的,不是权威**:每个判定都可追溯、有时间界限,且能被更强的反证重新打开。**核查器自己也可能错。**

## 作为"事实门"接入任何多 agent 系统

GroundCheck 是一个**通用、可插拔的验证门**——agent-arena、CrewAI、AutoGen、LangGraph 或任何编排器都能通过 [事实门契约](interop/fact-gate.md) 接入:

```
各 agent 独立作答
   → 辩论前的门:对每个答案跑 groundcheck
       → refuted? 把该声明(附证据)在辩论前打回该 agent
   → 辩论
   → 辩论后的门:复查新引入/改动的声明
```

在**辩论之前**抓出事实错误,正是阻止 panel 强化共享幻觉的关键。打回时送的是**证据,不是结论**,且原始答案保持**不可变**——这样原 agent 是基于证据独立推理,而不是为讨好核查器而修改。

## 何时用 / 不用

**用:** 核查声明 · 检查幻觉 · "这些引用/数字/API 是真的吗" · 发布前事实核查 · RAG groundedness · 作为多 agent 流程里的事实门。

**不用于:** 纯观点/创意内容 · 已有测试覆盖的代码逻辑 · "哪个方案更好"这类决策(那是 agent-arena)。

## 安装

```bash
npx skills add zhjai/groundcheck -g -a claude-code
```

支持 Claude Code、Codex、Cursor、OpenCode 等 Agent Skills 宿主。也可以把 `skills/groundcheck/` 复制到你的 agent skills 目录。

## 状态

`v0.1.0` 预览版。与任何厂商无隶属关系。MIT 许可证。
