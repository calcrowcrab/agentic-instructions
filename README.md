# Agentic Design

**Stop writing flowcharts for things that can think.**

An evidence-based Claude skill for designing, debugging, and evaluating AI agent prompts: voice agents, chat/SMS agents, coding agents (CLAUDE.md), and workflow automations.

## The idea

Most agent prompts fail from over-specification, not under-specification. Stacking requirements measurably drops instruction-following (up to 19%, Yang et al. 2025, arXiv:2505.13360), rigid output formats tax reasoning by 10-30% (Tam et al. 2024, arXiv:2408.02442), and roughly two-thirds of rules audited in real production prompts restated what the model already did by default.

This skill teaches the alternative: write at the right altitude. Give the agent a clear goal, a short list of hard boundaries with their reasons, a few canonical examples, and room to think. Enforce everything else in code, and never ship a prompt change without an eval pass.

## What's inside

```
agentic-design/
├── SKILL.md                     Core method: altitude, drafting workflow,
│                                debugging workflow, context engineering,
│                                tool design, architecture, meta-prompting
├── references/
│   ├── voice-agents.md          Real-time voice: latency, ASR, turn-taking,
│   │                            barge-in, function calling on live calls
│   ├── chat-agents.md           Text/SMS/email: channel style rules, async
│   │                            multi-day state, send-layer compliance,
│   │                            human takeover
│   ├── task-agents.md           Follow-up, outbound, and workflow agents:
│   │                            idempotency, structured outputs, HITL gates
│   ├── coding-agents.md         Claude Code: CLAUDE.md rules, plan-first,
│   │                            verification loops, hooks over prose
│   └── evaluation.md            The mandatory pre-ship eval pass, regression
│                                sets, error analysis, LLM-as-judge,
│                                meta-prompting done right
└── assets/
    └── prompt-skeletons.md      Starting skeletons (voice, task step,
                                 CLAUDE.md) plus a worked before/after
                                 rewrite from procedure to goal altitude
```

## Install

**One-click:** open `agentic-design.skill` in a Claude chat and click Save skill.

**Claude Code / Cowork:** clone this folder into your skills directory:

```bash
git clone <this-repo> ~/.claude/skills/agentic-design
```

**Claude.ai:** upload the `.skill` file, or add the folder under Settings > Capabilities > Skills where available.

## Use

The skill triggers on requests like:

- "Write a system prompt for a voice agent that books appointments"
- "The agent keeps quoting prices it shouldn't. Fix the prompt."
- "Write a CLAUDE.md for this project"
- "Should this be one agent or two?"
- "Is this prompt change safe to ship?"

## How it was built

Distilled from a research pass across peer-reviewed work (instruction-stacking degradation, format-tax studies, lost-in-the-middle), lab guidance from Anthropic and OpenAI on agent building and context engineering, and practitioner engineering posts on voice latency, tool design, and multi-agent tradeoffs. Changes to the skill are A/B tested: the same tasks run with the previous and candidate versions and are graded against fixed, observable criteria before shipping. The skill practices what it preaches.

## License

MIT, copyright Chris Weir. See LICENSE. Use it, adapt it, ship it; keep the copyright notice with it.
