---
name: product-council
description: Use when a startup idea, product idea, feature bet, or business decision needs a full multi-perspective evaluation before committing — product, customer, marketing, investor, technical, and adversarial review synthesized into one go/no-go decision with a 30-day plan. Triggers include "should we build this", "evaluate this idea", "run the council", "is this worth doing".
---

# Product Council

## Overview

Runs a panel of six expert seats over one idea, then hands every report to a
CEO seat that must commit to a decision.

Core principle: **the value is in the disagreement.** Each seat is deliberately
biased and must not be softened toward consensus. The synthesis happens once,
at the end, in the CEO seat — never inside the individual reports.

**The seats:**

| Seat | Skill | Bias it contributes |
|---|---|---|
| Product Manager | `product-manager` | Is the problem real; what's the MVP |
| Customer | `customer-agent` | Would a real buyer say yes |
| Marketing | `marketing-agent` | Can we reach and price them |
| Investor | `investor-agent` | Is it a business worth funding |
| Technical Architect | `technical-architect-agent` | Can it be built, at what cost |
| Devil's Advocate | `devil-advocate-agent` | Why it fails |
| CEO | `ceo-decision-agent` | The decision |

## When to Use

- Before committing real time or money to an idea
- Comparing two directions and needing a structured read on each
- A team keeps circling the same decision without resolving it

**Not for:** small implementation choices, or questions with a single expert
answer (just use that seat's skill directly — every seat works standalone).

## Step 1 — Gather the Idea

If the user hasn't already described the idea in enough detail, ask for it.
Request these, and note which are missing rather than inventing them:

1. **The idea** — what it is, in a few sentences
2. **Who it's for** — the intended user or customer
3. **The problem** it claims to solve
4. **Business model** — how it makes money (if known)
5. **Stage** — idea / prototype / launched, and any evidence so far
6. **Constraints** — team size, budget, timeline, must-use technology
7. **Decision at hand** — what the user will do with this report

Only #1 is required. Ask once for the rest, accept whatever comes back, and
mark the gaps as `[Unknown]` in the brief. Do not stall the council over a
missing field.

## Step 2 — Write the Brief

Write a single shared brief to `council/<slug>/brief.md` (create the
directory). Every seat receives this exact text — identical input is what
makes the reports comparable.

The brief contains: the idea, target user, problem, business model, stage and
evidence, constraints, the decision at hand, and an explicit `[Unknown]` list.

**Do not add your own analysis to the brief.** A brief that already contains
conclusions biases every seat toward them.

## Step 3 — Run the Six Expert Seats

Run all six **in parallel** as subagents, in a single message with six tool
calls. They are independent — no seat reads another's output.

Give each subagent this prompt shape:

```
Read <skills-dir>/<seat-skill>/SKILL.md and follow it exactly — including its
Output Format section, which is a contract.

Here is the brief you are evaluating:

<full text of brief.md>

Return only your report in the skill's output format. No preamble, no
summary of the brief, no meta-commentary. Your entire response is the report.
```

Resolve `<skills-dir>` to wherever these skills are installed — the project's
`.claude/skills/` or the user's `~/.claude/skills/`.

Save each returned report to `council/<slug>/<seat>.md` as it arrives.

**If subagents are unavailable**, run the seats sequentially in the current
context instead: for each seat, load its SKILL.md, produce the full report,
write it to its file, and **clear it from your working focus before starting
the next** — do not let the previous seat's conclusions leak into the next
one. Order: product-manager → customer-agent → marketing-agent →
investor-agent → technical-architect-agent → devil-advocate-agent.

**If a seat fails or returns nothing**, retry once. If it fails again, carry
on with five seats and record the gap — the CEO seat is told to handle
missing reports.

## Step 4 — Run the CEO Seat

Once all six reports exist, run `ceo-decision-agent` with the brief plus the
**full text of all six reports** concatenated. Save to
`council/<slug>/ceo.md`.

Never pre-summarize the reports for the CEO seat — resolving the conflicts in
the raw reports is the whole job.

## Step 5 — Produce the Final Report

Write `council/<slug>/report.md` in exactly this format, then present it to
the user:

```markdown
# Product Council Report

**Idea:** <one line>  ·  **Date:** <date>  ·  **Decision:** <Proceed / Modify / Reject>

## Executive Decision
The CEO seat's decision, main reasons, and confidence. Lead with the verdict.

## Product Analysis
From the PM seat: problem validation, target customer, MVP recommendation,
feature priorities.

## Customer Feedback
From the customer seat, in their voice: would buy / would not buy, biggest
frustrations, missing features, verdict.

## Marketing Strategy
From the marketing seat: audience, angle, growth channels, competitive
positioning, pricing.

## Investor Opinion
From the investor seat: thesis, market opportunity, scorecard, invest
decision.

## Technical Assessment
From the architect seat: approach, architecture, effort estimate, roadmap.

## Risks & Challenges
Merged from the devil's advocate seat and every other seat's risk section.
Deduplicated, ranked, each labeled FATAL or SURVIVABLE with its owner.

## 30 Day Action Plan
The CEO seat's week-by-week table, unchanged.

## Where the Council Disagreed
The conflicts and how the CEO ruled on each. Keep this — it's the most useful
section for the reader.

## Appendix
Links to the full per-seat reports in `council/<slug>/`.
```

Each section compresses that seat's report to its decision-relevant content —
**do not soften it**. If the customer seat said "I would not buy this", that
sentence appears verbatim in Customer Feedback. Preserve `[Assumption]` and
`[Estimate]` tags; a reader must be able to see which numbers were invented.

## Execution Checklist

Create a todo for each:

- [ ] Idea gathered; missing fields marked `[Unknown]`
- [ ] `brief.md` written, analysis-free
- [ ] Six seats dispatched in parallel with identical briefs
- [ ] Six reports saved; failures retried once and gaps recorded
- [ ] CEO seat run on the full raw reports
- [ ] `report.md` written in the required format
- [ ] Disagreements preserved, not smoothed
- [ ] Presented to the user with the decision first

## Common Mistakes

| Mistake | Fix |
|---|---|
| Writing the brief with your own conclusions in it | Brief is facts only |
| Feeding seats each other's reports | Seats run independently; synthesis is the CEO's job |
| Summarizing reports before the CEO seat | Give the CEO the raw text |
| Final report reads as consensus | Keep "Where the Council Disagreed" sharp |
| Burying the verdict in section 8 | Decision goes in the header and the first section |
| Running seats sequentially by default | Parallel — six independent subagents, one message |
