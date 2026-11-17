---
name: feature-pipeline
description: Use when your human partner asks for the full gated pipeline on a feature, or wants it taken from spec to release with independent requirements, architecture, security, QA, and release gates rather than the standard brainstorm-plan-build flow
---

# Feature Pipeline

Take one feature from an approved spec to a release-ready branch through
independent gates. Producers write the artifacts (requirements,
architecture); fresh reviewers judge them; nothing advances past a gate with
open Critical or Important findings unless you rule on them in the ledger.

**Core principle:** Every artifact is judged by a context that did not write
it, and every gate verdict is backed by evidence in a file.

**This is the heavy path.** For a small change, the standard flow —
superpowers:brainstorming → superpowers:writing-plans →
superpowers:subagent-driven-development — is enough. Use this pipeline when
your human partner asks for it, or when the feature is large or risky enough
that a design mistake or a security gap would cost more than eleven
subagent dispatches.

**Rulings, not stalls.** Same rule as superpowers:subagent-driven-development:
decide conflicts and ambiguities yourself and record each as
`Ruling: <what you decided> — <why> — <what it costs if wrong>` in the ledger.
Stop and ask your human partner only for: an irreversible or destructive
operation; a security-sensitive action; a side effect outside this worktree
(merge, push, publish, deploy); a REDESIGN verdict after code exists; a
Critical finding still open at a gate's loop cap; or a question only they can
answer about the spec itself.

## Setup

1. **Spec.** You need an approved spec. If there is none, use
   superpowers:brainstorming first. When brainstorming hands off to
   writing-plans, come back here instead — requirements and architecture
   come before the plan.
2. **Workspace.** Use superpowers:using-git-worktrees. Never run the pipeline
   on main/master without your human partner's explicit consent.
3. **Run directory.** `docs/superpowers/pipeline/YYYY-MM-DD-<feature>/` —
   called `[RUN_DIR]` below. Every pipeline artifact lives here.
4. **Ledger.** `[RUN_DIR]/progress.md`, first line
   `# Pipeline ledger — spec: <spec path>`. Conversation memory does not
   survive compaction; the ledger does. If it already exists, read it and
   resume after the last stage it records as complete — never re-run a
   completed stage from memory.

Ledger lines, one per event:

- `Stage <N> <name>: dispatched (round <R>)`
- `Stage <N> <name>: <PASS|FIX|REDESIGN> (C<n> I<n> M<n>) — <report path>`
- `Stage <N> <name>: complete` / `complete (<K> parked)`
- `Ruling: ...`

Commit `[RUN_DIR]` each time a stage completes (`docs: <feature> pipeline —
<stage> complete`).

## Stages

| # | Stage | Do this | Output in [RUN_DIR] | Gate |
|---|---|---|---|---|
| 1 | Requirements | dispatch [requirements-analyst-prompt.md](requirements-analyst-prompt.md) | requirements.md | — |
| 2 | Scope gate | dispatch [scope-reviewer-prompt.md](scope-reviewer-prompt.md) | scope.md | FIX → redo 1 |
| 3 | Architecture | dispatch [architect-prompt.md](architect-prompt.md) | architecture.md | — |
| 4 | Design gate | dispatch [design-challenger-prompt.md](design-challenger-prompt.md) | challenge.md | FIX → redo 3 |
| 5 | Plan | superpowers:writing-plans, inline | (docs/superpowers/plans/) | — |
| 6 | Build | superpowers:subagent-driven-development | commits | its own task + final reviews |
| 7 | Reviews | dispatch in parallel: [adversarial-reviewer-prompt.md](adversarial-reviewer-prompt.md), [security-reviewer-prompt.md](security-reviewer-prompt.md), [privacy-reviewer-prompt.md](privacy-reviewer-prompt.md) when it applies | adversarial.md, security.md, privacy.md | FIX → fix wave; REDESIGN → ask |
| 8 | QA | dispatch [qa-validator-prompt.md](qa-validator-prompt.md) | qa.md | FIX → fix wave; REDESIGN → ask |
| 9 | Product validation | dispatch [product-validator-prompt.md](product-validator-prompt.md) | validation.md | FIX → fix wave; REDESIGN → ask |
| 10 | Docs and report | dispatch [doc-reviewer-prompt.md](doc-reviewer-prompt.md), then [feature-reporter-prompt.md](feature-reporter-prompt.md) | docs.md, summary.md, feature_report.html | — (fixes in place) |
| 11 | Release | dispatch [release-engineer-prompt.md](release-engineer-prompt.md) `MODE=prepare`, then superpowers:finishing-a-development-branch, then optional deploy | release.md | — |

