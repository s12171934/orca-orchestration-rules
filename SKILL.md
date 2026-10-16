---
name: orca-orchestration-rules
description: >-
  House rules for supervised Orca orchestration — when a task deserves a tracked Run/Task
  worker at all, and how such a worker is launched, named, modelled, waited on, and closed
  out. Load it together with the `orchestration` skill whenever the user asks to supervise,
  monitor, wait for results, track completion, coordinate a DAG or decision gate, or fan
  work out across several workers ("오케스트레이션으로 돌려줘", "워커 여러 개로 병렬
  처리해줘", "결과 받아서 정리해줘", "진행 상황 추적해줘"). For a plain isolated-work
  launch with no supervision, use `orca-work-placement` instead — creating a worktree or
  handing work to another agent is not by itself orchestration.
---

# Orca Orchestration Rules

Supervision is a mode the user opts into, not a default. This file says when to enter it and
what the coordinator owes once it has. It does not restate the command surface — load the
version-matched guide with `orca skills get orchestration` and take every subcommand and
flag from there, never from memory or from the snippets below.

## 1. Enter supervision only when asked, and know what it costs

Use a tracked Run/Task/Dispatch when the user asks to **supervise, monitor, wait for
results, track completion, coordinate a DAG, use a decision gate**, or run a fan-out the
coordinator will synthesize. Everything else — "브랜치 따서 해줘", "다른 에이전트한테
넘겨줘", "워크트리 만들어서 작업해줘" — is a plain launch: `orca-work-placement`.

The distinction is mechanical, not stylistic. A supervised dispatch injects a multi-KB
preamble that binds the worker to obligations only a live coordinator can discharge:

- **Completion is not pushed.** `worker_done` persists to the coordinator's inbox; it does
  not wake an idle coordinator. If nobody is waiting, the task never settles, the terminal
  is never released, and the human ends up being the delivery layer.
- **The worker may not ask the human.** The preamble forbids local interactive prompts and
  routes every blocking question through `ask`, which waits on the coordinator. Without one,
  a blocked worker simply stalls.
- **Failures are recorded automatically.** Escalations count against the dispatch; a task
  that accumulates three is circuit-broken and marked failed — even if a human quietly
  handled the work in the tab.

So a Run nobody supervises is worse than no Run at all. If the user has not asked for
supervision, do not create the Task row.

## 2. Launching a supervised worker

`orca-work-placement` Rules 2–4 apply unchanged — **name the tab for its work, launch on
Opus unless a model is named, and settle a complex or subtly-conditioned goal with the user
before dispatch.** The orchestration-specific mechanics on top of them:

- **Placement is still a decision.** Read-only investigation can share the current worktree;
  anything that writes goes to the worktree that owns those files — reuse an existing one
  before creating another, and place by repo, not just by path.
- **Model flags do not combine with terminal reuse.** `--effort` requires `--model`, and
  neither can be passed when attaching to an existing terminal. A model choice therefore
  forecloses reuse: start a fresh agent terminal instead of re-engaging a settled worker.
  A retry inherits nothing either — repeat the placement and model flags explicitly.
- **Verify the launch actually took.** Compare the requested launch against the effective
  one in the receipt; if they differ, say so before sending work into that worker.
- **Rename immediately after the receipt returns, before any wait** — create-time display
  names are rejected on some placements, so renaming after the fact is the one path that
  always works. Rename again when a settled worker is re-engaged for a different task; a
  stale title points the user at the wrong answer.

## 3. What the coordinator owes

- **Write a self-contained spec.** A fresh worker knows nothing this session established:
  restate the confirmed facts, name the exact refs and paths, say which findings are already
  settled, and fence exactly what it may write. Pass on the repo's `AGENTS.md`/`CLAUDE.md`
  constraints that bear on the task (never-push rules, commit conventions, skill routing).
  Retrofitting rules by message does not work — mail lands in an inbox a busy worker may not
  read until it is already done.
- **Keep the completion body thin, and put the substance in a file.** The `worker_done` body
  is the one part that always crosses the coordinator's context — and once it lands there it
  is re-sent on every later turn. Name an absolute artifact path in the brief, outside the
  work tree so it cannot pollute a staged commit, and require a ≤3-line body: what was
  finished, that path, and anything needing a decision now. Never accept findings, diffs,
  test output or logs inline; redirect a tool's stdout to the artifact path too, then read
  back only the entries you need rather than the whole file.
- **Name sibling territory when fanning out**, or two workers converge on the same files and
  the same finding arrives twice under different titles.
- **Wait without blocking the session.** Use rolling waits with an explicit timeout, run long
  ones in the background, and treat a timeout or an empty return as a checkpoint rather than
  a failure — real tasks run 15–60 minutes. Never substitute a foreground sleep or a poll
  loop, and never truncate a wait receipt through `tail`/`head`; the completion body is in
  the JSON you would be cutting off.
- **Account for every settled worker before waiting again**: re-engage it for the next task,
  retain it deliberately, or release it. A renamed tab counts as user-modified, so releasing
  it settles the bookkeeping without closing anything — disposal stays the user's call.
- **Batch independent reads into one turn.** A coordinator's token bill is its average
  context times its turn count, and with no parallel calls the turn count just *is* the tool
  call count. Measured on two real supervision sessions, batching only the adjacent
  read-only calls removed 28-30% of all turns; one run of 18 consecutive single-call reads
  should have been one turn. Split turns only where the next call depends on the last
  result, and never batch state-changing calls — check each write's result before the next.
- **Read as little as the judgment needs, and never on a worker's behalf.** Measured on one supervised batch, the coordinator's three largest context items were all things it read for itself — Jira returns 87.7k chars, vault rule docs 64.6k, design docs 56.7k — 66% of its tool-result cost, against $0.08 for everything exchanged with workers. Four rules follow: keep Jira MCP calls to the minimum — its return cannot be narrowed (`MAX_MCP_OUTPUT_TOKENS` is a global truncation and a `PostToolUse` hook fires too late to rewrite a result), so call count is the only variable you control: one JQL for many tickets, never re-ask for a ticket you already have, and no confirming read after a write; leave ticket transitions, comments and description edits to the worker that owns the ticket; cite design-doc line ranges in the brief instead of reading them yourself; and pull rule documents with `grep -n`/`sed -n` at the moment of judgment, never whole.
- **An idle turn costs full price.** The fixed prefix alone — system prompt, tool definitions, skills, AGENTS.md — bills roughly $0.04 per turn regardless of what the turn does, so a wait split across hundreds of polling turns pays that hundreds of times. This is the cost behind the foreground-wait rule above.
- **Relay corrections immediately.** A worker contradicting the briefing it was given is the
  most valuable thing this arrangement produces; surface it and update the spec of every
  task not yet dispatched.

## 4. Reporting

Lead with what was found or changed, not with orchestration mechanics. One line per worker —
`label · worktree · model · what it was told to do` — and one table of outcomes at the end;
the user does not need dispatch IDs. Name the tab that holds each answer, so a follow-up
goes to the agent that still has the context rather than being re-derived here.
