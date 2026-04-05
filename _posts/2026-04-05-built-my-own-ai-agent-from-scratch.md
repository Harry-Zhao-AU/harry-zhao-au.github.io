---
layout: post
title: "I Built My Own AI Agent from Scratch to Understand How They Actually Work"
date: 2026-04-05
categories: [engineering]
tags: [ai, typescript, agents, nanobotts]
excerpt: "I didn't actually understand how AI agents work under the hood. So I decided to build one myself, from zero."
---

Last year I watched OpenClaw — the open-source reimplementation of Claude Code — and had a realisation: I didn't actually understand how AI agents work under the hood. I used them daily, but the gap between "user" and "builder" felt uncomfortably wide.

So I decided to build one myself, from zero.

The result is **NanoBotTS** — an ultra-lightweight AI agent framework in TypeScript, built incrementally over 13 phases. Not because the world needed another agent framework, but because *I* needed to understand every layer of the stack by writing it with my own hands.

## Why Not Just Read the Code?

Reading code teaches you *what* something does. Building it teaches you *why* it was designed that way.

I could have spent weeks reading through LangChain or AutoGPT source code. But I knew from experience that building something small and growing it layer by layer would teach me more than reading something large and complete. Every design decision you make yourself — and every mistake you recover from — sticks in a way that reading documentation never does.

## The Inspiration: nanobot

