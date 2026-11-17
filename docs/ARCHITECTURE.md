# Architecture

The whole system is a **control repository** plus the repositories it looks after. The control repository contains no product code. It contains what a human manager would otherwise keep in their head or in a wiki, written so that an AI agent can read it and obey it.

## The pieces

| Piece | What it is | Why it exists |
|---|---|---|
| Operating manual | One file the agent reads first in every run: mission, file map, hard rules, language, tooling | The agent starts every run with no memory; the manual is its memory and its contract |
| Strategy | The idea rubric, kill criteria, launch checklist | Decisions about what to build are made against written criteria, not mood |
| Playbooks | One per kind of run (daily, work, maintenance, scout, review, build) | Each kind of run has a goal, steps, stop conditions and anti-patterns |
| State | A run log, a portfolio file (tier and next step per repo), a backlog of scored ideas, one-line learnings, the work queue | Work continues across runs; state is plain text in git, so every change is reviewable |
| Inbox | A file of decisions and tasks that belong to the human, in plain language, with a recommendation each | The human-in-the-loop channel: tick or reply, nothing else |
| Scripts | Small dependency-free programs: snapshot (metrics), queue (work list), contribs (measurement), lock, doctor, secret scan, launch check, approvals | Everything deterministic is code, so the agent spends its judgement only where judgement is needed |
| Configuration | Owner, caps per tier, daily target, themes and exclusions, standing approvals | Behaviour changes by editing one file |

## Run types

<!-- gen:runs -->
| Run | When | Purpose |
|---|---|---|
| Daily | 09:03 | Health first: CI, security alerts, broken install paths, new issues and pull requests; then the highest-value work |
| Work | 12:37, 15:37, 18:37, 21:37 | Turn the ranked queue into verified commits until a stop condition |
| Maintenance | Tuesday and Friday 07:30 | Deep health check of the public flagship, including a live canary against the real interface it wraps |
| Scout | Monday and Thursday 07:30 | Research new ideas from several signal sources, score them with the rubric, start the best as a **private** project while a slot is free |
| Review | Sunday 18:00 | Critical review of every repository, tiers, proposals to archive, a weekly report in the owner's language |
<!-- /gen:runs -->

Runs never overlap: a lock file is taken first; a run that finds it busy logs "skipped" and stops.

## The work queue

`scripts/queue.mjs` builds the queue from facts, not from imagination. Each line has its evidence, and a line the agent may not decide alone is marked as waiting for the human instead of being hidden.

Sources: failing CI; open pull requests and issues; outdated workflow actions (compared with each action's latest release); missing basics (licence, README sections, CI, changelog, contributing guide, security policy, templates, dependency updates); TODO markers; source modules that no test mentions; each project's own next step; repositories nobody has classified yet; candidates for the curated list (found through repository search, filtered for relevance, activity and licence).

Ranking: priority first (fires, people, maintenance, improvements), and inside a priority the **repository that was touched longest ago comes first**. That single rule is what makes every repository grow evenly instead of the favourite getting all the attention.

Items the agent checked and judged not to be work (a false alarm, a rejected candidate) are written to a dismissed list with the reason, so the next run does not spend time on them again.

## Measurement

`scripts/contribs.mjs` asks GitHub how many contributions the day has (commits, pull requests, issues, reviews) and compares it with the target range in the configuration. It also reported something useful early on: private repositories were not counted at all, because the profile setting for private contributions was off. A measurement that finds a blind spot in the thing it measures is worth more than the number.

## What a commit is

One verifiable change: a fix, a feature, tests that pin real behaviour, a corrected claim, a dependency bump, a verified list entry. Tests must be able to fail (break the code once on purpose and see it go red). A change is never split into several commits to count more, and nothing is touched for its own sake. The target number is a yardstick for real work; when the queue is empty, the run stops and reports the shortfall.

## Where the human stays in control

See [Guardrails](GUARDRAILS.md). In short: hard rules the agent cannot override, standing approvals the owner granted explicitly and can withdraw by deleting one key, and an inbox for everything else.
