# dosu-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). This is the open-source version of [Dosu's](https://dosu.dev) internal GitHub stale bot.

Unlike traditional stale bots that blindly close issues after a timeout, dosu-stale-bot uses AI to summarize each issue, determine whether it appears resolved, and post a contextual comment before applying the stale label.

**What it does:**

- Identifies issues inactive for 90+ days (configurable)
- Ranks issues by engagement — low-activity issues are processed first
- Summarizes each issue and its discussion thread
- Determines if the issue appears resolved or still open
- Posts a helpful, context-aware stale comment
- Closes stale-labeled issues after a 7-day grace period (configurable)
- Removes the stale label if someone comments, giving the issue a second chance
- Respects exempt labels (`pinned`, `security`, `help wanted`)

## Installation

### Prerequisites

- **AI engine account** — one of:
  - [GitHub Copilot](https://github.com/features/copilot) (default engine)
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

| Engine | Secret name | Get your key |
|--------|-------------|-------------|
| Copilot (default) | `COPILOT_GITHUB_TOKEN` | [Auth setup](https://github.github.com/gh-aw/reference/auth/#copilot_github_token) |
| Claude | `ANTHROPIC_API_KEY` | [console.anthropic.com](https://console.anthropic.com) |
| Codex | `OPENAI_API_KEY` | [platform.openai.com](https://platform.openai.com/api-keys) |

### Step 3: Add the workflow

Copy the `dosu-stale-bot.md` file from this repo into your repository:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/dosu-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/dosu-stale-bot/main/.github/workflows/dosu-stale-bot.md
```

Compile the workflow to generate the lock file:

```bash
gh aw compile dosu-stale-bot
```

Commit and push both files:

```bash
git add .github/workflows/dosu-stale-bot.md .github/workflows/dosu-stale-bot.lock.yml
git commit -m "Add dosu-stale-bot workflow"
git push
```

### Step 4: Trigger a run

```bash
gh aw run dosu-stale-bot
```

Or go to the **Actions** tab in your repo and manually trigger the workflow.

## Customization

### Renaming the workflow

If you want a different workflow name (e.g., `issue-cleanup`):

1. Rename the file:
   ```bash
   mv .github/workflows/dosu-stale-bot.md .github/workflows/issue-cleanup.md
   ```

2. Recompile:
   ```bash
   gh aw compile issue-cleanup
   ```

3. Commit both the renamed `.md` and new `.lock.yml`
4. Delete the old `dosu-stale-bot.lock.yml` if it wasn't automatically removed

### Adjusting the prompt manually

Open `.github/workflows/dosu-stale-bot.md` in any editor. The file has two parts:

- **Frontmatter** (between `---` markers) — configuration: triggers, permissions, engine, safe outputs. Changes here **require recompilation** with `gh aw compile dosu-stale-bot`.
- **Markdown body** (everything after the frontmatter) — the AI agent's instructions. **You can edit this freely without recompilation.** Changes take effect on the next run.

Common customizations in the markdown body:

- **Thresholds** — change the days before stale (default 90) and days before close (default 7)
- **Exempt labels** — edit the list of labels that should never be marked stale
- **Comment tone** — adjust the stale comment guidelines to match your project's voice
- **Max issues per run** — change the limit (default 25)
- **Footer** — customize or remove the `— dosu-stale-bot` signature

### Adjusting the prompt with an AI agent

Instead of editing manually, use a coding agent (Copilot, Claude Code, Cursor, etc.) to customize the workflow. Run this prompt from your repository:

```
Update the stale bot workflow at .github/workflows/dosu-stale-bot.md using
https://raw.githubusercontent.com/github/gh-aw/main/create.md

I want to change:
- [describe your changes, e.g., "mark issues stale after 30 days instead of 90"]
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

**Claude (recommended: Haiku for cost efficiency):**

```yaml
engine:
  id: claude
  model: haiku
```

**Claude with Sonnet (default sonnet: higher quality, higher cost):**

```yaml
engine: claude
```

**Codex:**

```yaml
engine: codex
```

After changing engine config, recompile: `gh aw compile dosu-stale-bot` and push to remote before running the workflow

## Cost

Cost depends on engine, model, and number of issues processed. Baseline for processing 5 issues:

| Engine + Model | Cost | Duration |
|---------------|------|----------|
| Claude Haiku 4.5 | ~$0.10 | ~4 min |
| Claude Sonnet 4.6 | ~$0.24 | ~4 min |

## Additional Resources

- [GitHub Agentic Workflows — Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)

## License

MIT