Every template's verdict is the last section of its report:
`Verdict: PASS | FIX | REDESIGN` and a `Critical / Important / Minor` count.
PASS means zero Critical and zero Important.

### 1–2. Requirements and scope gate

Dispatch the requirements analyst with the spec. Then dispatch the scope
reviewer on spec + requirements. On FIX, re-dispatch the analyst with
`[PRIOR_FINDINGS_FILE]` = the scope report (see Re-runs), then re-run the
scope gate. Questions the analyst lists for your human partner: answer
them from the spec if you can (ledger a Ruling); ask only if the spec
genuinely cannot decide.

### 3–4. Architecture and design gate

Dispatch the architect with spec + requirements, then the design
challenger on all three. On FIX, re-dispatch the architect with the
challenge report, then re-run the gate. Important findings block here —
fixing a design is cheap now and expensive once code is built on it.

### 5. Plan

Use superpowers:writing-plans. The plan's **Spec** line names the spec; its
**Global Constraints** must name `requirements.md` and `architecture.md` by
path and carry the binding REQ IDs and architecture decisions verbatim, so
every implementer and task reviewer sees them. When writing-plans offers the
execution choice, choose subagent-driven-development (or executing-plans if
no subagent tool is available) — do not stop to ask; your human partner
already chose this pipeline.

### 6. Build

Use superpowers:subagent-driven-development. Its final whole-branch review
uses requesting-code-review's code-reviewer.md: fill
`[ARCHITECTURE_OR_NONE]` with `[RUN_DIR]/architecture.md` so the review
checks architecture conformance. Treat an `Architecture: Violated — needs
redesign` assessment as a REDESIGN verdict.

When subagent-driven-development reaches its finish step, **do not** use
finishing-a-development-branch yet. Copy its "Rulings I made" list into the
pipeline ledger and continue at stage 7.

### 7. Reviews

Decide whether the privacy reviewer applies, starting from requirements.md's
Personal Data section: it does when the spec or requirements involve
personal data, health information, financial data,
credentials tied to a person, or anything a regulation governs. Ledger the
decision as a Ruling either way.

Build one review package over the whole branch with
`../subagent-driven-development/scripts/review-package PLAN_FILE MERGE_BASE HEAD`
(by its absolute path, from the working tree; MERGE_BASE =
`git merge-base <base-branch> HEAD`). Dispatch every applicable reviewer at
once with that same package — they are read-only and independent.

The security reviewer runs the AWS Automated Security Helper (ASH) through
`uvx`, which fetches and caches it on first use. If `uv` isn't installed,
the review continues without the scan and asks your human partner to
install it; nothing is installed without their consent.

- All PASS → stage 8.
- Any REDESIGN → stop and ask your human partner: return to stage 3 with the
  findings, or park them and continue.
- Otherwise → one fix wave covering every FIX report, then re-run only the
  reviewers that returned FIX.

### 8–9. QA and product validation

Dispatch the QA validator, then the product validator. Each one's FIX goes
through a fix wave and a re-run of that same stage. A product-validation fix
wave also re-runs QA before product validation runs again — a product fix
can break what QA proved. If the product validator raises a question about
the spec itself, ask your human partner; that is not a code fix.

### 10. Docs and feature report

Dispatch the doc reviewer. It fixes documentation in place and commits.
Ledger anything it reports as unresolved.

Then dispatch the feature reporter over the whole branch (merge base to
HEAD). It writes `summary.md` and `feature_report.html` — a standalone,
light/dark-aware HTML record of what was built, why, and how, with every
accepted risk from the ledger called out near the top. Make sure every
parked finding and `Ruling:` line is in the ledger before you dispatch it;
the report is only as complete as the ledger.

### 11. Release

1. Dispatch the release engineer with `MODE=prepare`. It checks commit
   hygiene, updates the changelog/version if the repo keeps them, checks CI,
   drafts the PR title and body, and — if the repo has a `DEPLOY.md` —
   writes a deploy preflight. It never pushes, merges, or deploys. Blockers
   it reports go through a fix wave, then prepare runs again.
2. Use superpowers:finishing-a-development-branch. If your human partner
   picks the PR option, use the PR title and body from `release.md`.
3. **Deploy only if `DEPLOY.md` exists at the repo root.** After the work is
   integrated, show your human partner the deploy preflight from
   `release.md` and ask whether to deploy. Only on an explicit yes, dispatch
   the release engineer with `MODE=deploy` and `[DEPLOY_APPROVAL]` set to
   their reply quoted verbatim plus the target environment. No `DEPLOY.md`
   means no deploy step — do not improvise one.

