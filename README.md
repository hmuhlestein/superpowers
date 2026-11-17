# Superpowers

Superpowers is a complete software development methodology for your coding agents, built on top of a set of composable skills and some initial instructions that make sure your agent uses them.

## This Fork

This is [hmuhlestein/superpowers](https://github.com/hmuhlestein/superpowers), a fork of [obra/superpowers](https://github.com/obra/superpowers). It adds a gated spec-to-release pipeline ported from the [HephaestusNG](https://github.com/hmuhlestein/HephaestusNG) autopilot workflow, rebuilt in superpowers' own style: prompt templates dispatched to fresh subagents, not named agents or a backend.

### Install the fork (Claude Code)

```bash
claude plugin marketplace add hmuhlestein/superpowers   # registers "superpowers-dev"
claude plugin install superpowers@superpowers-dev --scope user
```

To pick up a new version: `claude plugin marketplace update superpowers-dev && claude plugin update superpowers@superpowers-dev`, then restart Claude Code. Installed copies are cached by version, so every release that changes skill content bumps the version in all 12 manifests listed in `.version-bump.json`.

### What the fork adds

**`feature-pipeline` skill** ([skills/feature-pipeline/](skills/feature-pipeline/)) takes one feature from an approved spec to a release-ready branch. Every artifact is judged by a context that did not write it:

| # | Stage | Template / skill |
|---|---|---|
| 1–2 | Requirements, then a scope gate against the spec | `requirements-analyst`, `scope-reviewer` |
| 3–4 | Architecture, then an adversarial design gate | `architect`, `design-challenger` |
| 5–6 | Plan and build | `writing-plans`, `subagent-driven-development` (final review checks architecture conformance) |
| 7 | Parallel adversarial, security, and privacy reviews | `adversarial-reviewer`, `security-reviewer` (runs AWS Automated Security Helper via `uvx`), `privacy-reviewer` (only when personal data is involved) |
| 8–9 | QA, then product validation | `qa-validator`, `product-validator` |
| 10 | Docs fixed in place | `doc-reviewer` |
| 11 | Release prep, HTML feature report, integration, post-PR CI and review follow-up, deploy only with `DEPLOY.md` and explicit approval | `release-engineer`, `feature-reporter`, `finishing-a-development-branch` |

Gates return `PASS`, `FIX`, or `REDESIGN`. A FIX runs one fix subagent, with a regression test proven to fail first for each fix, then re-reviews the fix. Every loop is capped. Pre-build gates never wave findings through; post-build gates park Important findings with a recorded ruling and stop on open Critical ones. Each run keeps a resumable ledger and writes a standalone, light/dark-aware `feature_report.html`, with every accepted risk called out near the top. Details: [SKILL.md](skills/feature-pipeline/SKILL.md) and [references/](skills/feature-pipeline/references/).

### Hands-free mode (unattended runs)

Hands-free mode runs the whole pipeline in a headless `claude -p` session, from a spec you have already written to a finished branch with its feature report. Nobody is asked anything along the way. Use it when the spec is settled and you want the result, not the conversation.

**Before you launch**

- **A written spec.** Hands-free mode never brainstorms or runs `speckit-specify`. Write the spec first, interactively: with superpowers:brainstorming, or with `speckit-constitution` and `speckit-specify` in a Spec Kit project. Settle any `[NEEDS CLARIFICATION: …]` markers first (`speckit-clarify`), because a run that finds one stops as BLOCKED.
- **Committed spec.** The run works in its own worktree. It copies uncommitted spec files in, but committing them first keeps things simple.
- **Tools.** You need the `claude` CLI and this plugin installed. `uv` enables the security scan; without it the run continues and lists the missing scan as an accepted risk. `gh` is needed only for `--integration pr`.

**Launch**

```bash
LAUNCHER=~/.claude/plugins/cache/superpowers-dev/superpowers/<version>/skills/feature-pipeline/scripts/run-unattended

# a plain spec
bash "$LAUNCHER" docs/superpowers/specs/2026-11-17-export-design.md -- --dangerously-skip-permissions

# a Spec Kit feature: pass the feature directory or its spec.md
bash "$LAUNCHER" specs/001-cli-calculator --integration keep -- --dangerously-skip-permissions
```

| Option | Meaning |
|---|---|
| `--integration keep` (default) | Leave the finished branch and worktree for you to review and merge. |
| `--integration pr` | Push the branch, open a PR with the generated title and body, then follow CI and automated review comments for up to two fix rounds. It never merges and never force-pushes. |
| `--log FILE` | Where the transcript goes (default `$TMPDIR/feature-pipeline/<timestamp>.log`). |
| `-- <claude flags>` | Passed straight to `claude -p`. **Permissions go here.** A headless run can't answer permission prompts, so anything not pre-approved is denied and the run fails. `--dangerously-skip-permissions` is the simplest choice, but it gates nothing, so use it in a container, VM, or throwaway checkout. The narrower alternative is `--permission-mode acceptEdits --allowedTools Bash Read Write Edit Glob Grep Agent Skill`. |

**How it decides without you**

- **Open questions:** it takes the most conservative reading of the spec (least scope, least data exposure, nothing irreversible). Each one is logged as `Ruling: [unattended] …` and written where later stages read it: `## Resolved Questions` in its own documents, or `## Clarifications` in a Spec Kit spec.md.
- **Gate findings:** they are fixed and re-reviewed. Post-build Important findings still open after three rounds are parked with a ruling, and every parked finding appears in the report's Accepted Risks section.
- **A redesign verdict after code exists:** the run rewinds to architecture once.
- **Never, in hands-free mode:** merging, deploying (even when `DEPLOY.md` exists), rewriting history, force-pushing, or any destructive or security-sensitive action.

**When it stops as BLOCKED**

It stops as BLOCKED instead of guessing when:

- a pre-build gate (requirements, design, or the Spec Kit analyze gate) is still failing after three rounds;
- a Critical finding is still open at a cap;
- a second redesign is needed;
- a merge conflict appears with the base branch;
- a release blocker needs a human (secret rotation, history rewrite);
- the spec has unresolved clarification markers.

It commits its run directory and writes `blocked.md` there, naming the decisions it needs and the worktree path. Answer in `<run dir>/human-input.md` (for Spec Kit clarifications, edit spec.md in that worktree), commit, and relaunch with the same spec. It resumes from its ledger.

**What you get**

The run's final lines are machine-readable, and the launcher's exit code matches them:

```
PIPELINE REPORT: specs/001-cli-calculator/feature_report.html
PIPELINE STATUS: COMPLETE — <reason>
```

Exit codes: 0 COMPLETE, 2 BLOCKED, 1 FAILED or no status line, 64 bad arguments. The run directory (`docs/superpowers/pipeline/<date>-<feature>/`, or the Spec Kit feature directory) holds:

- `progress.md`, the ledger of every stage, verdict, fix wave, and ruling;
- every gate report, including earlier rounds as `<name>-r<N>.md`;
- `release.md`, with the PR draft and any deploy preflight;
- `walkthrough.md` (what was built, why, and how) and `feature_report.html`;
- `build-reports/`, the build's per-task reports with the test-first evidence.

While it works, the launcher prints each new ledger line (stages, verdicts, fix waves, rulings, blocks) as `[progress] …`. The headless session itself prints only its final message.

**Native Spec Kit support.** When a repo has `.specify/` and the spec is a Spec Kit feature (`specs/NNN-name/`), the pipeline resumes from whatever Spec Kit artifacts exist instead of writing its own:

- spec.md serves as the requirements, plan.md and its companion files as the design, tasks.md as the plan.
- Missing `speckit-plan` and `speckit-tasks` steps run as headless subagents.
- `speckit-analyze` becomes a gate before the build, and `speckit-converge` closes gaps after it.
- The constitution is binding on every review.
- [scripts/speckit-plan-from-tasks](skills/feature-pipeline/scripts/speckit-plan-from-tasks) turns tasks.md into a subagent-driven-development plan, one task per phase, keyed by T-ID so progress survives regeneration. Implementers tick tasks.md as they go.

Run files live in the feature directory beside Spec Kit's own. See [references/speckit.md](skills/feature-pipeline/references/speckit.md).

### Changes to upstream skills

- **requesting-code-review**: `code-reviewer.md` takes an optional `[ARCHITECTURE_OR_NONE]`. Given an architecture document, it also checks conformance (components, contracts, invariants, boundaries, error and concurrency model, wiring) and reports `Architecture: Conforms | Deviates | Violated — needs redesign`. Given `none`, it behaves as upstream.
- **subagent-driven-development** (final review) and **executing-plans** pass the architecture document through to that reviewer.
- **brainstorming** returns to `feature-pipeline` when the pipeline sent it there, instead of going straight to writing-plans.
- **using-superpowers** routes "run the full pipeline on X" to `feature-pipeline`.

## Table of Contents

- [This Fork](#this-fork)
- [How it works](#how-it-works)
- [Commercial Services](#commercial-services)
- [Getting Started](#installation)
  - [Claude Code](#claude-code)
  - [Antigravity](#antigravity)
  - [Codex App](#codex-app)
  - [Codex CLI](#codex-cli)
  - [Cursor](#cursor)
  - [Devin CLI](#devin-cli)
  - [Factory Droid](#factory-droid)
  - [Gemini CLI](#gemini-cli)
  - [GitHub Copilot CLI](#github-copilot-cli)
  - [Grok Build CLI](#grok-build-cli)
  - [Kimi Code](#kimi-code)
  - [OpenCode](#opencode)
  - [Pi](#pi)
  - [Qwen Code](#qwen-code)
  - [Hermes Agent](#hermes-agent)
  - [Muse](#muse)
- [The Basic Workflow](#the-basic-workflow)
- [When Something Goes Wrong](#when-something-goes-wrong)
- [Community](#community)
- [What's Inside](#whats-inside)
- [Philosophy](#philosophy)
- [Contributing](#contributing)
- [Updating](#updating)
- [License](#license)
- [Visual companion telemetry](#visual-companion-telemetry)

## How it works

It starts from the moment you fire up your coding agent. As soon as it sees that you're building something, it *doesn't* just jump into trying to write code. Instead, it steps back and asks you what you're really trying to do. 

Once it's teased a spec out of the conversation, it shows it to you in chunks short enough to actually read and digest. 

After you've signed off on the design, your agent puts together an implementation plan that's clear enough for an enthusiastic junior engineer with poor taste, no judgement, no project context, and an aversion to testing to follow. It emphasizes true red/green TDD, YAGNI (You Aren't Gonna Need It), and DRY. 

Next up, once you say "go", it launches a *subagent-driven-development* process, having agents work through each engineering task, inspecting and reviewing their work, and continuing forward. It's not uncommon for your agent to work autonomously for a couple hours at a time without deviating from the plan you put together.

There's a bunch more to it, but that's the core of the system. And because the skills trigger automatically, you don't need to do anything special. Your coding agent just has Superpowers.

## Commercial Services

If you're using Superpowers in enterprise and could benefit from commercial support, additional tooling, or managed spending, please don't hesitate to drop us a line at sales@primeradiant.com.

## Installation

Installation differs by harness. If you use more than one, install Superpowers separately for each one.

### Claude Code

Superpowers is available via the [official Claude plugin marketplace](https://claude.com/plugins/superpowers)

#### Official Marketplace

- Install the plugin from Anthropic's official marketplace:

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers Marketplace

The Superpowers marketplace provides Superpowers and some other related plugins for Claude Code.

- Register the marketplace:

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- Install the plugin from this marketplace:

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

Install Superpowers as a plugin from this repository:

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity lists each installed skill's description, and the description of
the `using-superpowers` skill prompts the model to load that skill at the start
of a conversation. Reinstall with the same command to update.

### Codex App

Superpowers is available via the [official Codex plugin marketplace](https://github.com/openai/plugins).

- In the Codex app, click on Plugins in the sidebar.
- You should see `Superpowers` in the Coding section.
- Click the `+` next to Superpowers and follow the prompts.

### Codex CLI

Superpowers is available via the [official Codex plugin marketplace](https://github.com/openai/plugins).

- Open the plugin search interface:

  ```bash
  /plugins
  ```

- Search for Superpowers:

  ```bash
  superpowers
  ```

- Select `Install Plugin`.

### Cursor

- In Cursor Agent chat, install from marketplace:

  ```text
  /add-plugin superpowers
  ```

- Or search for "superpowers" in the plugin marketplace.

### Devin CLI

- Install the plugin from this repository:

  ```bash
  devin plugins install obra/superpowers
  ```

- Update to the latest version with:

  ```bash
  devin plugins update superpowers
  ```

### Factory Droid

- Register the marketplace:

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- Install the plugin:

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- Install the extension:

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- Update later:

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- Register the marketplace:

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- Install the plugin:

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Grok Build CLI

Superpowers is available via the [official Grok plugin marketplace](https://github.com/xai-org/plugin-marketplace).

- Install the plugin from xAI's official marketplace:

  ```bash
  grok plugin install superpowers@xai-official --trust
  ```

- Or open the marketplace in the TUI, search for Superpowers, and install it:

  ```text
  /marketplace
  ```

### Kimi Code

Superpowers is available in Kimi Code's plugin marketplace.

- Open Kimi Code's plugin manager:

  ```text
  /plugins
  ```

- Go to `Marketplace` > `Superpowers` and install it.

- Or install directly from this repository:

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- Detailed docs: [docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode uses its own plugin install; install Superpowers separately even if you
already use it in another harness.

- Tell OpenCode:

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- Detailed docs: [docs/README.opencode.md](docs/README.opencode.md)

### Pi

Install Superpowers as a Pi package from this repository:

```bash
pi install git:github.com/obra/superpowers
```

For local development, run Pi with this checkout loaded as a temporary package:

```bash
pi -e /path/to/superpowers
```

The Pi package loads the Superpowers skills and a small extension that injects the `using-superpowers` bootstrap at session startup and again after compaction. Pi has native skills, so no compatibility `Skill` tool is required. Subagent and task-list tools remain optional Pi companion packages.

### Qwen Code

Qwen Code installs plugins from Claude Code marketplaces directly.

- Install the plugin from this repository, and pick `superpowers` when prompted:

  ```bash
  qwen extensions install obra/superpowers
  ```

- Update later:

  ```bash
  qwen extensions update superpowers
  ```

### Hermes Agent

Install Superpowers as a Hermes plugin from this repository:

```bash
hermes plugins install obra/superpowers --enable
```

Restart any active Hermes sessions after installing. Note: Hermes has no
post-compaction hook, so a very long session that compacts over its first
turn loses the bootstrap — start a fresh session if skills stop triggering.

### Muse

Superpowers is available as a native Muse plugin — same repo, same skills, all harnesses. The `using-superpowers` bootstrap is injected via the native `SessionStart` hook alongside Claude Code, Codex, Cursor, Gemini, Pi, and the rest — no per-session opt-in.

- Install from a local checkout:

  ```bash
  muse plugins install ./
  muse plugins approve superpowers
  ```

  Or clone and install:

  ```bash
  git clone https://github.com/obra/superpowers.git
  muse plugins install ./superpowers
  muse plugins approve superpowers
  ```

- Update later:

  ```bash
  muse plugins update superpowers
  ```

Restart any active Muse sessions after installing so the `SessionStart` hook takes effect — skills are active immediately, hooks require approval on first install. To verify, start a fresh session and send `Let's make a react todo list` — a working install auto-triggers `brainstorming` before any code is written. Version is tracked in `.version-bump.json` so `scripts/bump-version.sh` keeps it in sync.

## The Basic Workflow

1. **brainstorming** - Activates before building anything. Draws out what you actually want, and why, through questions that get you describing it, then plays its understanding back for correction. Small changes get agreed in chat; projects get a written design document a builder could plan from.

2. **using-git-worktrees** - Activates after design approval. Creates isolated workspace on new branch, runs project setup, verifies clean test baseline.

3. **writing-plans** - Activates with approved design. Breaks work into bite-sized tasks (2-5 minutes each). Every task has exact file paths, complete code, verification steps.

4. **subagent-driven-development** or **executing-plans** - Activates with plan. Either dispatches a fresh subagent per task with a review after each (most thorough), or implements every task inline in the current session with one fresh review of the whole branch at the end (cheapest).

5. **test-driven-development** - Activates during implementation. Enforces RED-GREEN-REFACTOR: write failing test, watch it fail, write minimal code, watch it pass, commit. Deletes code written before tests.

6. **requesting-code-review** - Activates between tasks. Reviews against plan, reports issues by severity. Critical issues block progress.

7. **finishing-a-development-branch** - Activates when tasks complete. Verifies tests, presents options (merge/PR/keep/discard), cleans up worktree.

**The agent checks for relevant skills before any task.** Mandatory workflows, not suggestions.

## When Something Goes Wrong

Sometimes a session misbehaves: a skill fires when it shouldn't, stays silent when it should, or the agent ignores its plan, repeats work, or burns more tokens than you'd expect. Ask your coding agent to "figure out what went wrong with superpowers in this session" and it will invoke the **diagnosing-superpowers** skill. To examine an earlier session, name it: "figure out what went wrong with superpowers in session `<id>`".

The skill reads the session transcript, reports what happened with line-level evidence, and, if you want, packages a scrubbed bundle for a bug report.

## Community

Superpowers is built by [Jesse Vincent](https://blog.fsck.com) and the rest of the folks at [Prime Radiant](https://primeradiant.com).

- **Discord**: [Join us](https://discord.gg/35wsABTejz) for community support, questions, and sharing what you're building with Superpowers
- **Issues**: https://github.com/obra/superpowers/issues
- **Release announcements**: [Sign up](https://primeradiant.com/superpowers/) to get notified about new versions

## What's Inside

### Skills Library

**Testing**
- **test-driven-development** - RED-GREEN-REFACTOR cycle (includes testing anti-patterns reference)

**Debugging**
- **systematic-debugging** - 4-phase root cause process (includes root-cause-tracing, defense-in-depth, condition-based-waiting techniques)
- **verification-before-completion** - Ensure it's actually fixed
- **diagnosing-superpowers** - Work out what went wrong in a session, with evidence; export a scrubbed bundle or file an issue

**Collaboration** 
- **brainstorming** - Drawing out intent before building
- **writing-plans** - Detailed implementation plans
- **executing-plans** - Inline plan execution: one context, one final review
- **dispatching-parallel-agents** - Concurrent subagent workflows
- **requesting-code-review** - Pre-review checklist
- **receiving-code-review** - Responding to feedback
- **using-git-worktrees** - Parallel development branches
- **finishing-a-development-branch** - Merge/PR decision workflow
- **subagent-driven-development** - Fast iteration with two-stage review (spec compliance, then code quality)
- **feature-pipeline** - Gated spec-to-release pipeline: requirements, architecture, design challenge, adversarial/security/privacy review, QA, product validation, docs, HTML feature report, release. Runs interactively or unattended (`skills/feature-pipeline/scripts/run-unattended SPEC_FILE`), and natively on Spec Kit features (`specs/NNN-name/`): runs the missing Spec Kit commands, gates them, and builds from tasks.md

**Meta**
- **writing-skills** - Create new skills following best practices (includes testing methodology)
- **using-superpowers** - Introduction to the skills system

## Philosophy

- **Test-Driven Development** - Write tests first, always
- **Systematic over ad-hoc** - Process over guessing
- **Complexity reduction** - Simplicity as primary goal
- **Evidence over claims** - Verify before declaring success

Read [the original release announcement](https://blog.fsck.com/2025/10/09/superpowers/).

## Contributing

The general contribution process for Superpowers is below. Keep in mind that we don't generally accept contributions of new skills and that any updates to skills must work across all of the coding agents we support. We also don't accept agent-written PRs that fix issues filed by someone else unless a maintainer asked for one, even when you hit the same problem; add your report to the existing issue instead.

1. Fork the repository
2. Switch to the 'dev' branch
3. Create a branch for your work
4. Follow the `writing-skills` skill for creating and testing new and modified skills
5. Submit a PR, filling in every section of the pull request template, including the summary a human writes in their own words.

Skill-behavior tests use the drill eval harness from [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/), cloned into `evals/` — see `evals/README.md` for setup. Plugin-infrastructure tests live at `tests/` and run via the relevant `run-*.sh` or `npm test`.

See `skills/writing-skills/SKILL.md` for the complete guide.

## Updating

Superpowers updates are somewhat coding-agent dependent, but are often automatic.

## License

MIT License - see LICENSE file for details

## Visual companion telemetry

Because skills and plugins don't provide any feedback to creators, we have no idea how many of you are using Superpowers. By default, the Prime Radiant logo on brainstorming's optional visual companion feature is loaded from our website. It includes the version of Superpowers in use. It does not include any details about your project, prompt, or coding agent. We don't see your clicks or anything about what you're building. This helps us have a rough idea of how many folks are using Superpowers and which version of Superpowers they're using. It's 100% optional. To disable this, set the environment variable `SUPERPOWERS_DISABLE_TELEMETRY` to any true value. Superpowers also honors Claude Code's `DISABLE_TELEMETRY` and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` opt-outs.
