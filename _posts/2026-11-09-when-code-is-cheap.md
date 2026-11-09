---
layout: post
title: "When Code Is Cheap: The Engineering Skills That Matter in the AI Era"
date: 2026-11-09 08:00:00 -0000
categories: ai
description: "If agents write the code, an engineer's value moves from producing it to directing and verifying it. A breakdown of the skills that shift demands: framing problems, verifying output, owning systems, managing context, and the human traits underneath all of it."
---

Most of my code is written by agents now. I describe what I want, review what comes back, push back, and iterate. That raises an uncomfortable question: if the machine writes the code, what exactly is my job, and which skills will keep mattering?

My short answer is that an engineer's value is moving from producing code to directing and verifying it. The craft hasn't disappeared. It has moved up a level, to the work around the code: deciding what to build, checking that it's right, and owning what gets shipped. Here is how I think about the skills that shift demands.

## Directing the work

When execution is cheap, a vague intent turns into a lot of wrong code very quickly. Directing the work well comes down to three skills.

**Framing the problem precisely.** The most useful thing you can give an agent is a clear spec: what you want, the constraints, the non-goals, and what "done" looks like. That makes clear writing a core engineering skill. If you can't explain the problem in plain sentences, you probably don't understand it well enough to delegate it.

**Decomposing and delegating like a tech lead.** Working with agents looks a lot like leading a team. You split the work into pieces that can be done independently and checked, decide what to hand off and what to keep, run things in parallel, and integrate the results. It's a management skill applied to machines.

**Generating ideas and researching deeply.** Once building gets cheap, the bottleneck becomes knowing what's worth building. The engineers who stand out dig into a problem or a domain, find the gaps nobody has addressed, and work with agents to turn those gaps into concrete proposals. They also prototype ideas that used to be too expensive to try.

## Verifying the work

This half gets talked about less, and it's where good engineers are pulling ahead.

Reasoning hasn't become obsolete. It has moved. You do less reasoning to produce code and more to evaluate it. When an agent hands you a plausible 400-line diff, you have to reason about invariants, edge cases, concurrency and failure modes in code you didn't write. That's harder than writing it yourself, because you never built the mental model while you typed. Critical thinking, meaning asking why something is done a certain way and poking holes in it, depends entirely on that reasoning ability.

**Reading carefully, not just reading fast.** Most of our work now is reading natural-language explanations and quickly building mental models of unfamiliar concepts. Speed helps, but calibration matters more. The real danger is fluent, confident output that you skim and accept. Knowing when to slow down and read line by line is the skill.

**Designing verification.** The question used to be "can I build it?" Now it's "how will I know it's correct?" That covers designing tests, evals, invariants, canaries and rollout checks, and building feedback loops the agent can run itself. Engineers who are good at this get much more out of agents, because the agent can check its own work against something real.

**Taste.** Agents give you a working solution. They're much worse at picking the right one: what's simple enough, what will age well, what's over-engineered, what should be deleted instead of added. Taste comes from experience, and it's what separates "it works" from "it's good."

## Owning the system

**Systems thinking.** Agents work well locally and poorly globally. They'll write a clean function and miss that it adds a synchronous call on a hot path, or breaks a consistency guarantee three services away. Someone still has to own boundaries, data flow, failure domains, consistency and cost. Deep systems knowledge is worth more now, not less.

**Keeping your fundamentals sharp on purpose.** You can only catch an agent's mistake in a domain you understand at least one level below where it's working. Outsource everything and your ability to judge the output slowly decays. I've started doing some things by hand on purpose, the way pilots still fly manually so their skills don't fade.

**Knowing when to stop delegating.** Some problems get worse with an agent in the loop: a subtle production incident, a genuinely novel algorithm, a bug the agent keeps confidently circling. Spotting that early and switching to deep focus yourself can save hours.

**Ownership.** The agent wrote it, but you shipped it. Standing behind your code in a postmortem, and bringing the security, privacy and operational judgment that goes with it, can't be automated.

## Managing context

We now jump between more projects, with fewer low-level details in each. My first instinct was that the skill is holding all that context in your head for longer. I now think that's fragile. Memory fades, and an agent can't read your mind.

The better approach is to get context out of your head: design docs, decision records that explain why things are the way they are, and well-maintained agent context files. Then both you and your agents can reload it on demand.

What does need to live in your head is a compressed model of each system: its shape, its key constraints, and where the bodies are buried. Knowing what to keep in your head and what to write down is the real skill.

## The human side

**Curiosity, paired with rigor.** If I had to pick one trait, it would be curiosity: the urge to understand how things work, dig deeper, and experiment. The industry is moving too fast for anyone who waits to be taught. But curiosity is a multiplier on rigor, not a replacement for it. Curiosity without rigor gives you lots of interesting prototypes and very few reliable systems.

**Comfort with uncertainty.** Tools, models and best practices now change every few months. What worked last quarter may be obsolete. Being able to act on incomplete information, hold opinions loosely, and relearn without anxiety is a real professional skill now.

**Communication and influence.** If building is cheap, the bottleneck becomes agreeing on what to build: aligning stakeholders, scoping, persuading, saying no. You don't have to be the loudest person in the room for this. A sharp design doc or a well-argued proposal does a lot of the persuading for you.
