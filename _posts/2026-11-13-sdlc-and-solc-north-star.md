---
layout: post
title: "The North Star for SDLC and Software Operations in the AI Era"
date: 2026-11-13 09:00:00 -0000
categories: ai
description: "Humans stop writing and reviewing code. They write intent and verification, and agents do everything in between. What the development and operations lifecycles look like when agents build and run software, the guardrails that make it safe, and what's still unsolved."
---

In the AI era, humans stop writing and reviewing code. They write intent and verification, and agents do everything in between.

Most teams today use AI as a faster keyboard. An engineer prompts, the model writes code, and the engineer reviews every line before it ships. That is a real productivity gain, but it keeps the human as the bottleneck. When an agent can produce in an hour what a team produced in a week, line-by-line review doesn't scale.

In [When Code Is Cheap]({% post_url 2026-11-09-when-code-is-cheap %}) I wrote about how an individual engineer's skills shift. This post zooms out to the whole system. It sketches the north star: what the software development lifecycle (SDLC) and the software operations lifecycle (SOLC) look like when agents do the building and running. It also covers the guardrails that make it safe and the problems that are still open.

## Five levels of AI-native software delivery

Self-driving cars use levels L0 to L5 to describe autonomy. Software delivery needs the same vocabulary, so teams can say where they are and what the next step is.

| Level | Who writes code | Who reviews | Who runs production | Human role |
| --- | --- | --- | --- | --- |
| **L0: Assisted** | Humans, with autocomplete | Humans | Humans | Author |
| **L1: Pair** | Humans and agents, task by task | Humans, every PR | Humans | Author and reviewer |
| **L2: Delegated** | Agents, from tickets | Humans, every PR | Humans, with agent triage | Reviewer |
| **L3: Spec-driven** | Agents, from approved specs | Agents; humans review specs, not diffs | Agents first, humans paged | Spec approver |
| **L4: Risk-tiered** | Agents | Agents; humans gate only high-risk changes | Agents resolve most incidents | Intent author and verifier |
| **L5: Intent-driven** | Agents | Verification gates and agents | Agents, humans for novel or major incidents | Intent author, system designer |

Most teams are at L1 or L2 today. The rest of this post describes L4 to L5: the north star.

## The SDLC north star

In the target state, the human artifacts are intent and verification. Everything else is generated, checked and shipped by agents.

### 1. Intent is the source of truth

Humans write requirements in natural language as Markdown files, stored in version control next to the code. A good intent file covers:

- What the system should do, and for whom.
- Constraints: performance, security, compliance, cost.
- How to verify it: acceptance cases, edge cases, failure modes.

Code becomes a derived artifact, the way a compiled binary is derived from source. Two properties follow from that:

- **Traceability.** Every design decision, task and line of code links back to a clause in the intent. When the intent changes, the system knows what to regenerate and re-verify.
- **Drift detection.** If code changes without a matching intent change, for example after a hotfix, the system flags the divergence.

### 2. Agents write the design spec, and humans approve it

A planning agent reads the intent and produces a technical design: architecture, data models, interfaces, migration plan and risks.

This is the right place for a human checkpoint. Reviewing a two-page spec is high-leverage. Reviewing 4,000 lines of generated diff is not. The spec review is the new code review.

### 3. Orchestrated execution

From the approved spec, an orchestrator agent builds a task graph. It works out which tasks depend on each other and which can run in parallel, then hands them to execution agents. Each agent works in its own isolated workspace, and the orchestrator integrates the results.

### 4. Verification is the real product of human effort

Agents test their changes against the cases in the intent. That alone isn't enough, because an agent that can see every acceptance case will optimise for passing those cases. This is the software version of teaching to the test. A robust verification layer adds:

- **Holdout tests.** Some acceptance cases are hidden from the implementing agents and only run at the gate.
- **Mutation testing.** Deliberately injected bugs check that the tests actually catch failures.
- **Property-based and fuzz testing.** These find edge cases nobody thought to write down.
- **Adversarial agents.** A separate agent's only job is to break the change.

Humans don't need to write every test. Agents can propose verification cases from the intent, and humans approve the set.

### 5. Risk-tiered autonomy replaces PR review

If humans no longer write the code, requiring a human to review every PR makes little sense. But "no review ever" is the wrong framing. The right one is **review proportional to blast radius**:

| Change type | Examples | Gate |
| --- | --- | --- |
| **Low risk** | Internal tools, UI copy, well-tested services | Fully autonomous: verification plus agent review |
| **Medium risk** | New APIs, cross-service changes | Agent review plus human spec approval |
| **High risk** | Auth, payments, data migrations, IAM and infra, anything irreversible | Human gate before merge |

Compliance has to be part of the design. Frameworks such as SOC 2 and SOX expect separation of duties for changes. Independent reviewer agents, signed provenance and complete audit trails have to be accepted by auditors, or regulated industries can't adopt this model.

### 6. Version control and CI built for agents

Today's tools assume a few hundred humans pushing a few changes a day. Agents push orders of magnitude more.

On version control, partial clone, sparse checkout and virtual filesystems such as Sapling's EdenFS already reduce the cost of a full checkout. The real gaps are workflows designed for agents:

- APIs to read individual files, write changes and open merge requests without a local clone.
- Cheap, ephemeral branches and workspaces.
- Merge queues that handle thousands of changes a day.
- **Semantic conflict detection.** Two agents can edit different files that merge cleanly but break each other's behaviour.

On CI/CD, pre-warmed workers help absorb the load. Bigger gains come from:

- Test impact analysis, so only affected tests run.
- Remote build caching and hermetic builds.
- **Eliminating flaky tests.** For humans, flakes are an annoyance. For agents, they are a trap: agents will spend cycles "fixing" failures that aren't bugs.

