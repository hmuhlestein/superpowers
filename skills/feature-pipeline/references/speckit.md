# Spec Kit Mode

Part of superpowers:feature-pipeline. Read this at Setup when the repo has a
`.specify/` directory and the spec is a Spec Kit feature: any directory under
`specs/` that holds a spec.md (sequential `NNN-name` or timestamp-numbered),
that spec.md itself, or the directory `.specify/feature.json` names.

Spec Kit already produces what stages 1–5 would: spec.md holds the
requirements, plan.md and its companions hold the design, tasks.md holds the
plan. The pipeline runs whichever Spec Kit commands are still missing, puts
its gates around them, and builds from tasks.md.

## Paths and setup

- `FEATURE_DIR` = the feature directory (repo-relative). **`[RUN_DIR]` =
  `FEATURE_DIR`**: pipeline files sit beside Spec Kit's — progress.md,
  challenge.md, analyze.md, pipeline-plan.md, pipeline-constraints.md,
  review.md, adversarial.md, security.md, privacy.md, qa.md, validation.md,
  docs.md, release.md, summary.md, feature_report.html, fix-*.md. None
  collide with Spec Kit's names.
- **Run detection.** Read `FEATURE_DIR/progress.md` directly. If its ledger
  shows `Stage 11 Release: complete`, this is a new run on a finished
  feature: move the previous run's pipeline files into
  `FEATURE_DIR/pipeline-archive/<date>/` (git mv) and start a fresh ledger.
  Otherwise resume it.
- Ledger first line: `# Pipeline ledger — spec: <FEATURE_DIR>/spec.md`.
- **Constitution.** `.specify/memory/constitution.md` binds only when it is
  filled in. It is the unfilled template when it still contains
  `[PROJECT_NAME]` or `[PRINCIPLE_1_NAME]` outside HTML comments (the Sync
  Impact Report comment speckit-constitution writes lists the placeholders it
  replaced), or its sha256 equals the one in
  `.specify/memory/.constitution-template.json`. Unfilled → there is no
  constitution; say "none" wherever this file names it.
- **Feature resolution.** Spec Kit finds the feature from
  `SPECIFY_FEATURE_DIRECTORY` or `.specify/feature.json`, not the branch
  name. Every Spec Kit command or script run against an existing feature gets
  `SPECIFY_FEATURE_DIRECTORY=<FEATURE_DIR>`. Exception: speckit-specify runs
  without it (it would copy the spec template over an existing spec.md); read
  FEATURE_DIR from its report.
- **`.specify/feature.json`** is rewritten by every Spec Kit script call. At
  Setup, if git doesn't ignore it, add `.specify/feature.json` to
  `.git/info/exclude`. Never commit it, and never copy it into the worktree.
- **Extension hooks.** Spec Kit commands run hooks from
  `.specify/extensions.yml`, and mandatory ones can switch branches or
  commit. In the pipeline, run no extension hooks, mandatory or optional,
  from any Spec Kit command; list the ones skipped in the ledger and the
  final message.

## Stage map

| Stage | In Spec Kit mode |
|---|---|
| 1 Requirements | None — spec.md is the requirements. No spec.md: interactive → run speckit-specify with your human partner's description; unattended → FAILED. |
| 2 Scope gate | Clarification check (below). |
| 3 Architecture | plan.md + research.md + data-model.md + contracts/ + quickstart.md. Not yet produced (below) → Spec Kit producer, `speckit-plan`. |
| 4 Design gate | design-challenger on that design set and the constitution. FIX → producer in revise mode. |
| 5 Plan | tasks.md missing → Spec Kit producer, `speckit-tasks`. Then the analyze gate, the coverage check, and conversion. |
| 6 Build | subagent-driven-development on pipeline-plan.md, then convergence. |
| 7–11 | As usual, with the Spec Kit dispatch note. |

**Produced, not just present.** speckit-plan writes the unfilled template
into plan.md first and can stop partway. Count plan.md as produced only when
the ledger records stage 3 complete, or — on a feature the pipeline didn't
start — when it contains none of `[FEATURE]`, `ACTION REQUIRED`, or
`[###-feature-name]` and research.md exists.

**Clarification check.** A real marker is `[NEEDS CLARIFICATION:` followed
by a question — outside backticks and code fences. Prose that quotes or
discusses the marker is not one. If `checklists/requirements.md` marks "No
[NEEDS CLARIFICATION] markers remain" as done, trust it. Real markers left →
interactive: run speckit-clarify with your human partner; unattended:
Blocked.

**Caps.** The design gate and the analyze gate get three rounds each, and
nothing is parked before the build; unattended runs at either cap are
Blocked. Commit FEATURE_DIR after every producer round, not only when a stage
completes, so each revision can be compared with the last.

