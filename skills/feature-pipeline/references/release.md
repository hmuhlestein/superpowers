# Release (Stage 11)

Part of superpowers:feature-pipeline.

## 1. Prepare

Dispatch [release-engineer-prompt.md](../release-engineer-prompt.md) with
`[MODE]` = `prepare`, `[DEPLOY_APPROVAL]` = `n/a`, `[OUTPUT_FILE]` =
`[RUN_DIR]/release.md`, over MERGE_BASE..HEAD. It never pushes, merges, or
deploys.

Its blockers come tagged:

- `[fixable]` → a fix wave ([fix-wave.md](fix-wave.md)), then prepare again.
  Two waves at most; a blocker still open after that is treated as
  `[needs human]`.
- `[needs human]` (secret rotation, history rewrite, squash) → interactive:
  ask; unattended: see [unattended.md](unattended.md).

## 2. Feature report

Dispatch [feature-reporter-prompt.md](../feature-reporter-prompt.md) over
MERGE_BASE..HEAD, `[OUTPUT_FILE]` = `[RUN_DIR]/feature_report.html`,
`[SUMMARY_FILE]` = `[RUN_DIR]/summary.md`. Before you dispatch it, make sure
every parked finding and `Ruling:` line is in the ledger — the report is
only as complete as the ledger.

## 3. Commit and update

Commit everything in `[RUN_DIR]` so the tree is clean before integration.
If the base branch has a remote and it has moved on
(`git fetch` then `git merge-base --is-ancestor origin/<base> HEAD` fails),
merge it into the branch and run the full test suite. A merge conflict
stops the run (interactive: ask; unattended: Blocked) — conflict
resolution is a design decision.

## 4. Integrate

- **Interactive:** use superpowers:finishing-a-development-branch. If your
  human partner picks the PR option, use the title and body from
  `release.md`.
- **Unattended:** follow [unattended.md](unattended.md).

## 5. After a PR is opened

A pushed PR is not done: CI runs on the pushed commit, and automated
reviewers often post their findings as plain comments, so a PR with no
"changes requested" can still carry real findings.

1. Wait for checks with a bound: `timeout 1800 gh pr checks <pr> --watch`.
2. On a failing check, get the failing step with
   `gh run view <run-id> --log-failed`, then run that exact command locally.
3. Read review comments: `gh pr view <pr> --comments` and
   `gh api repos/{owner}/{repo}/pulls/<pr>/comments`.
4. Actionable CI failures and review findings → a fix wave
   ([fix-wave.md](fix-wave.md)) with the comments saved to
   `[RUN_DIR]/pr-feedback-r<R>.md` as its report, then a normal push — never
   a force-push. Two rounds at most; list anything left in your final
   message.

## 6. Deploy

Only when `DEPLOY.md` exists (at the repo root, or the main checkout's root
when you work in a linked worktree), only in interactive mode, and only
after integration:

- **Merged locally** → show your human partner the deploy preflight from
  `release.md` and ask whether to deploy.
- **PR** → deploy only after your human partner says the PR has merged and
  asks for it, from an up-to-date checkout of the base branch.
- **Kept as-is** → no deploy.

On an explicit yes, dispatch the release engineer with `[MODE]` = `deploy`,
`[DEPLOY_APPROVAL]` = their reply verbatim plus the target environment,
working in the checkout being deployed, with `[OUTPUT_FILE]` =
`[RUN_DIR]/release.md` in that checkout. Leave the appended results
uncommitted and tell your human partner where they are. No `DEPLOY.md`
means no deploy — never improvise one.
