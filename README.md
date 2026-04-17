# better-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). It summarizes inactive issues, applies a `Stale` label, posts a tailored comment, closes them after a quiet period, and removes `Stale` when a **non-bot** user engages again. 

Policy lives in `## Configuration` in `.github/workflows/better-stale-bot.md`: `days-before-stale`, `days-before-close`, and exempt labels (defaults **60** / **7** days). Per-run write caps are only in YAML `safe-outputs` → `max:` (frontmatter); recompile after changing them.

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

- [Installation](#installation)
- [Migrating from GitHub's stale bot](#migrating-from-githubs-stale-bot)
- [Migrating from Dosu's stale bot](#migrating-from-dosus-stale-bot)
- [Customization](#customization)
- [No-op run issues and the `agentic-workflows` exempt label](#no-op-run-issues-and-the-agentic-workflows-exempt-label)
- [Tips](#tips)
- [Additional Resources](#additional-resources)
- [License](#license)

## Installation

**Prerequisites**

- [GitHub Actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository) enabled
- [GitHub CLI](https://cli.github.com/) 2.0+
- AI account: [Copilot](https://github.com/features/copilot), [Claude](https://www.anthropic.com/), or [Codex](https://openai.com/api/)

**Install `gh-aw`**

```bash
gh auth login
gh extension install github/gh-aw
```

**Repository secret** ([engine auth](https://github.github.com/gh-aw/reference/auth/)) — repo **Settings → Secrets and variables → Actions**

| Engine  | Secret               |
| ------- | -------------------- |
| Copilot | `COPILOT_GITHUB_TOKEN` |
| Claude  | `ANTHROPIC_API_KEY`  |
| Codex   | `OPENAI_API_KEY`     |

The workflow you copy from this repo defaults to **Claude Haiku** in YAML frontmatter (`engine: id: claude, model: haiku`). Use `ANTHROPIC_API_KEY` unless you change the engine and recompile. The template also schedules a **daily** run (`on: schedule: daily`); adjust in frontmatter if you want a different frequency.

**Add workflow, compile, push**

```bash
mkdir -p .github/workflows
curl -o .github/workflows/better-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/.github/workflows/better-stale-bot.md
gh aw compile better-stale-bot
git add .github/workflows/better-stale-bot.md .github/workflows/better-stale-bot.lock.yml
git commit -m "Add better-stale-bot workflow"
git push
```

**Run**

- `gh aw run better-stale-bot`, or open the **Actions** tab and run the workflow

## Migrating from GitHub's stale bot

If you use [actions/stale](https://github.com/actions/stale) (or another workflow that wraps it), disable or remove the stale workflow and any schedule so two bots do not compete on items.

Then follow [Installation](#installation). Conceptually, most settings line up with defaults here: inactivity before marking stale, quiet period before closing, exempt labels, and a `Stale`-style label are all driven from `## Configuration` in `better-stale-bot.md` (defaults **60** / **7** days and the exempt list in that file). Adjust that section and frontmatter `safe-outputs` → `max:` to match how many writes you want per run.

**Issues only by default.** The shipped workflow scopes GitHub access to **issues** (frontmatter `permissions` and `tools.github` / `toolsets`), uses issue-only `safe-outputs` (for example `close-issue`), and the prompt is written for issues. To extend behavior to **pull requests**, you need to change the workflow yourself: widen YAML `permissions` (for example `pull-requests`), grant the GitHub `tools` your agent will need for PRs, add or adjust `safe-outputs` for any new write types (comments or state changes on PRs), and update the **markdown instructions** so the agent lists, ranks, and mutates PRs instead of (or in addition to) issues. Recompile with `gh aw compile` after frontmatter edits.

## Migrating from Dosu's stale bot

- Turn off the previous automation (Dosu's deployment stale bot and/or workflows like `actions/stale`) so two bots do not compete on items.
- Follow [Installation](#installation)
- Map old settings to this repo:
  - Thresholds → `## Configuration` defaults for `days-before-stale` / `days-before-close`
  - Per-run write limits → `safe-outputs` → `max:` in frontmatter, then `gh aw compile` (each output type capped separately; see [Safe Outputs](https://github.github.com/gh-aw/reference/safe-outputs/))
  - Exempt labels → only under `## Configuration`; keep Guidelines consistent (label **names** on GitHub; Dosu may have used IDs)
  - Shipped defaults here: **60 / 7** days vs common Dosu-style **90 / 7** — edit the Configuration table; adjust `safe-outputs` `max:` if you need different volume
- **Codex (GPT-style models):** `engine: codex`, optional `model:`, add `OPENAI_API_KEY`, recompile — see [engines](https://github.github.com/gh-aw/reference/engines/)

## Customization

- **Rename workflow:** rename the markdown file in `.github/workflows/`, then `gh aw compile <name>`, drop the old `.lock.yml` if it remains
- **Frontmatter**: modify AI engine and model choice, `safe-outputs` → `max:` (per output type), schedule frequency. **recompile** with `gh aw compile` after changes
  - **AI Engines:** e.g. `engine: copilot`; `engine: { id: claude, model: haiku }`; `engine: claude` (Sonnet); `engine: codex` (+ optional `model:`) — match the repo secret to the engine 
- **Markdown body:** takes effect on the next run — no recompile needed unless frontmatter changed
  - `## Configuration`: edit `days-before-stale`, `days-before-close`, and exempt labels; keep Guidelines aligned
- **Edit with a coding agent:** tell your agent to reference https://raw.githubusercontent.com/github/gh-aw/main/create.md prompt as the base spec so the agent follows gh-aw workflows formatting and safety standards (YAML frontmatter + instruction body), say what you want changed, and reference `better-stale-bot.md` as the file to update. See more in [Additional Resources](#additional-resources)

## No-op posted as an issue

The default exempt list includes `agentic-workflows` because Agentic Workflows can open a repository issue such as `[aw] No-Op Runs` to track `noop` runs (when the agent reports that there is nothing to do). That issue is labeled `agentic-workflows`, and without an exemption this stale bot could keep summarizing, labeling, or closing it even though Agentic Workflows created it.

If you don't need `noop` runs posted as a repository issue (you can still see them in GitHub Actions run logs), turn off noop issue reporting in that workflow's YAML frontmatter, then recompile:

```yaml
safe-outputs:
  noop:
    report-as-issue: false
```

If no other issues in your repo still need that exemption, you can remove `agentic-workflows` from the exempt list in `## Configuration` inside `better-stale-bot.md`.

## Tips

- **Billing** — LLM usage is charged by your chosen engine (via `COPILOT_GITHUB_TOKEN`, `ANTHROPIC_API_KEY`, or `OPENAI_API_KEY` in repo secrets). Cost scales with model, how many issues a run touches, and how much thread context is read. Numbers from quick test runs in this repo are not representative.

## Additional Resources

- [GitHub Agentic Workflows — Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)

## License

MIT