## The fix wave

Stages 7–9 never fix code themselves. When a gate returns FIX:

1. Record `FIX_BASE=$(git rev-parse HEAD)`.
2. Dispatch ONE fix subagent using
   [implementer-prompt.md](../subagent-driven-development/implementer-prompt.md).
   Its brief lists the report path(s) and the Critical and Important finding
   IDs to fix; it reads the reports itself. Minor findings are fixed only
   if trivial and in a file already being touched. One fixer for the whole
   wave — per-finding fixers each rebuild context and re-run suites.
3. Re-run the gate(s) that returned FIX with a review package over
   `FIX_BASE..HEAD` and `[PRIOR_FINDINGS_FILE]` set (see Re-runs). Re-runs
   verify the fixes and check them for regressions.

**Loop cap: three rounds per gate.** At the cap, park each remaining
Important finding with a Ruling and continue. If a Critical finding is still
open at the cap, stop and ask your human partner. Past the cap, more rounds
don't converge; the problem is structural.

## Re-runs

Before re-dispatching any stage, rename its previous output to
`<name>-r<N>.md` (e.g. `scope-r1.md`) and pass that path as
`[PRIOR_FINDINGS_FILE]`. Producers (requirements analyst, architect) get the
gate's report so they can answer each finding; reviewers get their own prior
report so they can mark each finding FIXED or NOT FIXED. First runs pass
`none`.

## Dispatching

- **Hand over files, not content.** Fill placeholders with paths. Never
  paste a spec, report, or diff into a dispatch prompt — it stays in your
  context for the rest of the run.
- **Read the Report Back, not the report.** Each template returns only its
  verdict, counts, path, and one line per Critical or Important finding. The
  fixer and the next reviewer read the full file; you don't need to.
- **Subagents don't dispatch subagents.** Every template says so; a reviewer
  that spawns a helper is a defect to note, not extra rigor.
- **Always set the model explicitly.** An omitted model inherits your
  session's, usually the most expensive one.

### Placeholders

Each template lists its own. The shared ones: `[SPEC_FILE]`,
`[REQUIREMENTS_FILE]`, `[ARCHITECTURE_FILE]`, `[RUN_DIR]`, `[OUTPUT_FILE]`
(the stage's output in the table above), `[PRIOR_FINDINGS_FILE]` (see
Re-runs), `[FEATURE]` (the run's short name), and for code stages
`[BASE_SHA]`, `[HEAD_SHA]`, `[DIFF_FILE]` (the review package). The security
reviewer also takes `[BASE_BRANCH]` (bare local name, e.g. `main`); the
feature reporter takes `[SUMMARY_FILE]`; the release engineer takes `[MODE]` and `[DEPLOY_APPROVAL]` (`n/a` in prepare
mode).

### Model selection

| Role | Model |
|---|---|
| Architect, design challenger, adversarial reviewer | most capable |
| Security and privacy reviewers | most capable for auth, crypto, data handling, or a large diff; otherwise standard |
| Requirements analyst, scope reviewer, QA validator, product validator | standard |
| Fix subagent | standard; one tier up from the previous fixer if a gate is on its third round |
| Doc reviewer, feature reporter, release engineer | standard (cheap is fine for a small diff) |

## Finish

Your final message to your human partner contains:

- One line per stage: final verdict and round count.
- **Rulings I made** — every `Ruling:` line in the ledger, in order, each
  with what it costs if wrong. The list is exhaustive.
- Parked findings, with their report paths.
- Open questions for your human partner.
- The path to `feature_report.html` — the record they will open first.
- The run directory path.

The run directory is kept and committed — it is the design record of the
feature.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The spec is clear, skip requirements" | Requirements turn the spec into checkable acceptance criteria. QA and product validation have nothing to verify against without them. |
| "Design gate found only Important issues, start building" | Important blocks at the design gate. A design gap found after code exists costs a rebuild. |
| "Security found one small thing, I'll patch it myself" | You coordinate. Fixes go through one fix subagent and a re-review, or they land unreviewed. |
| "Privacy doesn't apply" without checking | Decide from the spec and requirements, and ledger the Ruling. Personal data hides in logs, errors, and analytics. |
| "QA passed, product validation is redundant" | QA proves the code meets the criteria. Product validation proves the criteria deliver what the spec meant. |
| "There's a deploy script, I'll run it" | Deploy follows `DEPLOY.md` only, and only after an explicit yes. |
| "Re-run the whole pipeline after a fix" | Re-run only the gate that failed, over the fix range. |