I found [HKUDS/nanobot](https://github.com/HKUDS/nanobot) — a Python AI agent in roughly 4,000 lines with a beautifully clean 5-layer architecture:

```
Channels -> MessageBus -> Agent Engine -> LLM Providers -> Tools
```

That simplicity was exactly what I needed. Not a framework with 200 abstractions, but a real agent with real capabilities that I could understand end-to-end. I decided to port its philosophy to TypeScript, building each layer from scratch.

## The Two Plans

I structured the entire project as two incremental plans, each written before I wrote a single line of code for that round.

### Plan 1: Build the Core (6 Phases)

The first plan was about getting a working agent from nothing:

**Phase 1 — Hello World Chat.** Just get TypeScript talking to Azure OpenAI GPT-4o. One file, one API call, one response. I learned `async/await`, the OpenAI SDK, and how a chat completion actually works at the API level.

**Phase 2 — Conversation Memory + Streaming.** Add a `Session` class to track message history so the bot remembers what we talked about. Add streaming so responses appear token-by-token instead of in one block. I learned about async iterators and state management.

**Phase 3 — Tool Calling (The Agent Loop).** This was the big one. This is where a chatbot becomes an *agent*. The core loop:

```
LLM thinks -> decides to call a tool -> we execute it -> feed results back -> LLM thinks again
```

I built a `ToolRegistry`, defined a `Tool` interface, created two simple tools (current time and web search), and wrote the `AgentRunner` that orchestrates the loop. Understanding this loop is understanding 80% of what makes agents work.

**Phase 4 — System Prompt + Configuration.** Configurable persona, model parameters, slash commands like `/help` and `/clear`. I learned how prompt engineering works from the builder's side — assembling a system prompt from multiple pieces of context.

**Phase 5 — Telegram Channel.** Adding a second interface forced me to think about the adapter pattern. The CLI and Telegram bot share the exact same agent core, just with different I/O adapters. This was my first taste of why decoupling matters.

**Phase 6 — Persistence.** Two-tier memory: the LLM consolidates important facts into `memory.md`, and raw sessions are saved as JSONL files. The agent can now remember things across restarts.

After Phase 6, I had a working personal AI agent. It could chat, use tools, remember context, and work over both CLI and Telegram. About 2,000 lines of TypeScript.

But I also understood why the original nanobot was architected the way it was — because I'd run into the same problems and arrived at similar solutions.

### Plan 2: Harden and Extend (7 More Phases)

The second plan came from doing a gap analysis: I compared what I'd built against nanobot's full architecture and identified every missing piece. I wrote `plan2.md` as a detailed roadmap with 11 phases and a dependency graph.

**Phase 7 — Foundation Hardening.** Schema validation for tool parameters. Token-aware context management — trimming old messages and capping tool outputs to stay within the context window. `SessionManager` with consolidation tracking. This phase taught me that "it works" and "it works reliably" are very different things.

**Phase 8 — MessageBus + AgentLoop.** The architectural unlock. Channels no longer call the agent directly. Instead, everything flows through an async message bus with inbound/outbound queues. The `AgentLoop` consumes from the bus, manages per-session locks and a global concurrency semaphore, and publishes results back. Channels become thin I/O adapters. This one change made everything else possible.

**Phase 9 — Provider Registry.** Abstracting away the LLM provider so I could swap between Azure OpenAI, standard OpenAI, Groq, Ollama, or any OpenAI-compatible API without changing agent code. Config-driven selection with automatic retry and backoff.

**Phase 10 — Expanded Tool Suite.** Filesystem tools (read, write, edit, list), shell execution with a deny-list, web fetch. Also concurrent tool execution — when the LLM calls multiple tools at once, execute the safe ones in parallel. This is where the agent started feeling genuinely *useful*.

**Phase 11 — Hooks + Templates.** An `AgentHook` interface with callbacks at every stage of the reasoning loop (before iteration, on stream, after tool execution, etc.). Template-based system prompts built from modular markdown files. This is where I understood why Claude Code has things like `CLAUDE.md` — it's the same idea of composing context from multiple sources.

**Phase 12 — Skills System.** Skills defined as markdown files with YAML frontmatter. Two loading strategies: always-on skills injected into every prompt, and on-demand skills the agent discovers and loads when needed. Workspace-level overrides so different projects can teach the agent different things.

**Phase 13 — Background Services.** `CronService` for scheduled tasks. `HeartbeatService` for autonomous monitoring (two-phase: cheap check first, full agent run only if needed). `SubagentManager` for spawning background worker agents with restricted tool access. This was the moment the agent stopped being purely reactive and started doing things on its own.

## What I Actually Learned

### 1. The Agent Loop Is Simpler Than You Think

The core of every AI agent is a while loop:

```typescript
while (true) {
  const response = await llm.chat(messages);
  if (response.toolCalls.length === 0) break;
  const results = await executeTools(response.toolCalls);
  messages.push(...results);
}
```

Everything else — memory, streaming, concurrency, multi-channel — is infrastructure *around* this loop. Once you understand the loop, you understand agents.

### 2. The MessageBus Pattern Is Why Real Agents Scale

When I had CLI calling the agent directly, adding Telegram meant duplicating logic. The moment I introduced a message bus, adding new channels became trivial — they just publish and subscribe. This is probably the most important architectural pattern in the whole project.

### 3. Context Management Is the Hard Part

The LLM has a fixed context window. Your conversation grows indefinitely. Managing this tension — trimming history, summarising old messages, budgeting tool output size, deciding what to keep and what to drop — is where most of the real complexity lives. The "AI" part is an API call. The engineering is everything around it.

### 4. Tools Are Just JSON Schema + a Function

Every tool is:
- A name
- A description (for the LLM)
- A JSON Schema (for parameter validation)
- An `execute()` function

That's it. Once you have a clean tool interface and registry, adding new capabilities is trivial. The LLM reads the schema and figures out when and how to use the tool.

### 5. Building Teaches You to Read

After building NanoBotTS, I went back and read through parts of Claude Code's architecture and OpenClaw's source. Everything made sense in a way it hadn't before. I could see *why* decisions were made because I'd faced the same trade-offs.

## The Tech Stack

Deliberately minimal:
- **TypeScript** + **Node.js** — strong types, huge ecosystem
- **OpenAI SDK** — works with any OpenAI-compatible API
- **Telegraf** — Telegram bot framework
- **Zod** — runtime schema validation

No LangChain. No vector databases. No orchestration frameworks. Just the fundamentals.

## The Architecture Today

```
User Input
    |
    v
Channel (CLI / Telegram)
    |
    v
MessageBus
    |
    v
AgentLoop (per-session lock)
    |
    v
ContextBuilder <- Templates + Memory + Skills
    |
    v
AgentRunner (LLM + tool loop)
    |
    +---> Provider (Azure / OpenAI / ...)
    +---> ToolRegistry (execute tools)
    +---> Response --> MessageBus --> Channel.send()
```

Background services (CronService, HeartbeatService, SubagentManager) inject messages into the bus independently, enabling autonomous behaviour.

## What's Next

The roadmap still has MCP integration (connecting to external tool servers) and a security layer on the horizon. But the core is solid, and more importantly — I understand every line of it.

## Advice If You Want to Do This

**Start with plan.md.** Before writing any code, write down your phases. What will you build in each one? What will you learn? Having a clear incremental plan kept me from getting lost in scope creep.

**Build the simplest version first.** Phase 1 was literally "send a message to GPT-4o and print the response." That's it. Get something working, then layer on complexity.

**Don't abstract too early.** In Phase 1, I had the LLM provider hardcoded. In Phase 9, I extracted a provider registry. The abstraction was better because I understood the concrete cases first.

**Stop after each phase.** Read the code. Understand what you built. Make sure you can explain every line before moving on. The goal isn't to finish — it's to learn.

**Do the gap analysis.** After Round 1, I compared what I had against the original nanobot's architecture. That gap analysis became `plan2.md` and gave me clear direction for Round 2. Knowing what you're missing is as valuable as knowing what you have.

## The Source

The full project is on GitHub: [NanoBotTS](https://github.com/Harry-Zhao-AU/NanoBotTS)

Both `plan.md` and `plan2.md` are in the repo. Read them as a learning roadmap, not just a task list.

---

*Building something from scratch is the fastest way to understand it deeply. You don't need to build the next ChatGPT. You just need to build something small enough to hold in your head, and grow it until you can't — then refactor until you can again.*
