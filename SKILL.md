---
name: guy-loop
description: A spec-to-build loop. Writes the spec with the user (brainstorming), takes it through up to 2 adversarial-review rounds (fresh Opus agent each round), writes the implementation plan, runs 1 adversarial-review on the plan (fresh Opus agent), then hands the build to Codex via codex-use. Use when the user says "/guy-loop", "guy-loop", "let's do our loop", "run the loop on this spec/idea", or "spec it, review it, plan it, then codex it". If a spec already exists, starts at the review step.
---

# guy-loop

Idea → spec → reviewed spec → plan → reviewed plan → Codex build. The main session is
the author: it fixes findings between rounds. Reviewers are always fresh Opus
agents so no reviewer has seen a previous round's reasoning.

Requires these skills: [superpowers](https://github.com/obra/superpowers)
(`brainstorming`, `writing-plans`),
[adversarial-review](https://github.com/guydotan55/adversarial-review) and
[codex-use](https://github.com/guydotan55/codex-use).

```
idea ──► write spec ──► user approves draft
                                 │
                                 ▼
                             review 1 ──► SHIP IT? ──yes──────────────┐
                                 │ no: fix                            │
                                 ▼                                    ▼
                             review 2 ──► fix / fold / escalate ──► write plan ──► plan review ──► fix ──► /codex-use
```

## Step 0. Entry

1. **Spec or idea?** If the user names a spec on disk (or one was just written
   this session), skip Step 1. Otherwise start at Step 1.
2. **Consent is covered.** If your setup asks before each review
   round, invoking this skill is that consent: it pre-approves exactly
   3 Opus review passes (2 spec + 1 plan). Say it in one line, e.g.
   "Running guy-loop: up to 2 Opus reviews on the spec, 1 on the plan, then Codex."
   That line is also the required Opus reason.
3. **Collect the review inputs once** (reused every round): the user's locked
   decisions verbatim, the out-of-scope list, and facts you verified against
   the real system (claim + file:line or command + output).
4. **Worktree.** If the spec or plan lives in a git repo, use your
   worktree skill if you have one, otherwise
   `git worktree add ../<repo>-<task> -b <branch>`, before writing to it. Codex must run inside that worktree.

## Step 1. Write the spec

Invoke `superpowers:brainstorming` and follow it: questions with the user, then the
written design document (in a repo: `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`
unless the project has its own specs folder).

Two adjustments inside guy-loop:
- **Skip brainstorming's builder-check subagent.** The Step 2 reviews replace
  it and are already part of the dispatch budget.
- **Its hard gate is the draft approval.** The user approves the draft spec here,
  then the rest of the loop runs without stopping until a Blocker, a Major
  that's the user's call, or the Codex launch dialog. Do NOT follow brainstorming's
  "next invoke writing-plans" handoff; go to Step 2.

## Step 2. Spec review — up to 2 rounds

For each round, dispatch a **new** agent (never SendMessage a previous
reviewer — fresh context is the point; do not reuse a reviewer to save a spawn):

```
Agent(model: "opus", subagent_type: "general-purpose", name: "spec-review-<n>",
      prompt: <reviewer brief below>)
```

After each round, triage per the adversarial-review Loop: fix Blockers and
Majors, fold in Minors, record deferred Minors in the spec.

- Round 1 verdict **SHIP IT** → skip round 2, go to Step 3.
- Round 1 **NOT READY** → fix, then round 2. Its brief carries round 1's
  findings, what you changed and where, and every deferral or dispute.
- After round 2, **stop reviewing**. Fold in Minors. A Major you'd rather not
  fix is the user's call: list it, don't defer it yourself. A remaining
  **Blocker** stops the loop: show it to the user and wait.

## Step 3. Write the implementation plan

Invoke `superpowers:writing-plans` with the reviewed spec. Save the plan next
to the spec (`<spec-name>-plan.md`) unless the project already has a plans
folder. The plan should list slices Codex can build and check one at a time.

## Step 4. Plan review — 1 round

One new Opus agent (`name: "plan-review"`), same brief. Hand it **both** the
plan and the reviewed spec, and tell it to also check that the plan covers the
whole spec. Fix Blockers and Majors, fold in Minors. No second round. A
Blocker you can't fix with confidence → show it to the user and wait.

## Step 5. Build with Codex

Invoke `codex-use` in build mode, pointing it at the plan and spec inside the
worktree. codex-use asks the user for model and effort; that dialog is the
checkpoint. If Majors are still waiting on the user, ask about them in the same
message before launching. If preflight shows Codex is logged out or out of
credit, stop and tell the user. Don't switch to a Claude builder on your own.

## Step 6. Report

Keep it short: spec path, verdicts per round, what changed, deferrals and open
user's-call items, the plan path, and the Codex run id/status. Offer
`/code-review` on the result once Codex finishes. Don't start it unasked.

## Reviewer brief (fill every field)

```
GOAL: Adversarial review of <spec|plan> at <absolute path>. Round <n> of guy-loop.
CONTRACT: Read ~/.claude/skills/adversarial-review/SKILL.md and follow it
  exactly. You are the dispatched reviewer: do not dispatch further reviewers.
  Do not recommend or launch a panel.
SEAT MODEL: dispatched on opus; main session is <opus|fable>.
ARTIFACT: <path(s)>. Read the files themselves, not a summary.
  (Plan review: also <spec path>; check the plan covers the spec.)
LOCKED DECISIONS (verbatim): ...
OUT OF SCOPE: ...
ALREADY VERIFIED (bare evidence; re-verify, don't re-flag): ...
PRIOR ROUNDS (round 2 only): findings, fixes with locations, deferrals, disputes.
DON'T: edit any file. Don't consult an advisor model. If a command fails twice
  with the same error, stop and report.
REPORT: the full adversarial-review output (Findings, Hard Questions, Seeds
  checked, Carried forward, Verdict, Seat model). SEND it to the main session
  with SendMessage before finishing. The task is not done until it's sent.
```

Never put your reasons for why the artifact is good in the brief. Evidence only.

## Rules

- Hard cap: 3 review dispatches total (2 spec + 1 plan). The user's explicit
  "do another round" is the only way to add one.
- Every reviewer is `model: "opus"`, passed explicitly, and freshly spawned.
- Fixing findings is main-session work if it's ≤~5 edits to files already in
  context. Bigger rewrites go to a worker agent (e.g. Sonnet) with a full brief: goal, files, constraints, how to report.
- Never relay a reviewer's verdict without reading its findings yourself.
- Watch for silence: if a reviewer goes idle with no SendMessage, read its
  output and ask it once for the report.

## Red flags

| Thought | Reality |
|---|---|
| "Round 2 is the same reviewer, I'll just SendMessage it" | Fresh agent every round. The skill's rule. |
| "Round 2 still found Majors, one more round" | Cap is 2. Majors go to the user as their call. |
| "Plan is simple, skip its review" | The plan review is mandatory. Only spec round 2 is conditional. |
| "Codex is out, I'll have Sonnet build it" | Stop and ask the user. |
| "I'll self-review the plan, I wrote it" | Author review never counts. Dispatch. |
| "The user approved the draft, the reviews can wait" | Approval starts the reviews, it doesn't replace them. |
| "Brainstorming says invoke writing-plans now" | Not inside guy-loop. Spec reviews come first. |
