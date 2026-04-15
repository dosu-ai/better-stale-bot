# better-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). It summarizes inactive issues, applies a `Stale` label with a tailored comment, closes them after a quiet period, and removes the label when a **non-bot** user engages again. Policy lives in **`## Configuration`** in `.github/workflows/better-stale-bot.md`: **`days-before-stale`**, **`days-before-close`**, and exempt labels (defaults 60 / 7 days).

Unlike timer-only bots, the model reads each thread and drafts issue-specific comments.

**What it does**

- Rank and process low-engagement issues first  
- Summarize the thread; note resolved vs open  
- Apply `Stale`, comment, close after **`days-before-close`**, remove **Stale** on non-bot activity  
- Respect exempt labels (`agentic-workflows`, `pinned`, `security`, `help wanted` by default)  
- Cap volume per run (`safe-outputs` + Step 2 “top N”, default 30)

## Table of contents

- [Installation](#installation)
- [Migrating from Dosu's stale bot](#migrating-from-dosus-stale-bot)
- [Customization](#customization)
- [Tips](#tips)
- [Additional Resources](#additional-resources)
- [License](#license)

## Installation

**Prerequisites:** [GitHub Actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository) enabled, [GitHub CLI](https://cli.github.com/) 2.0+, and an AI engine account ([Copilot](https://github.com/features/copilot), [Claude](https://www.anthropic.com/), or [Codex](https://openai.com/api/)).

```bash
gh auth login
gh extension install github/gh-aw
```

Add a repository secret under **Settings → Secrets and variables → Actions** (see [engine auth](https://github.github.com/gh-aw/reference/auth/)):

| Engine   | Secret              |
| -------- | ------------------- |
| Copilot  | `COPILOT_GITHUB_TOKEN` |
| Claude   | `ANTHROPIC_API_KEY` |
| Codex    | `OPENAI_API_KEY`    |

Add the workflow and compile:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/better-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/.github/workflows/better-stale-bot.md
gh aw compile better-stale-bot
git add .github/workflows/better-stale-bot.md .github/workflows/better-stale-bot.lock.yml
git commit -m "Add better-stale-bot workflow"
git push
```

Run: `gh aw run better-stale-bot` or use the **Actions** tab.

## Migrating from Dosu's stale bot

1. **Disable** the old automation (Dosu's deployment stale bot and/or other GitHub workflows like `actions/stale`) so two bots do not compete.
2. Follow **[Installation](#installation)**.
3. **Map settings:** thresholds → **`## Configuration`** defaults for **`days-before-stale`** / **`days-before-close`**; volume → frontmatter **`safe-outputs`** `max:` + Step 2 “top N” (then `gh aw compile`); exempt labels → listed under **Configuration** only—keep **Guidelines** consistent (use **label names** on GitHub; Dosu may have used IDs). Shipped defaults are roughly **60 / 7 / 30** vs common Dosu-style **90 / 7 / 25**—adjust the table and frontmatter to match your policy.

**Codex (GPT-style models):** set `engine: codex` (+ optional `model:`) in frontmatter, add `OPENAI_API_KEY`, recompile — see [engines](https://github.github.com/gh-aw/reference/engines/).

## Customization

- **Rename workflow:** `mv .github/workflows/better-stale-bot.md …` then `gh aw compile <name>`; remove the old `.lock.yml` if needed.
- **Frontmatter** (`---` … `---`): engine, `safe-outputs` caps, etc. — **recompile** after edits.
- **Markdown body:** agent instructions — edits apply on the next run; **no recompile** unless frontmatter changed.

Edit **`## Configuration`** for timing (parameter table defaults) and exempt labels; keep **Guidelines** consistent with that list (Bucket B defers to Configuration). Tune tone and footer in Step 3. To change per-run volume, update every `safe-outputs` `max:` and Step 2’s “top N”, then recompile.

Use an AI agent with [create.md](https://raw.githubusercontent.com/github/gh-aw/main/create.md) if you prefer not to edit by hand.

**Engines** (frontmatter, then recompile): e.g. `engine: copilot`; `engine: { id: claude, model: haiku }`; `engine: claude` (Sonnet); `engine: codex` (+ optional `model:`). Match the secret to the engine.

## Tips

- **Billing** — costs follow your engine’s pricing and scale with how many issues you touch and how long threads are; test runs in this repo are not representative since thread sizes can vary.
- Exempt meta issues (e.g. `agentic-workflows`, `pinned`, `security`, `help wanted`) via **`## Configuration`**.
- The **30-issue** cap is per workflow run (each `safe-outputs` type + Step 2 budget).
- Steps 2–3 handle new **Stale** labels; Step 4 closes expired stale issues / removes **Stale** on non-bot activity.

## Additional Resources

- [GitHub Agentic Workflows — Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)

## License

MIT
