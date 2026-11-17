---
name: feature-pipeline
description: Use when your human partner asks for the full gated pipeline on a feature (interactive or unattended), or wants it taken from spec to release with independent requirements, architecture, security, QA, and release gates rather than the standard brainstorm-plan-build flow
---

# Feature Pipeline

Take one feature from an approved spec to a release-ready branch through
independent gates. Producers write the artifacts (requirements,
architecture); fresh reviewers judge them; nothing advances past a gate with
open Critical or Important findings unless you rule on them in the ledger.

**Core principle:** Every artifact is judged by a context that did not write
it, and every gate verdict is backed by evidence in a file you have checked
exists.

**This is the heavy path.** For a small change, the standard flow —
superpowers:brainstorming → superpowers:writing-plans →
superpowers:subagent-driven-development — is enough. Use this pipeline when
your human partner asks for it, or when the feature is large or risky enough
that a design mistake or a security gap would cost more than a dozen
subagent dispatches.

**Requires a subagent tool.** Stages 1–4 and 7–11 are all fresh-context
dispatches. Without a subagent tool, say so and stop.

**Two modes.** *Interactive* (default): you stop and ask at the points named
below. *Unattended*: your human partner asked for an unattended run, or you
were launched by [scripts/run-unattended](scripts/run-unattended). In
unattended mode, read [references/unattended.md](references/unattended.md)
before Setup — it replaces every "ask your human partner" in this skill.

**Spec Kit projects.** When the repo has `.specify/` and the spec is a Spec
Kit feature (`specs/<NNN-name>/`), read
[references/speckit.md](references/speckit.md) before Setup. Spec Kit's
spec.md, plan.md, and tasks.md replace stages 1–5's artifacts; the pipeline
runs the Spec Kit commands still missing, gates them, and builds from
tasks.md. Run files live in the feature directory.

**Rulings, not stalls.** Same rule as superpowers:subagent-driven-development:
decide conflicts and ambiguities yourself and record each as
`Ruling: <what you decided> — <why> — <what it costs if wrong>` in the ledger.
In interactive mode, stop and ask only for: an irreversible or destructive
operation; a security-sensitive action; a side effect outside this worktree
(merge, push, publish, deploy); a REDESIGN verdict; a gate at its loop cap
with a finding it may not park; a malformed report twice in a row; or a
question only your human partner can answer about the spec itself.

## Setup

1. **Spec.** You need an approved spec. If there is none, use
   superpowers:brainstorming first, then come back here — brainstorming
   knows to return to this skill instead of going to writing-plans.
2. **Workspace.** Use superpowers:using-git-worktrees. Never run the
   pipeline on main/master. A new worktree sees only committed files: if
   the spec — or the Spec Kit feature directory, or `.specify/` — has
   uncommitted changes in the checkout you started from, copy them into the
   worktree and commit them there as the branch's first commit (never
   `.specify/feature.json`). Never
   commit on the branch you started from.
3. **Run directory.** Look for an existing run first:
   `grep -l "spec: <spec path>" docs/superpowers/pipeline/*/progress.md`.
   If one matches, it is `[RUN_DIR]` — resume. Otherwise create
   `docs/superpowers/pipeline/YYYY-MM-DD-<feature>/` (today's date, fixed
   for the life of the run).
4. **Ledger.** `[RUN_DIR]/progress.md`, first line
   `# Pipeline ledger — spec: <spec path>`. Conversation memory does not
   survive compaction; the ledger does. On resume, read it and continue
   after the last `complete` line that follows the latest `Rewind` line —
   never from memory. If `[RUN_DIR]/human-input.md` exists, read it first:
   it holds your human partner's answers to an earlier stop.

Ledger lines, one per event:

- `Stage <N> <name>: dispatched (round <R>)`
- `Stage <N> <name> [<role>]: <PASS|FIX|REDESIGN> (C<n> I<n> M<n>) — <report path>`
  (one line per reviewer in stage 7)
- `Fix wave <stage> round <R>: FIX_BASE=<sha> — reports <paths>` and
  `Fix wave <stage> round <R>: done HEAD=<sha>`
