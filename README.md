# better-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). On each run it finds inactive issues, summarizes the thread, applies a `Stale` label with a tailored comment, closes issues after a grace period when they stay quiet, and removes the label when someone participates again—all according to thresholds and exempt labels you set in the workflow.

Rules-based stale bots often use fixed timers and canned messages. Here the model reads each issue end to end, reasons about whether the thread still looks actionable, and drafts a short comment that reflects that state—so maintainers and readers get context, not just a countdown.

**What it does:**

- Identifies issues inactive for 60+ days (configurable)
- Ranks issues by engagement — low-activity issues are processed first
- Summarizes each issue and its discussion thread
- Determines if the issue appears resolved or still open
- Posts a helpful, context-aware stale comment
- Closes stale-labeled issues after a 7-day grace period (configurable)
- Removes the stale label when a **non-bot** user comments, giving the issue a second chance
- Respects exempt labels (`agentic-workflows`, `pinned`, `security`, `help wanted`)

## Installation

### Prerequisites

- **AI engine account** — one of:
  - [GitHub Copilot](https://github.com/features/copilot) (GitHub Agentic-Workflows default engine)
  - [Anthropic Claude](https://www.anthropic.com/)
  - [OpenAI Codex](https://openai.com/api/)
- **GitHub repository** with write access and [Actions enabled](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)
- **GitHub CLI** v2.0.0+ — [install here](https://cli.github.com/). Check version: `gh --version`
- **OS**: Linux, macOS, or Windows with WSL

### Step 1: Install the gh-aw extension

```bash
gh auth login
gh extension install github/gh-aw
```

### Step 2: Set up your AI engine secret

Go to your repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.

| Engine             | Secret name            | Get your key                                                                        |
| ------------------ | ---------------------- | ----------------------------------------------------------------------------------- |
| Copilot (default)  | `COPILOT_GITHUB_TOKEN` | [Auth setup](https://github.github.com/gh-aw/reference/auth/#copilot_github_token) |
| Claude             | `ANTHROPIC_API_KEY`    | [console.anthropic.com](https://console.anthropic.com)                              |
| Codex              | `OPENAI_API_KEY`       | [platform.openai.com](https://platform.openai.com/api-keys)                        |

### Step 3: Add the workflow

Copy the `better-stale-bot.md` file from this repo into your repository:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/better-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/.github/workflows/better-stale-bot.md
```

Compile the workflow to generate the lock file:

```bash
gh aw compile better-stale-bot
```

Commit and push both files:

```bash
git add .github/workflows/better-stale-bot.md .github/workflows/better-stale-bot.lock.yml
git commit -m "Add better-stale-bot workflow"
git push
```

### Step 4: Trigger a run

```bash
gh aw run better-stale-bot
```

Or go to the **Actions** tab in your repo and manually trigger the workflow.

## Customization

### Renaming the workflow

If you want a different workflow name (e.g., `issue-cleanup`):

1. Rename the file:
   ```bash
   mv .github/workflows/better-stale-bot.md .github/workflows/issue-cleanup.md
   ```

2. Recompile:
   ```bash
   gh aw compile issue-cleanup
   ```

3. Commit both the renamed `.md` and new `.lock.yml`
4. Delete the old `better-stale-bot.lock.yml` if it wasn't automatically removed

### Adjusting the prompt manually

Open `.github/workflows/better-stale-bot.md` in any editor. The file has two parts:

- **Frontmatter** (between `---` markers) — configuration: triggers, permissions, engine, safe outputs. Changes here **require recompilation** with `gh aw compile better-stale-bot`.
- **Markdown body** (everything after the frontmatter) — the AI agent's instructions. **You can edit this freely without recompilation.** Changes take effect on the next run.

Common customizations in the markdown body:

- **Thresholds** — change the days before stale (default 60) and days before close (default 7)
- **Exempt labels** — edit the list of labels that should never be marked stale
- **Comment tone** — adjust the stale comment guidelines to match your project's voice
- **Max issues per run** — change the limit (default 30)
- **Footer** — customize or remove the `— better-stale-bot` signature

### Adjusting the prompt with an AI agent

Instead of editing manually, use a coding agent (Copilot, Claude Code, Cursor, etc.) to customize the workflow. Run this prompt from your repository:

```
Update the stale bot workflow at .github/workflows/better-stale-bot.md using
https://raw.githubusercontent.com/github/gh-aw/main/create.md

I want to change:
- [describe your changes, e.g., "mark issues stale after 30 days instead of 60"]
- [e.g., "add a 'wontfix' exempt label"]
- [e.g., "make the stale comment more casual and friendly"]
```

The agent will read the gh-aw spec and update your workflow accordingly. If it modifies the frontmatter, remember to recompile.

### Changing the AI engine and model

Edit the `engine` field in the frontmatter (requires recompilation):

**Copilot (default — can omit `engine` entirely):**

```yaml
engine: copilot
```

**Claude with Haiku (recommended for cost efficiency):**

```yaml
engine:
  id: claude
  model: haiku
```

**Claude with Sonnet (higher quality, higher cost):**

```yaml
engine: claude
```

**Codex:**

```yaml
engine: codex
```

After changing engine config, recompile with `gh aw compile better-stale-bot` and push to remote before running the workflow. Ensure you have the correct repository secret for the chosen engine.

## Cost

Cost depends on engine, model, and number of issues processed. All costs are per workflow run.

| Engine + Model    | Issues | Cost   | Agent Duration | Workflow Duration |
| ----------------- | ------ | ------ | -------------- | ----------------- |
| Claude Haiku 4.5  | 5      | ~$0.10 | ~2 min         | ~4 min            |
| Claude Haiku 4.5  | 25     | ~$0.43 | ~4 min         | ~6 min            |
| Claude Sonnet 4.6 | 5      | ~$0.24 | ~2 min         | ~4 min            |

**Average cost per issue with Haiku: ~$0.02.** The table rows are **example runs** from tests (fixed issue counts); real spend depends on model, thread size, and how many issues a run actually touches.

## Tips

- **Protect system issues from stale labeling** — the default prompt exempts issues with the `agentic-workflows` label (auto-applied by GitHub Agentic Workflows to noop tracking issues). If your repo has other auto-generated or meta issues you want to protect, add their labels to the exempt list in the markdown body, or apply one of the existing exempt labels (`pinned`, `security`, `help wanted`).
- **The 30-issue limit is per run, not per day** — if you manually trigger the workflow multiple times in a day, each run can process up to 30 issues independently.
- **Step order in the prompt** — the markdown walks through **Bucket B** (rank and apply new `Stale` labels) in Steps 2–3, then **Bucket A** (close expired stale issues and remove `Stale` when someone commented) in Step 4. Safe-output `max: 30` caps writes per type; edit the prompt if you want a different ordering or a single combined budget across buckets.

## Additional Resources

- [GitHub Agentic Workflows — Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)

## License

MIT