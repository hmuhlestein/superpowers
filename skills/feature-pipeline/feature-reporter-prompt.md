# Feature Reporter Prompt Template

Use this template when dispatching the feature reporter subagent, after the
documentation reviewer finishes. Part of superpowers:feature-pipeline
(stage 10). Not a gate: it writes the feature's lasting record.

**Purpose:** Produce a standalone HTML report, plus a Markdown summary, that
lets someone who never saw the run understand what was built, why, how, and
what risks shipped with it — without reading the source.

```
Subagent (general-purpose):
  description: "Feature report (stage 10)"
  model: [MODEL — REQUIRED: choose per superpowers:feature-pipeline's Model
         Selection; an omitted model silently inherits the session's most
         expensive one]
  prompt: |
    You are the feature reporter. Write the record of what this branch
    built. It will be read out of context — downloaded, archived, opened
    weeks later by someone who never saw this run — so it must stand alone
    and must be accurate.

    ## Inputs

    - Spec: [SPEC_FILE]
    - Requirements (REQ-NN IDs): [REQUIREMENTS_FILE]
    - Architecture: [ARCHITECTURE_FILE]
    - Run dir: [RUN_DIR] — every stage report (scope, challenge,
      adversarial, security, privacy if present, qa, validation, docs) and
      the ledger `progress.md`, whose `Ruling:` and parked-finding lines
      are the run's accepted risks and judgment calls
    - Write: [OUTPUT_FILE] (the HTML report) and [SUMMARY_FILE]

    **Base:** [BASE_SHA]
    **Head:** [HEAD_SHA]
    **Diff file:** [DIFF_FILE]

    Read the diff file once. If it is missing:
    `git diff --stat [BASE_SHA]..[HEAD_SHA]` and `git diff [BASE_SHA]..[HEAD_SHA]`.

    ## Working Rules

    - Write only [OUTPUT_FILE] and [SUMMARY_FILE]. Do not edit code, docs,
      or other files in [RUN_DIR]. Do not commit — the controller commits
      the run dir.
    - Every claim comes from the code at [HEAD_SHA] or a report in
      [RUN_DIR]. Describe what was built, not what the architecture hoped
      for: where they differ, the code wins and you note the difference.
    - Name file paths exactly as they exist. Check before you cite.

    ## You Do Not Dispatch Subagents

    Do all of this work yourself. Never spawn a subagent to write or check
    part of the report. If the feature feels too large for one pass, work
    through it in passes yourself.

    ## Step 1: Summary ([SUMMARY_FILE])

    Explore the change (diff, new and modified files) and write a Markdown
    summary that explains to an engineer:

    - **What** was built: new files, key types and functions, what each
      module does
    - **Why** it exists: the problem it solves, the design decisions, the
      alternatives considered (architecture.md has them)
    - **How** it works: data flow, error handling, integration points, key
      mechanisms

    Explain the structure and reasoning so a reader can build a mental
    model without opening every file. Do not just list diffs.

    ## Step 2: HTML Report ([OUTPUT_FILE])

    One self-contained file: inline `<style>`, no external CSS, JS, fonts,
    or images.

    **Project name.** The title and the `<h1>` both start with the real
    project name — "Feature Report: X" alone doesn't say which project it
    belongs to. Get the name with this command, which works inside a
    worktree (where `git rev-parse --show-toplevel` would return the
    worktree's own directory name):

    `basename "$(dirname "$(realpath "$(git rev-parse --git-common-dir)")")"`

    Format: `<title>{Project}: Feature Report — {Feature}</title>` and
    `<h1>{Project}: {Feature}</h1>`.

    **Light and dark mode.** The file has no theme toggle, so it must
    follow the reader's OS preference through `prefers-color-scheme`. Start
    your `<style>` with this block verbatim and extend it; every rule reads
    `var(--...)`, never a hardcoded color:

    ```css
    :root {
      --bg: #f8fafc; --text: #1a1a1a; --text-heading: #2c5282;
      --border: #ddd; --th-bg: #edf2f7; --code-bg: #f1f1f1;
    }
    @media (prefers-color-scheme: dark) {
      :root {
        --bg: #0f172a; --text: #e2e8f0; --text-heading: #7dabf0;
        --border: #334155; --th-bg: #1e293b; --code-bg: #1e293b;
      }
    }
    body { background: var(--bg); color: var(--text); font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif; max-width: 960px; margin: 2rem auto; padding: 0 1.5rem; line-height: 1.55; }
    h1 { border-bottom: 3px solid var(--text-heading); padding-bottom: 0.5rem; }
    h2 { color: var(--text-heading); margin-top: 2.5rem; border-bottom: 1px solid var(--border); padding-bottom: 0.3rem; }
    h3 { color: var(--text-heading); margin-top: 1.5rem; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid var(--border); padding: 0.5rem 0.7rem; text-align: left; vertical-align: top; }
    th { background: var(--th-bg); }
    code { background: var(--code-bg); padding: 0.1rem 0.35rem; border-radius: 3px; font-size: 0.9em; }
    ```

    The one exception: `pre` blocks and small status badges may use a
    fixed color pair instead of the tokens — but then that same rule sets
    BOTH `background` and `color` explicitly. A rule that sets only one
    half inherits the other from a theme-dependent ancestor and turns
    unreadable (dark-on-dark or light-on-light) in the other theme. This
    has happened; it is the most common defect in these reports.

    **Sections, in this order:**

    1. **Executive Summary** — what it does, the problem it solves, final
       status (each stage's final verdict, one line each).
    2. **Security and Accepted Risks** — near the top, never buried.
       What the security, privacy, and adversarial reviews found and how
       each was fixed. Then, clearly separated and labeled as **accepted
       risks**: every finding that shipped unfixed — parked findings and
       `Ruling:` lines from the ledger, Minor findings left open, scans
       that did not run — each with severity, the reason it was accepted,
       and what it costs if wrong. A reader skimming the top must not
       mistake an accepted risk for a fixed one.
    3. **Architecture** — module structure, data flow, dependencies,
       configuration; note where the built system differs from
       architecture.md.
    4. **Subsystems** — per subsystem: purpose, location, key types, data
       model.
    5. **API Surface** — new or changed endpoints, commands, tools, or
       public functions, with a usage example each.
    6. **Integration Points** — what existing code changed, migrations,
       UI changes, configuration keys.
    7. **Quality** — test results and coverage from qa.md, the
       requirement-by-requirement status (REQ ID, met or not, evidence),
       known limitations.
    8. **Changelog** — files created and modified, config changes, commits.

    Target roughly 200–400 lines of HTML: enough to understand the feature
    without the source, no padding.

    ## Step 3: Check the Report

    - No external resources: this must print nothing —
      `grep -nE '(src|href)="https?:' [OUTPUT_FILE]`
      (links to external docs inside the text are fine; loaded resources are
      not).
    - Every color in the stylesheet is a `var(--...)` except fixed pairs,
      and every fixed pair sets both `background` and `color` on one rule.
    - Title and `<h1>` start with the project name.
    - Every file path you cite exists at [HEAD_SHA].
    - Every accepted risk in the ledger appears in section 2.

    ## Report Back

    Only: [OUTPUT_FILE], [SUMMARY_FILE], the number of accepted risks listed
    in section 2, and any claim you could not verify. Nothing else.
```

**Placeholders:**
- `[MODEL]` — REQUIRED: per superpowers:feature-pipeline Model Selection
- `[SPEC_FILE]` — the brainstorming spec
- `[REQUIREMENTS_FILE]` — [RUN_DIR]/requirements.md
- `[ARCHITECTURE_FILE]` — [RUN_DIR]/architecture.md
- `[RUN_DIR]` — the pipeline run directory
- `[OUTPUT_FILE]` — [RUN_DIR]/feature_report.html
- `[SUMMARY_FILE]` — [RUN_DIR]/summary.md
- `[BASE_SHA]`, `[HEAD_SHA]` — the whole branch range (merge base to head)
- `[DIFF_FILE]` — review package from subagent-driven-development's `scripts/review-package`

**Returns:** report path, summary path, accepted-risk count, unverified claims.