- `Stage <N> <name>: complete` / `complete (<K> parked)`
- `Rewind to stage 3: <finding IDs> — <reason>`
- `Ruling: ...`, `Question: ...`, `Answer: ...`, `Blocked: ...`

Commit `[RUN_DIR]` each time a stage completes (`docs: <feature> pipeline —
<stage> complete`).

## Stages

| # | Stage | Do this | Output in [RUN_DIR] | Gate |
|---|---|---|---|---|
| 1 | Requirements | dispatch [requirements-analyst-prompt.md](requirements-analyst-prompt.md) | requirements.md | — |
| 2 | Scope gate | dispatch [scope-reviewer-prompt.md](scope-reviewer-prompt.md) | scope.md | FIX → redo 1 |
| 3 | Architecture | dispatch [architect-prompt.md](architect-prompt.md) | architecture.md | — |
| 4 | Design gate | dispatch [design-challenger-prompt.md](design-challenger-prompt.md) | challenge.md | FIX → redo 3 |
| 5 | Plan | superpowers:writing-plans, inline | (docs/superpowers/plans/) | REQ coverage check |
| 6 | Build | superpowers:subagent-driven-development | commits, review.md | its own reviews |
| 7 | Reviews | dispatch in parallel: [adversarial-reviewer-prompt.md](adversarial-reviewer-prompt.md), [security-reviewer-prompt.md](security-reviewer-prompt.md), [privacy-reviewer-prompt.md](privacy-reviewer-prompt.md) when it applies | adversarial.md, security.md, privacy.md | fix wave / redesign |
| 8 | QA | dispatch [qa-validator-prompt.md](qa-validator-prompt.md) | qa.md | fix wave / redesign |
| 9 | Product validation | dispatch [product-validator-prompt.md](product-validator-prompt.md) | validation.md | fix wave / redesign |
| 10 | Docs | dispatch [doc-reviewer-prompt.md](doc-reviewer-prompt.md) | docs.md | — (fixes in place) |
| 11 | Release | [references/release.md](references/release.md): [release-engineer-prompt.md](release-engineer-prompt.md), [feature-reporter-prompt.md](feature-reporter-prompt.md), integration, post-PR follow-up, optional deploy | release.md, walkthrough.md, feature_report.html | release blockers |

Every gate report ends with `Verdict: PASS | FIX | REDESIGN` and a
`Critical / Important / Minor` count. PASS means zero Critical and zero
Important.

### 1–2. Requirements and scope gate

Dispatch the requirements analyst with the spec. Then dispatch the scope
reviewer on spec + requirements. On FIX, re-dispatch the analyst with the
scope report (see Re-runs), then re-run the scope gate.

### 3–4. Architecture and design gate

Dispatch the architect with spec + requirements, then the design
challenger on all three. On FIX, re-dispatch the architect with the
challenge report, then re-run the gate. Important findings block here —
fixing a design is cheap now and expensive once code is built on it.

**Pre-build loop cap: three rounds each for stages 2 and 4.** At the cap,
nothing is parked: building on unconfirmed requirements or design is the
failure these gates exist to stop. Interactive: stop and ask. Unattended:
Blocked.

### 5. Plan

Use superpowers:writing-plans. The plan's **Spec** line names the spec. Its
**Global Constraints** must carry, verbatim: the paths of `requirements.md`
and `architecture.md`; the binding REQ IDs; architecture.md's Security and
Privacy decisions (including the log-redaction approach) and its
configuration decisions; and the changed-line coverage floor (80% unless the
spec sets one) — only when the project has a coverage tool or its
constraints allow adding one. Otherwise write "Coverage: not measured — no
coverage tool permitted" and rely on the acceptance-criteria test mapping
below; never have implementers build a coverage tool. Implementers see only
their brief plus these constraints.

Then check coverage yourself: every `REQ-NN.AC-M` maps to a task whose
tests exercise it. Add a task for each gap. Commit the plan
(`docs: <feature> implementation plan`).

writing-plans offers an execution choice and a plan review. Choose
subagent-driven; do not wait for a review — the design gate already ran.
Interactive: link the plan for your human partner and keep going unless
they object.

