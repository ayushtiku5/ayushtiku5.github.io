---
layout: post
title: "We're Writing Skills and Prompts for Agents Like Thousand-Line Switch Statements"
date: 2026-08-05 03:00:00 -0000
categories: ai
description: "Enumerating edge cases as rules in a skill only covers the failures you've already seen. Push determinism into the execution layer and keep the prompt layer for judgment."
---

I keep seeing the same pattern in agentic systems right now: engineers write a prompt or skill, it breaks on an edge case, they add a rule for it. Breaks again, add another rule. Six months in, you have a 2,000-word skill with forty conditional branches trying to force a non-deterministic system into deterministic behavior. It doesn't work, and it makes things worse.

Every rule you add is competing for the model's attention with the rules that actually matter. Bloat doesn't just slow things down, it dilutes the instructions you need followed most.

Enumerating cases only covers the failures you've already seen. New ones show up outside your rule set, and now the agent has no principle to fall back on, just a list it doesn't match.

You end up debugging natural language like it's code, except you can't set a breakpoint in a paragraph.

The fix isn't "write better prompts." It's recognizing that most of what people are cramming into skills as rules isn't judgment, it's computation. Validation, formatting, retries, sequencing, idempotency checks. That's not a rule for the agent to interpret every time, sampling its way to compliance. That's a script. Give the agent a tool that just does it correctly, every time, testably.

What's left in the skill after that extraction is small, and it's the part LLMs are actually good at: judgment calls. Which approach fits this context. When to ask instead of assume. What's genuinely ambiguous.

Push determinism into the execution layer. Keep the prompt layer for reasoning. Stop trying to make a probabilistic system behave like a deterministic one by writing harder, build the tools that make determinism unnecessary to write about at all.

Hope this helps.
