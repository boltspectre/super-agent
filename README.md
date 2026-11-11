# Super Agent · Multi-Agent Review & Fix Workflow

> A **single-file, zero-dependency** agent skill that turns your main agent into a *super agent*: it stops writing code itself and instead plans, dispatches, merges and validates.
>
> Works with **Claude Code**, **CodeBuddy**, and any agent tool that supports subagents (a Task / delegate tool).

[简体中文](README.zh-CN.md) | **English**

![type](https://img.shields.io/badge/type-Agent%20Skill-blue)
![works](https://img.shields.io/badge/works%20with-Claude%20Code%20%7C%20CodeBuddy%20%7C%20any%20subagent--capable%20agent-8a2be2)
![deps](https://img.shields.io/badge/dependencies-none-brightgreen)

> **Made for complex tasks.** Multi-file or multi-module work where risk must be mapped before a single line changes, and where "is it actually fixed?" cannot be answered by the agent that did the fixing — reviews, migrations, refactors. The more complex the task, the more this pays off: work one context can no longer hold gets split across subagents with separate roles, then verified as a whole.
>
> For small tasks, skip it and edit directly (see [When not to use it](#when-not-to-use-it)).

---

## What is this

Normally, when you give an agent a big task, **one model plays three roles at once**: the one who finds the bugs, the one who fixes them, and the one who declares them fixed.

This skill splits those roles apart. The main agent only *judges and dispatches*; discovery, editing and verification each go to a different subagent, and nobody grades their own homework.

All you have to do is **describe the task the way you normally would**. Task decomposition, risk routing and acceptance are the super agent's problem.

```
You:   Review the whole codebase and find the vulnerabilities.
Agent: Before I start — here's my understanding of the task,
       my dispatch plan, and my definition of done…
```

## The main conversation does only four things

With this skill installed, your main conversation stops being a general-purpose worker and becomes a gate:

| Action | What it does | What it refuses to do |
|---|---|---|
| **Check** | Confirms the goal, the dispatch plan, the definition of done — then dispatches | No recon by itself |
| **Accept** | Takes each subagent's delivery and checks it against the criteria | No "looks fine to me" self-certification |
| **Reject** | Sends it back with concrete reasons, looping until everything passes | No quietly fixing it itself |
| **Report** | Delivers one summary when all is done: what changed, what was verified, what remains | No process log dumped on you |

The loop is **automatic**: reject → re-verify → until done. You are not pulled in mid-flight to judge half-finished work.

**The context you save is spent on remembering your requirements.** Reading files, tracing implementations, dead-end attempts — all of that stays inside each subagent's own window. The main conversation keeps only the issue list, the diff verdicts and your own words. With the window free of code-level noise, it still remembers the conventions you set earlier in a long session: naming rules, architectural boundaries, interfaces that must stay compatible, directories that must not be touched.

## What it solves

| Problem | Consequence | What this skill does |
|---|---|---|
| Multi-file tasks go sideways | Fixing while still exploring; nobody mapped the risk first | Layer 0 plans, Layer 1 recon is read-only, you get a graded issue list before any edit |
| Self-verification | "Fixed" means nothing when the fixer decides | Layer 3 verifies with an **independent read-only** agent, never the same instance |
| Context pollution | Dumping a whole review transcript into the fixer wastes tokens and invites scope creep | Pass `ID + location + suggested direction` only |
| Main window eaten by coding detail | It fills up, and the conventions you set earlier fade out of a long session | Coding and trial-and-error stay in subagent windows; the main conversation keeps judgment, verdicts and your requirements |
| One agent per tiny issue | Overhead exceeds the benefit | Merge similar `simple` issues into one batch |
| Big changes with no design | Highest rework cost of all | `complex` issues require an approved plan first |
| **Subagents racing on one file** | Last write wins — **code is silently lost** | File-ownership serialization + batching by file/module (below) |

## Concurrency safety

When several subagents write to the same file at the same time, writes are last-write-wins: earlier edits are silently overwritten while every agent believes it succeeded. It is one of the hardest bug classes in multi-agent work.

This skill shrinks that window as far as an agent workflow can:

1. **Centralized write ownership** — at any moment a file belongs to exactly one executor. The main agent never hands the same file to two subagents at once.
2. **Batching by file/module** — one batch never jumps across unrelated files, which removes cross-overwrite by construction.
3. **Sequential dispatch** — batches that could overlap are issued in order; writers are never started concurrently on overlapping paths.
4. **Verifiers are read-only** — the acceptance step never holds write permission, so it cannot compete with executors.
5. **Diff check after every batch** — a loss is caught inside the smallest possible scope instead of blowing up at the end.
6. **Rework is re-routed, not re-sent** — failed items go back through risk routing instead of being pushed back to the original executor with stale context.

When real parallelism is needed, partition **by file**: different files may run in parallel, the same file is strictly serial.

## How it works

```
One sentence from you
   │
   ▼
[Layer 0] Pre-flight confirmation
   │   Main agent restates the goal + dispatch plan + definition of done → waits for you
   ▼
[Layer 1] Read-only recon (read/search tools only)
   │   Structured issue list: C1..Cn (severity + risk tier + location + direction)
   ▼
[Layer 2] Risk-tiered fixes
   │   simple  → lightweight path, merge similar issues into one batch
   │   batch   → new executor, receives the issue list only
   │   complex → design proposal first, then implement
   ▼
[Layer 3] Independent read-only verification (fresh context)
   │   Per item: resolved? / new issues introduced? / pass or rework
   ▼
   ├── all pass ──► summarize and deliver
   └── rework   ──► back to Layer 2 with risk re-routing
```

**Risk tiers are assigned during recon**, so Layer 2 routing needs no extra round trip.

| Tier | Criteria | Handed to |
|---|---|---|
| `simple` | Single file, ≤5 lines, unambiguous | Reuse the recon agent (if it has write access); otherwise the main agent applies them as one batch |
| `batch` | Multiple files, uniform pattern | A new executor fed only the issue list |
| `complex` | Needs redesign or architectural judgment | A new executor with fuller context, **plan first** |

## Install

### Claude Code

```bash
git clone https://github.com/<your-name>/git-super-agent.git
mkdir -p ~/.claude/skills
cp -r git-super-agent/super-agent ~/.claude/skills/super-agent
```

Windows (PowerShell):

```powershell
git clone https://github.com/<your-name>/git-super-agent.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse .\git-super-agent\super-agent "$env:USERPROFILE\.claude\skills\super-agent"
```

For project scope only, put it under `.claude/skills/super-agent/SKILL.md`.

### CodeBuddy

- **Import (recommended)**: `Settings → Skills → Import Skill`, select the `super-agent` folder.
- **Project scope**: copy to `.codebuddy/skills/super-agent/SKILL.md` in the workspace (shared with the repo).
- **User scope**: place it under the user-level `.codebuddy/skills/` directory (applies to all projects).

### Other agent tools

Any tool with subagent / task-delegation support works. Drop `SKILL.md` into whatever directory that tool uses for skills, rules or instructions — or paste it as system instructions if it has no skill mechanism. The tables above show the CodeBuddy mapping; substitute equivalent capabilities elsewhere:

| Role | Requirement |
|---|---|
| Read-only recon agent | Can read files and search, cannot write |
| Resumable agent | Keeps context, accepts follow-up scope |
| Writable executor | Can edit files (fall back to batched main-agent edits if unavailable) |
| Independent verifier | Read-only, and not the same instance that made the edits |

## Usage

No commands to memorize. The skill's `description` matches your intent and loads automatically — just talk normally:

| You say | What happens |
|---|---|
| "Review the whole system and find the vulnerabilities." | Full three-layer run, delivered with per-item verification |
| "Migrate the auth module to TypeScript, and verify the result." | Recon → batched migration → independent acceptance |
| "Find and fix non-idiomatic Vue patterns across these components." | Judged against the Vue-native standard, fixed, then re-checked |
| "Just give me the report, don't touch the code." | Layer 1 only, then summary |
| "Go ahead, no need to confirm." | Skips Layer 0 and starts immediately |

## Repository layout

```
git-super-agent/
└── super-agent/
    └── SKILL.md    # the entire logic — one file, no scripts, no dependencies
```

## When not to use it

- Tiny fixes in a single file or function: editing directly is faster, subagents are pure overhead
- Pure Q&A or explanation tasks
- Report-only requests where you explicitly do not want code changed: run Layer 1 only

## Design notes

**Why doesn't the main agent write code?**
Two reasons. First, "writing" and "accepting" must be different parties, otherwise it's self-certification. Second, context budget — coding burns a large share of the window, and delegating it frees the main conversation to spend that window on judgment and on your requirements rather than on the 37th failed build attempt.

**Why pass only the issue list, not the review conversation?**
A full transcript is mostly hypotheses that were already ruled out. Handing it to the fixer burns context and tempts it to "helpfully" change things nobody confirmed.

**What if the environment has no writable subagent?**
Don't fake it. The main agent applies batches itself — but verification still runs on an independent read-only agent. That part is non-negotiable.

**Why confirm before starting?**
A misread goal is the most expensive failure in a multi-agent pipeline: everything downstream is wasted work. The confirmation lists bullet points only, so the cost of saying "yes" is near zero.

---

If this saved you the afternoon you'd otherwise spend hunting a silently overwritten change, a star is enough. MIT License.