### 6. Build

Use superpowers:subagent-driven-development. For its final whole-branch
review (code-reviewer.md), fill `[ARCHITECTURE_OR_NONE]` with
`[RUN_DIR]/architecture.md`, and save the reviewer's full report to
`[RUN_DIR]/review.md` — it is the architecture-conformance record. Ledger
its `Architecture:` assessment line. Before subagent-driven-development
deletes its workspace, copy its ledger and every task and fix report into
`[RUN_DIR]/build-reports/` — they hold the test-first (red/green) evidence
that QA and the feature report cite, and git history alone doesn't show it.

When subagent-driven-development reaches its finish step, **do not** use
finishing-a-development-branch. Copy its "Rulings I made" list into the
pipeline ledger, then:

- `Architecture: Violated — needs redesign` → [references/redesign.md](references/redesign.md).
- Critical or Important findings still open in review.md after SDD's own
  fix wave → they join stage 7's first fix wave; code-reviewer.md re-reviews
  them there with `[PRIOR_FINDINGS_FILE]` = review.md.
- Otherwise → stage 7.

### 7. Reviews

**Privacy applies** when requirements.md's Personal Data section names any
field, or when the diff stores, logs, transmits, or displays personal
data — emails, names, IPs, device or account IDs, location, health,
financial data, person-tied credentials. Check the diff, not just the
section. Ledger the decision as a Ruling either way.

Build one review package over the whole branch with
`../subagent-driven-development/scripts/review-package PLAN_FILE MERGE_BASE HEAD`
(by its absolute path, from the worktree; MERGE_BASE =
`git merge-base <base-branch> HEAD`). Dispatch every applicable reviewer at
once with that package. The security reviewer takes `[BASE_BRANCH]`; it
runs the AWS Automated Security Helper through `uvx`, which fetches ASH on
first use. If `uv` itself is missing, the scan is skipped and reported; uv
is never installed for you.

- All PASS and nothing carried from review.md → stage 8.
- Any REDESIGN → [references/redesign.md](references/redesign.md).
- Otherwise → [the fix wave](references/fix-wave.md).

### 8–9. QA and product validation

QA always gets the whole branch: `[BASE_SHA]` = MERGE_BASE on every run,
with `[FIX_RANGE]` naming the fix wave on re-runs. Then product validation.
FIX → fix wave; REDESIGN → redesign. A product-validation fix wave re-runs
QA before product validation runs again; that QA run counts against
product validation's cap, not QA's.

### 10. Docs

Dispatch the doc reviewer. It fixes documentation in place and commits. Its
Report Back lists unresolved items; ledger each one.

### 11. Release

Read [references/release.md](references/release.md) and follow it: release
prepare, the feature report, integration, post-PR follow-up, and deploy.

## Checking a report before it counts

After every dispatch, before you ledger anything:

1. `[OUTPUT_FILE]` exists and was written by this dispatch (not a stale
   file from an earlier round).
2. A gate report ends with the Verdict block, its verdict matches the
   Report Back, and its counts match the findings it lists. If they
   disagree, the higher severity wins.
3. The security report has an `## Automated Scan Results` section.
4. Before the next dispatch: every input path you are about to hand over
   exists.

A missing or malformed report is a failed dispatch: re-dispatch once with
what was wrong. A second failure stops the run (interactive: ask;
unattended: Blocked).

## Open questions

Report Backs list open questions. For each: answer it from the spec,
requirements, or codebase if you can (ledger a Ruling); otherwise ask your
human partner (unattended: see references/unattended.md). Then write every
answer where later stages will read it: append it under
`## Resolved Questions` in requirements.md or architecture.md (that section
is the only edit you make to those files) and commit. In Spec Kit mode,
spec.md's `## Clarifications` and plan.md take them instead — see
references/speckit.md. A ruling that lives only in the ledger is invisible
to the next subagent.

## The fix wave

Stages 7–9 and release never fix code themselves. Read
[references/fix-wave.md](references/fix-wave.md) before every fix wave, even
if you read it earlier in this session. In short: one fixer for all
findings; each Critical and Important fix ships a regression test proven to
fail first; the re-review covers the gate that asked plus every stage-7
reviewer when non-test source changed; three waves per stage.

