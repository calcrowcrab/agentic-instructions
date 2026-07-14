---
name: agentic-instructions
description: Write and revise the prompts, instructions, and tool definitions that tell an AI agent what to do. Use when creating or editing a system prompt for any agent: voice agents (receptionist, booking, estimate follow-up, speed-to-lead), chat or SMS agents (support, sales, internal assistants), or workflow automations with LLM steps. Also for writing a CLAUDE.md or agent instruction file, naming and describing tools and function calls, deciding whether a workload needs one agent or several, debugging an agent that follows instructions poorly, and judging whether a prompt change is safe to ship. Trigger on "write a system prompt," "fix the agent's prompt," "the agent keeps doing X," "write instructions for," "describe this tool," or "should this be one agent or two." For the system around the prompt (deployment, verification, reliability evals, model routing, cost, approval gates, durability), use the agentic-systems skill instead.
---

# Agentic Instructions

Rules for writing the instructions that drive an agent. The system around those instructions - verification, evals, risk gates, routing, deployment - is `agentic-systems`. Grounded in replicated findings: over-specified prompts measurably degrade performance. Stacking requirements drops instruction-following by up to 19% (Yang et al. 2025, "What Prompts Don't Say", arXiv:2505.13360), rigid output formats tax reasoning by 10-30% (Tam et al. 2024, "Let Me Speak Freely?", arXiv:2408.02442), and roughly two-thirds of rules audited in real production prompts restated what the model already did by default. The job of an agent prompt is not to control every step. It is to give the model a clear goal, hard boundaries, and room to comprehend.

## Core method: write at the right altitude

Every instruction lives at one of three altitudes. Pick deliberately.

1. **Goal altitude (default):** State the outcome, the success criteria, and the context the model needs. Let it choose the steps. Use this for anything requiring judgment: conversation handling, writing, debugging, triage.
2. **Constraint altitude:** Hard rules that must never break, stated as boundaries, not procedures. "Never quote a price below $X." "Never confirm a booking without an address." Keep the list short. Every constraint added dilutes attention on the others.
3. **Procedure altitude (rare):** Exact step-by-step scripts. Only for compliance-critical or genuinely deterministic sequences (payment capture, legal disclosures). If a procedure can live in code instead of the prompt, put it in code. When exact wording is legally required (consent language before collecting a phone number, mandated disclosures), procedure altitude is correct: quote the line, mark it "say exactly", state when it fires, and eval for verbatim delivery, because an unmarked script reads as a style suggestion and gets paraphrased. One scripted sentence is not a license to script the conversation around it.

The most common failure is writing everything at procedure altitude. Symptoms: the prompt reads like a flowchart, contains nested if/then branches, tries to anticipate every caller phrasing, and the agent still breaks on inputs the flowchart missed. The fix is almost always to delete, not add. A worked before/after rewrite is at the bottom of `assets/prompt-skeletons.md`.

## Drafting workflow

When writing or revising any agent prompt:

1. **Read the relevant reference file first.** Voice agent → `references/voice-agents.md`. Chat, SMS, or email agent → `references/chat-agents.md`. Task, follow-up, or outbound agent → `references/task-agents.md`. Claude Code / CLAUDE.md → `references/coding-agents.md`. Architecture decision (one agent or several) → the architecture section below.
2. **For a revision, detect and inventory the contract surface before touching anything.** Scan the existing prompt for every template variable (any {{placeholder}} or platform substitution syntax), every capture/output field name, and every tool name it references. These bind to code and platform config outside the prompt; the model revising the prompt does not get to rename them. Carry each one into the rewrite byte-exact, and list the inventory with the diff so a dropped or altered one is visible at review. If a better data shape would genuinely help (for example a computed variable replacing in-prompt arithmetic), propose it as a separate platform change alongside the rewrite, never as a silent substitution inside it.
3. **For a new prompt, start from the matching skeleton in `assets/prompt-skeletons.md`.** Then write the goal block first: who the agent is, what outcome it must produce, what a successful interaction looks like. 3-6 sentences.
4. **Add hard constraints as a short flat list, each with its reason.** "Never quote below the cost floor because a job under it loses money" outperforms the bare rule: the model applies the reason to cases the rule did not anticipate. If the list passes ~10 items, question each one. Move anything enforceable in code (function gating, validation, routing) out of the prompt.
5. **Add 2-4 canonical examples** of ideal handling, including one hard case. Examples teach behavior more reliably than rules describing behavior. But examples must earn their place like rules do: each one should demonstrate judgment a rule cannot convey (tone, a hard case, a tricky boundary). Delete any example that shows the model doing what it would do anyway, and any that repeats a case another example already covers; examples are the most token-expensive bloat in a prompt, and on voice they cost latency every turn. If the behavior is fully captured by a one-line rule, write the rule and skip the example.
6. **Cut ruthlessly.** For each remaining instruction ask: does this change behavior, or does it just make the author feel safe? Duplicated rules, hedges, and restated obvious behavior all cost attention. Shorter prompts are not lazier; they are higher-signal.
7. **Never ship a prompt change without the eval pass** in `references/evaluation.md`. A change that fixes one transcript can silently break five others.

## Debugging workflow

When an agent misbehaves, diagnose before touching the prompt. Most "prompt bugs" are not wording problems.

1. **Reproduce from real transcripts.** Collect the failing cases, note the first thing that went wrong in each, and bucket by failure type. Fix the biggest bucket first, not the most recent complaint.
2. **Classify before editing:**
   - Missing context: the agent never had the fact it needed. Fix: add the fact once, in the right place.
   - Wrong tool call or tool misuse: almost always a tool name or description bug, not a prompt bug. Fix the description or the error message.
   - Missing verification: the agent could not tell it was wrong. Fix: give it a check (validation, a test, a confirm-back step).
   - Constraint dilution: too many rules competing for attention. Fix: delete and merge before adding anything.
   - Only when none of these apply is it a wording problem.
