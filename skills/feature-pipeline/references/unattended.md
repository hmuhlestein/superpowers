# Unattended Mode

Part of superpowers:feature-pipeline. Read this before Setup when the run is
unattended. It overrides every "ask your human partner" in the pipeline and
the skills it uses: nobody is there to answer.

## Inputs

- **Spec** — a path to an existing spec, or a Spec Kit feature directory
  (see [speckit.md](speckit.md)'s Unattended section). No brainstorming:
  without a spec, end with `PIPELINE STATUS: FAILED — no spec`.
- **Integration** — `keep` (default) or `pr`. Never a local merge.
- **Deploy** — never. The report notes that deploy awaits a human.

## Overrides

- **Worktree.** The unattended launch is consent. Use
  superpowers:using-git-worktrees without asking; if you are already in a
  linked worktree, use it. Never work on main/master.
- **Plan review and execution choice.** Waived; use subagent-driven.
- **Finishing.** Do not use finishing-a-development-branch's menu (see
  Integration below).

## Every decision point

| Situation | Unattended action |
|---|---|
| Open question at any stage | Rule from the spec, requirements, and codebase, taking the most conservative reading (least scope, least data exposure, nothing irreversible). Ledger `Ruling: [unattended] ...` and write the answer under `## Resolved Questions`. |
| A question about the spec itself | Same, and flag it at the top of the final message. |
| Stage 2 or 4 (or the Spec Kit analyze gate) at its cap with open Critical/Important | Blocked |
| Stages 7–9 or release at the cap, Important | Park with a Ruling |
| Any stage at the cap, Critical | Blocked |
| Malformed report twice | Blocked |
| REDESIGN | Rewind once ([redesign.md](redesign.md)); a second is Blocked |
| Release blocker `[needs human]` | Blocked — unless integration is `keep` and the blocker only concerns publishing (e.g. a squash choice): park it |
| Merge conflict with the base branch | Blocked |
| Destructive, irreversible, or security-sensitive action | Never take it. Blocked if the pipeline cannot continue without it. |
| `uv` missing for the ASH scan | Continue; list it as an accepted risk |

## Integration

- **`keep`:** leave the branch and worktree in place.
- **`pr`:** run the full test suite (it must be green), `git push -u origin
  <branch>`, then `gh pr create` with the title and body from
  `release.md`. Then follow the post-PR steps in [release.md](release.md).
  Never merge, never push to the base branch, never force-push.

## Blocked

When the run is Blocked:

1. Ledger `Blocked: <stage> — <what is needed>` and commit `[RUN_DIR]`.
2. Write the questions or decisions needed into `[RUN_DIR]/blocked.md`.
3. End the run.

Your human partner answers in `[RUN_DIR]/human-input.md` and relaunches
with the same spec. The pipeline resumes from the ledger and reads that file
first.

## Final message

Everything SKILL.md's Finish section lists, then exactly these lines last:

```
PIPELINE REPORT: <path to feature_report.html, or none>
PIPELINE STATUS: <COMPLETE | BLOCKED | FAILED> — <one-line reason>
```

COMPLETE means stage 11 finished. The launcher reads the status line to set
its exit code.