## Spec Kit producer

Dispatch a general-purpose subagent (most capable model for `speckit-plan`,
standard otherwise):

> Invoke the `speckit-<command>` skill for the feature at `<FEATURE_DIR>` and
> follow it, with `SPECIFY_FEATURE_DIRECTORY=<FEATURE_DIR>` set for every
> Spec Kit script. Run no extension hooks, mandatory or optional; list the
> ones you skipped. Nobody can answer questions: where the command would
> ask, make the most conservative choice and list it under "Open questions"
> in your final message. Do not follow on to other commands. Do not commit.
> Do not dispatch subagents. Final message: the files you wrote, skipped
> hooks, then one line per open question.

**Revise mode** (after a design-gate or analyze-gate FIX, or a rewind):

> Revise `<FEATURE_DIR>`'s plan.md, research.md, data-model.md, contracts/,
> quickstart.md, and tasks.md in place to address every Critical and
> Important finding in `<report(s)>`. Do not rerun speckit-plan or
> speckit-tasks: they rebuild their files from templates, discarding the
> design and every `[X]` tick. Do not edit spec.md — a finding that needs a
> spec change goes under "Open questions". Never renumber or reword an
> existing task line; new work goes in new task lines with the next free
> T-IDs. Append to plan.md a `## Response to Design Challenge` section
> (create it once, then extend it): finding ID | CHANGED (where) or REJECTED
> (reason). Do not commit. Do not dispatch subagents. Final message: files
> changed, CHANGED and REJECTED counts, one line per open question.

## Analyze gate

After tasks.md exists, dispatch a general-purpose subagent (standard model):

> Invoke the `speckit-analyze` skill for `<FEATURE_DIR>`, with
> `SPECIFY_FEATURE_DIRECTORY=<FEATURE_DIR>` set. The analysis itself is
> read-only: apply and offer no remediation. When it is done, the pipeline —
> not the analysis — needs one file: save the full report, including any
> overflow summary, to `<FEATURE_DIR>/analyze.md`. Tag each CRITICAL or HIGH
> finding `[spec]` if fixing it requires changing spec.md, otherwise
> `[design]`. A constitution finding is resolved, not counted, when plan.md's
> Complexity Tracking justifies that exact deviation — say so. On a re-run,
> read `<prior analyze report>` and plan.md's `## Response to Design
> Challenge`: mark each prior CRITICAL/HIGH finding FIXED, NOT FIXED, or
> REJECTION ACCEPTED (the stated reason holds), and list NOT FIXED ones
> again with their severity. Then append `## Verdict` /
> `Verdict: PASS | FIX` / `Critical: N  Important: N  Minor: N` — CRITICAL is
> Critical, HIGH is Important, MEDIUM and LOW (overflow included) are Minor;
> PASS iff no Critical or Important. Final message: the verdict line, the
> counts line, the path, and one line per CRITICAL or HIGH finding with its
> tag. Run no extension hooks. Do not dispatch subagents.

- `[design]` findings → the producer in revise mode with analyze.md, then the
  analyze gate again (renaming analyze.md to `analyze-r<N>.md` and passing it
  as the prior report).
- `[spec]` findings are your human partner's: interactive → ask (they may run
  speckit-clarify or edit spec.md); unattended → Critical is Blocked;
  Important gets a conservative Ruling written into spec.md's
  `## Clarifications` (below) and counts as resolved.

## Coverage check, constraints, and conversion

Spec Kit includes tests only when the spec asks for them, so check coverage
yourself: every FR-NNN, every user story's acceptance scenario, and every
SC-NNN that can be tested maps to a task whose tests exercise it. Where one
doesn't, have the producer (revise mode) append test task lines with new
T-IDs in the right phase.

Write `FEATURE_DIR/pipeline-constraints.md` once, as bullet lines: tests
are required for every task's behavior; the changed-line coverage floor (80%
unless spec, plan, or constitution sets one); and, verbatim from plan.md and
the constitution, the security and privacy decisions (including log
redaction) and configuration decisions. The converter appends this file to
the plan's Global Constraints every time it runs.

Then run `bash <this skill>/scripts/speckit-plan-from-tasks <FEATURE_DIR>`.
It writes `pipeline-plan.md`: one `### Task <T-ID>` per tasks.md section that
still has unchecked lines, identified by the section's first Spec Kit task
ID, with the text verbatim and the instruction to tick each line in tasks.md
when it is finished. IDs never change between runs, so
subagent-driven-development's ledger stays valid when the plan is
regenerated. Exit 3: nothing left to build. Exit 4: task lines outside any
section — fix tasks.md first.

