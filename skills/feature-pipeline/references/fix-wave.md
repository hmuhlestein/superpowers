# The Fix Wave

Part of superpowers:feature-pipeline. Read this before every fix wave.

A fix wave turns gate findings into commits. Reviewers never fix; one fix
subagent does, then the right reviewers check its work.

## 1. Open the wave

Record `FIX_BASE=$(git rev-parse HEAD)` and ledger
`Fix wave <stage> round <R>: FIX_BASE=<sha> — reports <paths>`.

## 2. Write the brief

Write `[RUN_DIR]/fix-<stage>-r<R>-brief.md` containing:

- The report path(s) and the Critical and Important finding IDs to fix —
  every one of them. The fixer reads the reports itself.
- Minor findings: fix only if trivial and in a file already being touched.
- **Regression test per fix.** For every Critical or Important finding about
  behavior, a test that fails at `FIX_BASE` and passes at HEAD, with both
  runs' output in the report. Prove the failure by running the new test
  against `FIX_BASE` in a temporary worktree
  (`git worktree add <tmp> <FIX_BASE>`, copy the test in, run it, remove the
  worktree) — or with `git stash push -u`, never plain `git stash`, which
  leaves new untracked test files in place and makes the check meaningless.
- **Do not weaken existing tests.** Tests that existed at the merge base
  change only when a finding requires it, and the report says which and why.
- **Disputes.** A finding the fixer believes is wrong is not fixed; the
  report says why, with evidence.
- The plan's Global Constraints, verbatim.

## 3. Dispatch one fixer

Use [implementer-prompt.md](../../subagent-driven-development/implementer-prompt.md):

- `Task N: [task name]` → `Fix wave <stage> round <R>`
- `[BRIEF_FILE]` → the brief above
- `[directory]` → the worktree root
- `[REPORT_FILE]` → `[RUN_DIR]/fix-<stage>-r<R>-report.md`
- Context → one line on where the pipeline is, plus the pipeline-records line
  from SKILL.md's Dispatching section
- Model → per SKILL.md's model table

One fixer for the whole wave. Per-finding fixers each rebuild context and
re-run suites.

**Report status:** DONE → continue. DONE_WITH_CONCERNS, NEEDS_CONTEXT, or
BLOCKED → follow
[handling-reports.md](../../subagent-driven-development/references/handling-reports.md).

## 4. If nothing was committed

The fixer disputed every finding. Rule on each dispute yourself: accept it
(ledger a Ruling; the finding is closed) or re-dispatch once with your
ruling attached. With no new commits there is nothing to re-review.

## 5. Re-review

Build a package over `FIX_BASE..HEAD` with
`../../subagent-driven-development/scripts/review-package`. Then re-run:

- **The gate(s) that returned FIX**, each with its prior report renamed and
  passed as `[PRIOR_FINDINGS_FILE]` (see SKILL.md's Re-runs). QA re-runs are
  full passes over the whole branch with `[FIX_RANGE]` set.
- **Every applicable stage-7 reviewer** (adversarial, security, privacy if
  it applies) **when the fix changed non-test source**
  (`git diff --name-only FIX_BASE..HEAD`), over the fix package, each with
  its current report as `[PRIOR_FINDINGS_FILE]`. A QA or product fix that
  registers a route or logs a new field is unreviewed attack surface
  otherwise.
- **code-reviewer.md** with `[ARCHITECTURE_OR_NONE]` set, when the wave
  included findings carried from review.md.
- For a stage-9 wave, QA too, before product validation.

New Critical or Important findings from any of these join the next wave of
the same stage.

Ledger `Fix wave <stage> round <R>: done HEAD=<sha>` and each re-run's
verdict line.

## 6. Loop cap

**Three waves per stage** (stage 7 counts as one stage; release prepare
gets two). At the cap:

- Important findings → park each with a Ruling and continue.
- A Critical finding still open → stop. Interactive: ask your human partner.
  Unattended: Blocked.

Past the cap, more rounds don't converge; the problem is structural.