3. **Make the smallest change that fixes the reproduced cases,** run the eval pass, and add each failure to the regression set. Adding rules to compensate for missing context or verification is the classic over-specification trap.

## Context engineering

The prompt is one input among several. Manage the whole context:

- **Curate, don't dump.** Give the agent the minimum context that lets it act well. Retrieval and just-in-time lookups beat pasting everything upfront. Long contexts degrade recall of middle content ("lost in the middle").
- **Position matters.** Attention is U-shaped: the start and end of the prompt get the best recall, the middle the worst. Order prompts goal-first, hard rules second, examples last, and never bury a constraint in the middle of a long prompt.
- **Put stable identity and rules in the system prompt.** Put per-call or per-task data (customer record, job details, calendar state) in structured context blocks or tool results, not woven into the system prompt.
- **State lives outside the model.** Anything that must survive turns or sessions (booking state, follow-up stage, customer history) belongs in a database the agent reads and writes through tools, never in "remember that..." prompt text.

## Tool design

- One tool per real capability. Consolidate overlapping tools; keep a single agent to roughly 5-15 well-differentiated tools, because measured degradation past ~20 behaves like a cliff, not a slope. If an agent has 30 tools, that is an architecture problem, not a prompting problem. Litmus: if a human engineer could not say definitively which tool applies, the agent cannot either.
- Tool names and descriptions are prompts. Name by intent (`book_job`, not `api_post_v2`), describe when to use it and when not to, and document every parameter. Most "the agent called the wrong tool" bugs are description bugs.
- Enforce guardrails in tools, not prose. A rule like "never book outside the service area" is stronger as validation inside `book_job` that returns a clear error than as a prompt line the model may deprioritize under pressure.
- Return errors the model can act on: what failed, why, and what to try. Silent failures and raw stack traces produce loops.

## Architecture: default to one agent

Start with a single agent and a good prompt. Escalate only when forced:

- **Single agent + tools:** default. Handles the large majority of real workloads, including production voice agents.
- **Deterministic workflow with LLM steps:** when the sequence is genuinely fixed (intake → enrich → draft → send). Code owns the flow; the model owns judgment inside steps.
- **Multi-agent:** only for workloads that parallelize cleanly with low shared context (broad research, independent subtasks). Multi-agent systems fail primarily through context fragmentation: workers acting on inconsistent views of the task. If subtasks need to know what each other decided, keep one agent. Budget honestly: orchestrator-worker systems run at roughly 15x the tokens of a single conversation. When one agent must hand off to another, share the full trace, not a summary; actions carry implicit decisions, and summaries drop them.

When models improve, revisit scaffolding. Structure added to compensate for a weaker model becomes a ceiling on a stronger one. Prefer the simplest scaffold the current model can carry. The test: if model capability doubled tomorrow, would this system get simpler and better without a refactor? If not, the scaffolding is fighting the model. Treat prompts and workarounds as disposable; what endures is good tools, eval infrastructure, and clean context management.

## Meta-prompting rule

Never accept AI-generated prompt text on faith, including your own. AI-written prompts fail in a predictable way: they optimize for looking thorough (long, exhaustive, procedural) rather than performing well, which recreates the over-specification problem automatically. AI-assisted prompt writing is only valid inside a measurement loop: draft → run against test cases → compare → keep only what measurably wins. If no eval set exists, build one before optimizing. See `references/evaluation.md`.

## Output standards (always apply)

- Preserve contract surface. Detect the template variables ({{like_this}}), capture/output field names, and tool names present in any prompt being revised, and keep them byte-exact in the output. Never invent, rename, or drop one in a rewrite (full workflow in Drafting step 2).
- No em dashes anywhere in generated prompts, instructions, or docs. Hyphens in compound words are fine.
- Direct, dense wording. No filler, no hedging, no restating.
- Customer-facing agent language at a 7th-8th grade reading level.
- Prompts are versioned artifacts: date, change note, and the eval result that justified shipping.

## Reference files

- `references/voice-agents.md` - Real-time voice specifics: prompt structure for real-time voice platforms, latency, turn-taking, ASR errors, interruptions, edge callers, function calling mid-conversation.
- `references/chat-agents.md` - Text, chat, and messaging agents: channel style rules (SMS/webchat/email), async multi-day state, send-layer compliance, human takeover, text-medium escalation.
- `references/task-agents.md` - Follow-up, outbound, and workflow agents: triggers, idempotency, structured outputs, human-in-the-loop checkpoints, error recovery.
- `references/coding-agents.md` - Claude Code: CLAUDE.md rules, plan-first workflow, verification loops, what to specify vs. leave open.
- `references/evaluation.md` - The mandatory pre-ship eval pass, transcript review method, regression sets, LLM-as-judge usage, and the meta-prompting loop in full.

## Companion skill

A good prompt is necessary and not sufficient. Once the instructions are written, the agent still needs a system around it: a verification layer that checks the output before anything real happens, an eval harness that measures reliability across repeated runs rather than single tries, risk tiers that force a human approval gate on any action touching money, the ledger, published content, or legal output, model routing that puts each subtask on the cheapest model whose worst mistake the verifier can catch, durable execution so a long run survives a crash, and tracing so failures are diagnosable at all. That is `agentic-systems`. Use it whenever the question moves from "what do I tell this agent" to "how do I make this agent reliable in production."
