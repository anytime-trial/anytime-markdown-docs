# Claude Product Lineup Comparison

Updated: 2026-03-21


## Overview

Anthropic's Claude products fall into three categories.

| Category | Product | Target |
| --- | --- | --- |
| Consumer | Claude (Web / Desktop / Mobile) | General users |
| Developer tools | Claude Code (CLI / Desktop) | Software developers |
| Platform | Claude API | Application developers |


## Feature Comparison by Product

| Feature | Claude Web/Desktop | Claude Code CLI | Claude Code Desktop | Claude API |
| --- | --- | --- | --- | --- |
| Interface | Browser / App | Terminal | Desktop app | HTTP API |
| Conversation / Chat | o | o | o | o |
| File read/write | x | o | o | o (app-dependent) |
| Command execution | x | o | o | x |
| Web search | o (Pro+) | o | o | x |
| MCP servers | o | o | o | x |
| Custom skills | x | o | o | x |
| Background tasks | o (Cowork) | o (subagents) | o | x |
| Scheduled execution | o (Cowork `/schedule`) | o (`/loop`) | o | x |
| Git operations | x | o | o | x |
| IDE integration | x | o (VS Code etc.) | x | x |
| Multi-agent | x | o (Agent SDK) | x | o (Agent SDK) |


## Cowork (Claude Desktop Agent Feature)

Background agent feature built into Claude Desktop.\
Brings the same agentic architecture as Claude Code without requiring a terminal.

| Item | Details |
| --- | --- |
| Availability | Pro and above paid plans |
| Platforms | Windows / macOS |
| File access | Only explicitly permitted folders |
| Scheduling | Recurring tasks via `/schedule` |
| Sub-agents | Splits complex tasks for parallel execution |
| Persistent instructions | Global instructions / per-folder instructions |

> Difference from Claude Code CLI: Cowork has shorter sessions with more guardrails.\
> Multi-agent orchestration is only available via CLI and Agent SDK.


## Subscription Plans

| Plan | Monthly | Claude Code | Cowork | Models | Usage |
| --- | --- | --- | --- | --- | --- |
| Free | $0 | x | x | Sonnet 4.5 | Limited |
| Pro | $20 | o | o | Sonnet 4.5 + Opus (limited) | Standard |
| Max 5x | $100 | o | o | All models | 5x Pro |
| Max 20x | $200 | o | o | All models | 20x Pro |
| Team | $25–$150/seat | o (Premium seat) | o | All models | By seat type |
| Enterprise | Custom | o | o | All models | Custom |

> Team Standard seats ($25–$30/month) do not include Claude Code.\
> Premium seats ($150/month) include Claude Code access.


## API Pricing (Pay-as-you-go)

A **completely separate billing system** from subscriptions.\
Requires an API key with usage-based billing.

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
| --- | --- | --- |
| Opus 4.5 | $5 | $25 |
| Sonnet 4.5 | $3 | $15 |
| Haiku 4.5 | $1 | $5 |

### Cost Reduction Options

| Option | Discount | Condition |
| --- | --- | --- |
| Batch API | 50% off | Asynchronous processing (no immediate response) |
| Prompt Caching (5 min) | 90% off input | 1.25x write cost, pays off after 1 cache read |
| Prompt Caching (1 hour) | 90% off input | 2x write cost, pays off after 2 cache reads |


## Subscription vs API: When to Use Which

| Use case | Recommended | Reason |
| --- | --- | --- |
| Daily development | Pro / Max subscription | Fixed price includes Claude Code |
| CI/CD automation | API | API key required for GitHub Actions etc. |
| Embedding in your app | API | API is required to call Claude from applications |
| Large batch processing | API (Batch) | 50% discount for async processing |
| Team collaboration | Team / Enterprise | Admin features, SSO, audit logs |


## Product Relationships

```
Claude (Web/Desktop/Mobile)
├── Chat, file upload, web search
├── Cowork (background agents)
└── Projects (prompt management)

Claude Code
├── CLI (terminal-based)
│   ├── Skills & custom commands
│   ├── MCP servers
│   ├── Subagents & multi-agent
│   └── IDE integration (VS Code etc.)
└── Desktop (GUI-based)
    ├── Embedded browser
    └── Visual feedback

Claude API
├── Messages API (chat)
├── Batch API (async batch)
├── Agent SDK (agent building)
└── Tool Use (function calling)
```

> All products use the same Claude models (Opus / Sonnet / Haiku).\
> There is no difference in intelligence or answer quality — only tools, features, and interfaces differ.


## References

- [Plans & Pricing | Claude by Anthropic](https://claude.com/pricing)
- [Pricing - Claude API Docs](https://platform.claude.com/docs/en/about-claude/pricing)
- [Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan)
- [Claude Code Desktop](https://code.claude.com/docs/en/desktop)
- [Cowork Product Page](https://claude.com/product/cowork)
- [Claude, Claude API, and Claude Code: What's the Difference?](https://eval.16x.engineer/blog/claude-vs-claude-api-vs-claude-code)
- [Claude Code Pricing Guide](https://www.ksred.com/claude-code-pricing-guide-which-plan-actually-saves-you-money/)