## Re-runs

Each re-run supersedes one file. Before re-dispatching a **producer**
(analyst, architect), rename its output (`requirements.md` →
`requirements-r<N>.md`) and pass the gate report (`scope.md`) as
`[PRIOR_FINDINGS_FILE]`. Before re-running a **gate**, rename its report
(`scope.md` → `scope-r<N>.md`) and pass that renamed path as
`[PRIOR_FINDINGS_FILE]`. N is the round being superseded: 1, then 2. First
runs pass `none`.

## Dispatching

- **Hand over files, not content.** Fill placeholders with paths. Never
  paste a spec, report, or diff into a dispatch prompt.
- **Read the Report Back, then check the file** (see above). The fixer and
  the next reviewer read the full report; you read only what you must
  verify.
- **Pipeline records are not feature changes.** Add this line to every
  dispatch from stage 6 on: "Paths under docs/superpowers/ are pipeline
  records, not feature changes — exclude them from scope, hygiene,
  coverage, and scan judgments."
- **Subagents don't dispatch subagents.**
- **Always set the model explicitly.**

### Placeholders

Each template lists its own. The shared ones: `[SPEC_FILE]`,
`[REQUIREMENTS_FILE]`, `[ARCHITECTURE_FILE]`, `[RUN_DIR]`, `[OUTPUT_FILE]`
(the stage's output in the table above), `[PRIOR_FINDINGS_FILE]` (see
Re-runs), `[FEATURE]` (the run's short name), and for code stages
`[BASE_SHA]`, `[HEAD_SHA]`, `[DIFF_FILE]` (the review package). Also:
`[BASE_BRANCH]` (security; bare local name, e.g. `main`), `[FIX_RANGE]`
(QA), `[SUMMARY_FILE]` (feature reporter), `[MODE]` and `[DEPLOY_APPROVAL]`
(release engineer; `n/a` in prepare mode).

### Model selection

| Role | Model |
|---|---|
| Architect, design challenger, adversarial reviewer | most capable |
| Security and privacy reviewers | most capable for auth, crypto, data handling, or a large diff; otherwise standard |
| Requirements analyst, scope reviewer, QA validator, product validator | standard |
| Fix subagent | standard; one tier up on a stage's third wave |
| Doc reviewer, feature reporter, release engineer | standard (cheap is fine for a small diff) |

## Finish

Your final message contains:

- One line per stage: final verdict and round count.
- **Rulings I made** — every `Ruling:` line in the ledger, in order, each
  with what it costs if wrong. The list is exhaustive.
- Parked findings, with their report paths.
- Open questions for your human partner.
- The path to `feature_report.html` — the record they will open first.
- The run directory path.

Unattended runs end with the status lines in references/unattended.md.

The run directory is kept and committed — it is the design record of the
feature.

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The spec is clear, skip requirements" | Requirements turn the spec into checkable acceptance criteria. QA and product validation have nothing to verify against without them. |
| "Design gate hit its cap, park it and build" | Pre-build gates never park. A design gap found after code exists costs a rebuild. |
| "The Report Back says PASS, ledger it" | Check the file. A PASS with no report behind it is a hallucinated gate. |
| "Security found one small thing, I'll patch it myself" | You coordinate. Fixes go through the fix wave and a re-review, or they land unreviewed. |
| "QA's fix only touched one function, skip security" | Any non-test fix gets the stage-7 reviewers over the fix range. A fix that registers a route or logs a field is new attack surface. |
| "Privacy doesn't apply" without checking the diff | Personal data hides in logs, errors, and analytics. Check the code, ledger the Ruling. |
| "QA passed, product validation is redundant" | QA proves the code meets the criteria. Product validation proves the criteria deliver what the spec meant. |
| "There's a deploy script, I'll run it" | Deploy follows `DEPLOY.md` only, and only after an explicit yes. Unattended runs never deploy. |
| "Re-run the whole pipeline after a fix" | Re-run the gates the fix wave names, over the fix range. Only a rewind re-runs stages wholesale. |
