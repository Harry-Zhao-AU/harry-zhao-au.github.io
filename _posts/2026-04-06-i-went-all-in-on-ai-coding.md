---
layout: post
title: "I Went All-In on AI Coding — Here's What It Can't Do"
date: 2026-04-06
categories: [engineering]
tags: [ai, claude-code, agents, productivity, openclaw]
excerpt: "I learned every feature, read the source code, built my own agent from scratch. Here's what AI still can't do — and what actually makes it work."
---

I use AI to write code every day. Not as an experiment — as my primary workflow. I've studied how the tool works, read its source code, and built my own agent framework from scratch.

I'm not here to tell you AI is magic. I'm here to tell you what it actually can and can't do — and what makes the difference.

## 1. I Went All-In

When I started using [Claude Code](https://claude.ai/code), I didn't just install it and start prompting. I treated it like any serious tool — I studied it.

**The features.** Slash commands like `/review` for code review, `/init` to scaffold project context, `/simplify` to clean up code. Resume mode to pick up where I left off. Ultrathink for complex architectural reasoning. MCP servers to extend the tool with external integrations.

**The workflows.** I studied [Boris Tane's approach](https://boristane.com/blog/how-i-use-claude-code/) — his three-phase workflow of Research → Plan → Annotate → Implement. The core principle: never let the AI write code until you've reviewed and approved a written plan. That became my daily rhythm.

**The creator.** I studied how Boris Cherny — the person who built Claude Code — uses his own tool. There's something uniquely valuable about watching a toolmaker use their own creation. You see the intended workflow, not just the documented one.

From these influences, I built my own daily practice. Not copying anyone's workflow exactly, but understanding the principles well enough to adapt them to my context.

## 2. I Went Deeper

Using Claude Code daily made me curious. How does an AI agent actually work under the hood? What happens between my prompt and the response?

I found [OpenClaw](https://github.com/anthropics/claude-code) — and started reading the source. The agent loop, tool execution, context management, how slash commands are implemented. I wasn't trying to contribute — I was trying to understand.

One thing I noticed: both Claude Code and OpenClaw are built in **TypeScript**. Not Python, which dominates the AI/ML space. That choice told me something about the engineering trade-offs — TypeScript's type system, async-first design, and ecosystem make it well-suited for building complex interactive agents. That observation would matter later.

Reading the source bridged a gap. I went from "I know how to use this tool" to "I understand why it behaves the way it does." When something unexpected happened, I could reason about it instead of just retrying.

## 3. I Built My Own

Reading wasn't enough. I wanted to feel every design decision.

I found [nanobot](https://github.com/HKUDS/nanobot) — a Python AI agent in roughly 4,000 lines with a clean 5-layer architecture. The perfect learning target: small enough to understand end-to-end, complete enough to be real.

I decided to port its philosophy to TypeScript — not Python, because I'd seen why the agent ecosystem was moving toward TypeScript. The result was [NanoBotTS](https://github.com/Harry-Zhao-AU/NanoBotTS), built incrementally across 13 phases:

```
Phase 1:  Hello World — one API call, one response
Phase 2:  Conversation memory + streaming
Phase 3:  Tool calling — the agent loop (this is the big one)
Phase 4:  System prompts + configuration
Phase 5:  Telegram channel — the adapter pattern
Phase 6:  Persistence — two-tier memory
Phase 7:  Foundation hardening — schema validation, token management
Phase 8:  MessageBus + AgentLoop — the architectural unlock
Phase 9:  Provider registry — swap LLMs without changing agent code
Phase 10: Expanded tool suite — filesystem, shell, web fetch
Phase 11: Hooks + templates — extensibility points
Phase 12: Skills system — on-demand capabilities
Phase 13: Background services — cron, heartbeat, subagents
```

The meta part: I used Claude Code to help build this. The tool teaching me about itself.

After building NanoBotTS, I went back to OpenClaw's source. Everything made sense in a way it hadn't before. I could see *why* decisions were made because I'd faced the same trade-offs. (I wrote about this in detail in [my NanoBotTS post](/blog/2026/04/05/built-my-own-ai-agent-from-scratch/).)

## 4. Here's What It Can't Do

After all of that — learning, studying, building — here's the honest part.

**AI agents produce logic bugs that look correct at first glance.** The code compiles. The tests might even pass. But there's a subtle edge case, an off-by-one, a race condition that the AI didn't anticipate because it doesn't truly *reason* about your system — it predicts plausible code based on patterns.

**AI is confident, not correct.** It doesn't say "I'm not sure about this." It writes code with the same tone whether it's right or wrong. That confidence is dangerous if you're not paying attention.

**It doesn't know what it doesn't know.** Your business logic, your team's conventions, the incident from last quarter that changed how you handle retries — the AI has none of this context unless you explicitly provide it.

**"Just build it" prompts produce fragile code.** The 3-phase workflow (research → plan → implement) exists for a reason. When you skip the planning step and let the AI write code directly, you get something that works for the happy path and breaks everywhere else.

This isn't a criticism — it's a calibration. AI agents are incredibly powerful. But they're powerful in the way a jet engine is powerful: without a skilled pilot, you're not going anywhere useful.

## 5. What Actually Makes It Work

Knowing the limitations is only half the answer. Here's what I've learned about closing the gap.

### Doc as Code

The AI reads what's committed, not what's in your head. Keep your architecture decisions, conventions, and context **in the repo**:

- `CLAUDE.md` at the project root — project-specific instructions the AI loads automatically
- `README.md` that actually describes how things work, not just how to install
- Architecture Decision Records (ADRs) for non-obvious choices
- Inline comments where the *why* isn't obvious from the code

The better your documentation, the better your AI output. This isn't new wisdom — it's the same principle that makes onboarding new team members easier. The AI is just another team member who reads your docs on day one, every day.

### Context Engineering

The quality of AI output is a function of the quality of input. **Curate what the AI sees.**

- Structure your repo so related code is discoverable
- Write clear commit messages — the AI reads git history
- Use consistent naming conventions — the AI learns your patterns
- Keep your `CLAUDE.md` updated as the project evolves

This is what I call context engineering: deliberately shaping the information environment so the AI has the right context automatically, without you restating it every session.

### Validation as a First-Class Step

Don't just eyeball the diff. Build verification into the workflow:

- **Type checking** — catch structural errors before they become runtime bugs
- **Tests** — not just for coverage, but as executable specifications the AI must satisfy
- **Linting** — enforce conventions the AI might drift from
- **Run before accepting** — if it compiles and passes tests, you have a baseline. If it doesn't, you've caught the problem in seconds, not in production.

The Research → Plan → Implement workflow should really be Research → Plan → Implement → **Validate**. That last step is non-negotiable.

### Harness the Tool

Claude Code is extensible by design. Use that:

- **Hooks** — shell commands that run automatically before or after tool calls
- **Custom slash commands** — encode your team's workflows as reusable commands
- **MCP servers** — connect the AI to your actual systems (databases, APIs, internal tools)
- **CLAUDE.md** — project-level instructions that shape every interaction

Don't use the tool as-is. Configure it to work *your* way. The default experience is the starting point, not the ceiling.

### Self-Verification

Before accepting AI output, ask it to critique itself:

> "Review what you just wrote. What edge cases could break? What assumptions are you making?"

This is cheap and surprisingly effective. The AI often catches its own mistakes when explicitly asked to look for them. It's not a replacement for human review, but it's a useful first pass.

## The Takeaway

AI didn't make me a better engineer. Fourteen years of experience tell me what to ask for, when to push back, and when the output looks wrong even if I can't immediately articulate why.

What AI did is **compress the loop**. The research that took hours takes minutes. The boilerplate that took a day takes seconds. The exploration of unfamiliar codebases that took a week takes an afternoon.

But the judgment — what to build, how to structure it, when to reject the AI's suggestion — that's still human. And from what I've seen building my own agent, that's not changing anytime soon.

The engineers who benefit most from AI are the ones who already know what good looks like. AI amplifies skill. It doesn't replace it.

---

*If you're interested in the technical deep-dive, read [how I built NanoBotTS from scratch](/blog/2026/04/05/built-my-own-ai-agent-from-scratch/). If you want to see how I use Claude Code for everyday tasks, read [how I built this blog in 2 hours](/blog/2026/04/06/how-i-built-this-blog-with-claude-code/).*
