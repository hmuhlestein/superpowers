# Redesign

Part of superpowers:feature-pipeline. Read this when a post-build gate
returns REDESIGN, or when review.md's assessment is
`Architecture: Violated — needs redesign`.

REDESIGN means at least one finding cannot be fixed without changing
architecture.md. Code already exists, so this is the most expensive path in
the pipeline.

## Decide

- **Interactive:** stop and ask your human partner: rewind to architecture
  with these findings, or park them and continue. Show the finding IDs and
  the architecture decision each names.
- **Unattended:** rewind automatically the first time. A second REDESIGN in
  the same run is Blocked.
- **Cap:** two rewinds per run. A third REDESIGN always stops.

If the REDESIGN is parked, the FIX reports from the same round still get
their fix wave.

## Rewind procedure

In Spec Kit mode, follow the Rewind section of [speckit.md](speckit.md)
instead of the steps below.

1. Ledger `Rewind to stage 3: <finding IDs> — <reason>`. The resume point is
   now stage 3; stages after it re-run even though the ledger shows them
   complete earlier.
2. FIX reports from the same round are not fixed now — the code is about to
   change. They go to the architect with the REDESIGN reports, and the
   stage-7 re-run will check them again.
3. **Stage 3.** Rename architecture.md to `architecture-r<N>.md`. Dispatch the
   architect with `[PRIOR_FINDINGS_FILE]` listing the REDESIGN report(s),
   plus review.md if it triggered the rewind. The existing code is the
   starting point: the new architecture states which built components
   change and how.
4. **Stage 4.** Design gate as usual. Round counts restart for every stage
   after the rewind.
5. **Stage 5.** Write a delta plan,
   `docs/superpowers/plans/<date>-<feature>-redesign-<N>.md`, covering only
   the changes from the current code to the revised architecture. Same Spec
   line and Global Constraints rules. Commit it.
6. **Stage 6.** subagent-driven-development with the delta plan, on the same
   branch.
7. **Stages 7 onward** run again in full over the whole branch
   (MERGE_BASE..HEAD). Rename each stage's previous report to
   `<name>-r<N>.md` first, and pass it as `[PRIOR_FINDINGS_FILE]` so earlier
   findings are re-checked.
