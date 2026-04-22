# better-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). It summarizes inactive issues, applies a `Stale` label, posts a tailored comment, closes them after a quiet period, and removes `Stale` when a **non-bot** user engages again. 

Policy lives in `## Configuration` in the workflow markdown (`workflows/better-stale-bot.md` in this repository): `days-before-stale` (default 60), `days-before-close` (default 7), and exempt labels. Per-run write caps are only in YAML `safe-outputs` → `max:` (frontmatter); recompile after changing them.

Unlike timer-only bots, the model reads each thread end to end and drafts issue-specific comments.

**What it does**

- Rank and process low-engagement issues first  
- Summarize the thread; note resolved vs open  
- Apply `Stale` label, comment, close after `days-before-close`, remove `Stale` label on non-bot activity  
- Respect exempt labels (`agentic-workflows`, `pinned`, `security`, `help wanted` by default)  
- Write volume is capped by compiled `safe-outputs` (`add-comment`, `add-labels`, etc.); tune `max:` in frontmatter and `gh aw compile`
- Comments in the issue title's language through prompting
- Between runs, cache-memory stores a short run summary (`## Step 5` in the workflow); the instructions tell the agent to read it at the start of later runs to avoid blind reprocessing

## Table of contents

- [Repository layout](#repository-layout)
- [Installation](#installation)
- [Migrating from GitHub's stale bot](#migrating-from-githubs-stale-bot)
- [Migrating from Dosu's stale bot](#migrating-from-dosus-stale-bot)
- [Customization](#customization)
- [No-op posted as an issue](#no-op-posted-as-an-issue)
- [Tips](#tips)
- [Additional Resources](#additional-resources)
- [License](#license)

## Repository layout

This repository keeps two copies of the Agentic Workflow markdown:

| Location | Role |
| --- | --- |
| `workflows/` | **Distribution** — the template for others to copy, please install from this. |
| `.github/workflows/` | **Runs in this repo** — where it should be for GitHub Actions, contains `better-stale-bot.lock.yml` compiled from `gh aw compile`, and used for testing. |

The two `better-stale-bot.md` files should stay in sync, when `.github/workflows/` is modified, the distribution copy should be updated.

## Installation

**Prerequisites**

- [GitHub Actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository) enabled
- [GitHub CLI](https://cli.github.com/) 2.0+
- AI account: [Copilot](https://github.com/features/copilot), [Claude](https://www.anthropic.com/), or [Codex](https://openai.com/api/)

**Repository secret** ([engine auth](https://github.github.com/gh-aw/reference/auth/)) — repo **Settings → Secrets and variables → Actions** (the wizard in Option A can help set this; Option B assumes you add it yourself)

| Engine  | Secret               |
| ------- | -------------------- |
| Copilot | `COPILOT_GITHUB_TOKEN` |
| Claude  | `ANTHROPIC_API_KEY`  |
| Codex   | `OPENAI_API_KEY`     |

The workflow defaults to Claude Haiku in YAML frontmatter (`engine: id: claude, model: haiku`). Use `ANTHROPIC_API_KEY` unless you change the engine and recompile. The template schedules a daily run (`on: schedule: daily`); adjust in frontmatter or during Option A setup if you want a different frequency.

### Option A: add-wizard (recommended)

Run from a clone of the repository where you want the bot (you need write access so the wizard can open a PR if needed).

```bash
# 1. Authenticate with GitHub
gh auth login

# 2. Install the gh-aw CLI extension
gh extension install github/gh-aw

# 3. Add the workflow (sets secret, adds workflow, opens a PR — may need to manually merge)
# add a `--skip-secret` flag if you wish to set secret via GitHub UI instead
gh aw add-wizard dosu-ai/better-stale-bot/better-stale-bot

# 4. Pull locally (if needed)
git pull
```

### Option B: Manual setup (alternative)

In a local clone of the target repository, download the distribution markdown ([`workflows/better-stale-bot.md`](https://github.com/dosu-ai/better-stale-bot/blob/main/workflows/better-stale-bot.md) on the default branch; see [Repository layout](#repository-layout)), compile, commit, and push.

```bash
# 1. Install gh-aw
gh auth login
gh extension install github/gh-aw

# 2. Add your engine secret in repo Settings → Secrets and variables → Actions
# ANTHROPIC_API_KEY (Claude), COPILOT_GITHUB_TOKEN (Copilot), or OPENAI_API_KEY (Codex)

# 3. Download, compile, and push
mkdir -p .github/workflows
curl -fsSL -o .github/workflows/better-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/workflows/better-stale-bot.md
gh aw compile better-stale-bot
git add .github/ .gitattributes
git commit -m "Add better-stale-bot workflow"
git push
```

**Run**

- `gh aw run better-stale-bot`, or open the **Actions** tab and run the workflow

## Migrating from GitHub's stale bot

If you use [actions/stale](https://github.com/actions/stale) (or another workflow that wraps it), disable or remove the stale workflow and any schedule so two bots do not compete on items.

Then follow [Installation](#installation). Conceptually, most settings line up with defaults here: inactivity before marking stale, quiet period before closing, exempt labels, and a `Stale`-style label are all driven from `## Configuration` in `better-stale-bot.md` (defaults **60** / **7** days and the exempt list in that file). Adjust that section and frontmatter `safe-outputs` → `max:` to match how many writes you want per run.

**Issues only by default** — the shipped workflow is built for issues, not PRs:

- **Access:** frontmatter `permissions` and `tools.github` / `toolsets` are scoped to issues
- **Writes:** issue-oriented `safe-outputs` (for example `close-issue`)
- **Instructions:** the markdown body tells the agent to list, rank, and act on issues

**Pull requests:** if you want the same behavior for PRs, modify the workflow markdown:

- Widen YAML frontmatter `permissions` (for example `pull-requests`)
- Grant the GitHub `tools` the agent will need for PRs
- Add or adjust `safe-outputs` for any new write types (comments or state changes on PRs)
- Update the markdown instructions so the agent lists, ranks, and mutates PRs instead of (or in addition to) issues

Run `gh aw compile` after frontmatter edits

## Migrating from Dosu's stale bot

- Turn off the previous automation (Dosu's deployment stale bot and/or workflows like `actions/stale`) so two bots do not compete on items.
- Follow [Installation](#installation)
- Map old settings using the `## Configuration` section and frontmatter in `better-stale-bot.md`:
  - Edit the Configuration table for thresholds and exempt labels; use label names as they appear on GitHub.
  - Change Per-run caps (`safe-outputs` → `max:`) only in frontmatter, then run `gh aw compile`. Each output type is capped separately; see [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/).

| Setting | Dosu stale bot defaults | better-stale-bot defaults |
| --- | --- | --- |
| `days-before-stale` | 90 | 60 |
| `days-before-close` | 7 | 7 |
| Per-run caps | 25 | 30 |
| Exempt labels | none | `agentic-workflows`, `pinned`, `security`, `help wanted` |

- Codex (GPT-style models): `engine: codex`, optional `model:`, add `OPENAI_API_KEY`, recompile — see [engines](https://github.github.com/gh-aw/reference/engines/)

## Customization

- **Rename workflow:** rename the markdown file in `.github/workflows/`, then `gh aw compile <name>`, drop the old `.lock.yml` if it remains
- **Frontmatter**: modify AI engine and model choice, `safe-outputs` → `max:` (per output type), schedule frequency. **recompile** with `gh aw compile` after changes
  - **AI Engines:** e.g. `engine: copilot`; `engine: { id: claude, model: haiku }`; `engine: claude` (Sonnet); `engine: codex` (+ optional `model:`) — match the repo secret to the engine 
- **Markdown body:** takes effect on the next run — no recompile needed unless frontmatter changed
  - `## Configuration`: edit `days-before-stale`, `days-before-close`, and exempt labels; keep Guidelines aligned
- **Edit with a coding agent:** tell your agent to use https://raw.githubusercontent.com/github/gh-aw/main/create.md as the base spec so the agent follows gh-aw formatting and safety (YAML frontmatter + instruction body), say what you want changed, and reference `better-stale-bot.md` as the file to update. See more in [Additional Resources](#additional-resources)

## No-op posted as an issue

The default exempt list includes `agentic-workflows` because Agentic Workflows can open a repository issue such as `[aw] No-Op Runs` to track `noop` runs (when the agent reports that there is nothing to do). That issue is labeled `agentic-workflows`, and without an exemption this stale bot could keep summarizing, labeling, or closing it even though Agentic Workflows created it.

If you don't need better-stale-bot to post `noop` runs as a repository issue (you can still see them in its GitHub Actions run logs), add the following under `safe-outputs` in `.github/workflows/better-stale-bot.md`, then run `gh aw compile better-stale-bot`:

```yaml
safe-outputs:
  noop:
    report-as-issue: false
```

If no other issues in your repo still need that exemption, you can remove `agentic-workflows` from the exempt list in `## Configuration` inside `better-stale-bot.md`.

## Tips

- **Billing** — LLM usage is charged by your chosen engine (via `COPILOT_GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, or `OPENAI_API_KEY` in repo secrets). Cost scales with model, how many issues a run touches, and how much thread context is read. Numbers from quick test runs in this repo are not representative. To inspect a run: **Actions** → the workflow run → the agent summary log, and expand sections such as **agentic conversation** and **token usage**. From your repo clone you can also use `gh aw logs` and `gh aw audit`.

## Additional Resources

- [GitHub Agentic Workflows — Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)
- [Authoring Prompt (`create.md`, raw)](https://raw.githubusercontent.com/github/gh-aw/main/create.md) 

## License

Apache License 2.0 — see [LICENSE](LICENSE).
