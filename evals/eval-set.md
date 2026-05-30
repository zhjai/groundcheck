# GroundCheck trigger eval set

20 prompts to test whether the skill activates correctly. Target: both groups ≥ 90%.
Re-run after any `description` change (regression). ★ = boundary case vs agent-arena —
highest value, must not misfire.

## should-trigger (10)

| # | prompt | trigger dimension |
|---|--------|-------------------|
| 1 | 帮我核查这份调研报告里的数字和引用是不是真的 | numerical + citation |
| 2 | 这是 GPT 生成的答案,发布前检查有没有编造的事实 | generated content + pre-publish |
| 3 | Verify these API methods I cited actually exist in the library | code/API behavior |
| 4 | 我的 RAG 输出了这段,它真的被检索到的文档支持吗 | RAG groundedness |
| 5 | Fact-check this blog draft before I post it | explicit fact-check |
| 6 | 这段里引用的法律条款对吗 | legal citation (high-stakes) |
| 7 | Are the benchmark numbers in this README accurate? | benchmark claims |
| 8 | 检查这份文档里关于库版本和默认配置的说法 | config/version facts |
| 9 | Did I hallucinate any citations in my paper? | citation verification |
| 10 | 这段摘要忠实于原文吗,有没有加进原文没有的内容 | summary faithfulness |

## should-NOT-trigger (10)

| # | prompt | why not |
|---|--------|---------|
| ★1 | PostgreSQL 还是 DynamoDB 更适合我的认证服务? | decision/overconfidence → **agent-arena** |
| ★2 | 让 Codex 和 Claude 辩论这个架构方案 | explicit multi-agent debate → **agent-arena** |
| ★3 | 这两个方案哪个风险更低? | multi-perspective weighing → **agent-arena / deliberative** |
| 4 | 帮我把这段代码重构得更简洁 | refactor, no factual claims |
| 5 | 你觉得这个产品名好听吗? | pure opinion |
| 6 | 写一首关于秋天的诗 | creative |
| ★7 | 这个函数的逻辑对吗? | **code logic → tests** (but "does this API exist" WOULD trigger) |
| 8 | 把这段英文翻译成中文 | translation, nothing to verify |
| 9 | 解释一下什么是 RAG | explanation, no specific claim to ground |
| 10 | 帮我修复这个报错 | debugging, no fact-check |

## Notes

- ★1/★2/★3 guard the boundary with agent-arena: GroundCheck must **not** grab decision/debate
  requests. #7 guards the tests-vs-factcheck boundary: logic correctness → tests; API existence/behavior → groundcheck.
- Each "should-trigger" prompt should also be run through GroundCheck end-to-end at least once
  to confirm it produces a valid Claim Ledger (extraction → evidence → verdict → action).
