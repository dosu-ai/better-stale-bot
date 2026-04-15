# better-stale-bot

An AI-powered stale issue bot built with [GitHub Agentic Workflows](https://github.github.com/gh-aw/). It summarizes inactive issues, applies a `Stale` label, posts a tailored comment, closes them after a quiet period, and removes `Stale` when a **non-bot** user engages again. 

Policy lives in `## Configuration` in `.github/workflows/better-stale-bot.md` (`days-before-stale`, `days-before-close`, exempt labels; defaults **60** / **7** days).

Unlike timer-only bots, the model reads each thread end to end and drafts issue-specific comments.

**What it does**

- Rank and process low-engagement issues first  
- Summarize the thread; note resolved vs open  
- Apply `Stale` label, comment, close after `days-before-close`, remove `Stale` label on non-bot activity  
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

## Migrating from Dosu's stale bot

- Turn off the previous automation (Dosu's deployment stale bot and/or workflows like `actions/stale`) so two bots do not compete
- Follow [Installation](#installation)
- Map old settings to this repo:
  - Thresholds → `## Configuration` defaults for `days-before-stale` / `days-before-close`
  - Volume → frontmatter `safe-outputs` `max:` and Step 2 “top N”, then `gh aw compile`
  - Exempt labels → only under `## Configuration`; keep Guidelines consistent (label **names** on GitHub; Dosu may have used IDs)
  - Shipped defaults here: **60 / 7 / 30** vs common Dosu-style **90 / 7 / 25** — edit the table and frontmatter to match your policy
- **Codex (GPT-style models):** `engine: codex`, optional `model:`, add `OPENAI_API_KEY`, recompile — see [engines](https://github.github.com/gh-aw/reference/engines/)

## Customization

- **Rename workflow:** `mv .github/workflows/better-stale-bot.md …` then `gh aw compile <name>`; drop the old `.lock.yml` if it remains
- **Frontmatter** (`---` … `---`): engine, `safe-outputs` caps, etc. — **recompile** after changes
- **Markdown body:** takes effect on the next run — **no recompile** unless frontmatter changed
- **`## Configuration`:** edit timing (parameter defaults) and exempt labels; keep Guidelines aligned (Bucket B defers to Configuration)
- **Per-run cap:** set every `safe-outputs` `max:` and Step 2 “top N” to the same budget, then recompile
- **Edit with an agent:** [create.md](https://raw.githubusercontent.com/github/gh-aw/main/create.md)
- **Engines** (frontmatter, then recompile): e.g. `engine: copilot`; `engine: { id: claude, model: haiku }`; `engine: claude` (Sonnet); `engine: codex` (+ optional `model:`) — match the repo secret to the engine

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