**Verification-only lines** (run the scenarios against a live instance, run
the full suite, manual checks) are not implementation work. Tell
implementers: leave such a line unchecked and say so in the report. QA
performs them in stage 8; tick them after QA passes, in the same commit as
qa.md.

Commit tasks.md, the design files, pipeline-constraints.md, and
pipeline-plan.md.

## Build and convergence

In this order, ledgering each step:

1. Run superpowers:subagent-driven-development with pipeline-plan.md as the
   plan. For its final review, `[ARCHITECTURE_OR_NONE]` is plan.md (with
   data-model.md and contracts/ beside it); save the report to review.md.
   Ledger `Build round 1: complete`.
2. Apply SKILL.md's stage-6 routing (copy Rulings; "Violated" → Rewind
   below).
3. Dispatch the Spec Kit producer with `speckit-converge`, adding: "The
   pipeline's subagent-driven build counts as speckit-implement having run.
   Tests, and everything under `<FEATURE_DIR>`, are in scope — never flag
   them as unrequested." Ledger `Converge round 1: <appended N | clean>`.
4. If converge appended a section: commit tasks.md, convert again, and run
   subagent-driven-development on the regenerated plan, saving its final
   review to `review-converge.md` (review.md keeps the first round's open
   findings for stage 7; carry both). Ledger `Build round 2: complete`, and
   apply the stage-6 routing again.
5. Converge once more. Anything it still appends is ledgered as a parked
   finding at converge's own severity (HIGH → Important, MEDIUM or LOW →
   Minor). `unrequested` items never go to a build round; ledger them as
   Minor for the stage-7 reviewers to weigh.

## Rewind (Spec Kit)

When a post-build gate returns REDESIGN, references/redesign.md decides
whether to rewind. In Spec Kit mode the rewind itself is:

1. Ledger `Rewind to stage 3: <finding IDs> — <reason>`.
2. Producer in revise mode with the REDESIGN reports (plus review.md if it
   triggered the rewind): revise the design files, then append a new
   `## Phase N: Redesign` section to tasks.md with new T-IDs for the changes
   from the built code to the revised design.
3. Design gate, analyze gate, coverage check, and conversion as above —
   only the new unchecked lines become tasks.
4. Build and convergence, then stages 7 onward in full over the whole
   branch, with prior reports renamed to `<name>-r<N>.md`.

## Where answers go

Answers to open questions go where the next subagent reads them, never only
in the ledger: answers about the spec go in spec.md's `## Clarifications`
section (Spec Kit's own convention: a `### Session <date>` heading with
`- Q: … → A: …` lines — this is the one edit you make to spec.md); answers
about the design go in plan.md under `## Resolved Questions`. Commit both.

## Dispatch note for stages 4 and 6–11

Add this paragraph to every dispatch in those stages, including fix-wave
fixers, rewinds, and SDD implementers. It replaces SKILL.md's
docs/superpowers pipeline-records line:

> Spec Kit mode. `[REQUIREMENTS_FILE]` is spec.md: its FR-NNN functional
> requirements, each user story's acceptance scenarios (cite them as
> US<n>-AS<m>), and its SC-NNN success criteria stand in for REQ-NN and
> REQ-NN.AC-M; its Key Entities section stands in for a Personal Data
> section, and its `## Clarifications` are binding answers.
> `[ARCHITECTURE_FILE]` is plan.md, with research.md, data-model.md,
> contracts/, and quickstart.md beside it as part of the design; where your
> instructions name an architecture.md section, judge its equivalent across
> these files, and the response to prior findings is plan.md's
> `## Response to Design Challenge`. Constitution: `<path, or none>` — when
> present it is binding, and a violated MUST principle is Critical unless
> plan.md's Complexity Tracking justifies it. Paths under `<FEATURE_DIR>`
> and `.specify/` are pipeline records, not feature changes — exclude them
> from scope, hygiene, coverage, and scan judgments.

QA and product validation alone also get: "quickstart.md's scenarios are
acceptance checks. Run the ones that work against a local build; list any
that need a live or shared environment as UNVERIFIED with the reason —
never run them against shared infrastructure."

Privacy applies when spec.md's Key Entities or FRs name personal data, or
the diff handles it.

## Unattended

- The spec argument may be `FEATURE_DIR` or its spec.md.
- No spec.md → FAILED. Real `[NEEDS CLARIFICATION:` markers → Blocked;
  blocked.md names the run's worktree path, and your human partner runs
  speckit-clarify (or edits spec.md) **there**, commits, and relaunches —
  the run reads its own branch, not the checkout they started from.
- plan.md or tasks.md not yet produced → the pipeline produces them.
- Design gate or analyze gate at its cap with Critical or Important open →
  Blocked.