## The foundations that make it safe

The pipeline above only works if several foundations are in place. Without them, autonomy turns into risk.

**A machine-readable context layer.** Agents need shared memory: architecture decision records, coding conventions, service ownership, a map of the system, and explicit "never do this" constraints. This keeps a hundred agents consistent with each other and with the system's history.

**Progressive delivery as the main safety net.** When pre-merge human review goes away, post-merge safety has to get stronger. Feature flags, canary releases and automatic rollback on SLO regression mean verification continues in production. A bad change reaches 1% of traffic, not 100%.

**Security and supply chain.** Agents are a new attack surface. The north star needs:

- An identity for each agent, with least-privilege access.
- Defences against prompt injection hidden in issues, docs or dependencies.
- Checks for hallucinated package names, which attackers register in advance (slopsquatting).
- Strict secrets handling.
- Signed provenance for every agent-made change.

**Fighting codebase entropy.** Agents generate duplication and drift at scale, because each one optimises locally. Periodic refactoring agents and fitness functions, such as architecture rules enforced in CI, keep the codebase coherent.

**Economics.** Agent swarms are not free. Each task needs a token and compute budget, and teams should track cost per feature delivered the way they track cloud spend today.

## The SOLC north star

In operations, agents become the first responders. They receive every alert, triage it, and fix what they safely can. Humans get paged only for incidents that need human judgement.

### Agents as on-call, by default

The goal is a sharp reduction in human on-call load. An on-call agent correlates the alert with recent deploys, flag changes and config changes. It reads logs and traces, forms a root-cause hypothesis and acts on it.

### Graduated remediation autonomy

Not every fix carries the same risk, so not every fix gets the same autonomy:

1. **Diagnose.** Gather evidence, correlate signals, propose a root cause. Always allowed.
2. **Reversible actions.** Roll back, restart, scale out, toggle a feature flag. Safe to automate, because they can be undone in seconds.
3. **Code fixes.** These go back through the full SDLC pipeline, with verification. An agent should never hot-patch production around the gates.

### Observability built for agents

Most observability today is built for human eyes: dashboards and charts. Agents need:

- Structured, queryable APIs over logs, metrics and traces.
- High-cardinality data, so they can slice by customer, region or build.
- Change events (deploys, flag flips, config changes) on the same timeline as telemetry.

### Clear escalation criteria

An agent should page a human when any of these is true:

- Customer impact crosses a defined threshold.
- The agent's confidence in its diagnosis is low.
- The incident is novel and matches no known pattern.
- The next action would be irreversible.

### Guardrails on the agents themselves

Autonomous responders need their own safety controls:

- A kill switch and rate limits on agent actions.
- Caps on blast radius, such as a maximum number of hosts touched.
- Protection against feedback loops, for example an agent fighting the autoscaler, or two agents undoing each other's work.

### Earning trust through shadow mode

No team will hand production to an agent on day one. Start in shadow mode: the agent runs alongside the human on-call and proposes actions without taking them. Measure how often it is right. Grant autonomy one action type at a time as accuracy is proven.

### Proactive operations

The best incident is the one that never happens. Beyond reacting to alerts, agents should handle capacity planning, CVE patching, dependency upgrades and cost optimisation.

### Don't let silent fixes hide rot

If an agent restarts the same service 40 times a week, nobody gets paged, but the system has a design problem. Humans need trend reports on what agents are fixing, not just a quiet pager.

### What to measure

- Share of incidents resolved without a human.
- Escalation precision: how many pages actually needed a human.
- False-fix rate: actions that didn't fix the issue or made it worse.
- Mean time to resolve, compared with human-only on-call.

## Closing the loop: one lifecycle, not two

The biggest shift is that SDLC and SOLC stop being separate lifecycles. Every production incident produces a postmortem, new verification cases in the intent files, and updated runbooks. The next build is tested against everything production has taught the system.

![The closed loop: Intent, Design spec, Parallel build and Verify on the build side; Canary deploy, Observe, Agent on-call and Learn on the run side, with Learn feeding the next intent. Humans act on intent and the design spec.]({{ "/assets/images/sdlc-solc-closed-loop.png" | relative_url }})

Humans act at the top-left of the loop, on intent and spec. Everything else runs on agents, and every incident makes the loop stronger.

## What humans do now, and what's still unsolved

Engineers don't disappear in this model. Their work moves up a level:

- **Intent authors** decide what to build and why.
- **Verification designers** define what "correct" means, including the cases agents can't see.
- **System architects** shape the constraints, context and guardrails that agents work within.
- **Escalation responders** handle the novel, high-stakes incidents agents hand over.

That last role exposes a paradox. When the agent escalates at 3am, a human still has to understand the system deeply enough to fix it. If humans never touch the code, that skill fades. Teams will need deliberate practice, such as game days, rotations through agent-generated code, and readable design records, to keep it alive.

Several problems are still open:

- **Verification completeness.** The whole model rests on verification being good enough. How do we know when it is?
- **Accountability.** When an autonomous change causes an outage, who is responsible: the intent author, the verification designer, or the platform team?
- **Compliance.** Auditors and regulators have not yet accepted agent review as a substitute for human change approval.
- **Ambiguous intent.** Natural language is imprecise. Agents need a reliable way to ask clarifying questions instead of guessing.

## Conclusion

The north star is not "AI writes code faster." It is a system where humans state what they want and how to prove it, and agents handle everything else, in development and in production.

Getting there is less about better models and more about better scaffolding: intent files as the source of truth, verification that agents can't game, risk-tiered gates, tooling built for agent-scale throughput, and operations that feed every lesson back into the next build.

The practical first step for most teams is to move from L2 to L3. Start writing intent and verification files, review specs instead of diffs, and put an on-call agent in shadow mode. The rest follows as trust is earned.
