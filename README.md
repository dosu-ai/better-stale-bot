# better-stale-bot

Better Stale Bot is a [GitHub Agentic Workflows](https://github.github.com/gh-aw/) workflow for maintainers with a large issue backlog. It summarizes inactive issues, applies a `Stale` label, posts a tailored comment, closes them after a quiet period, and removes `Stale` when a **non-bot** user engages again.

Policy lives in `## Configuration` in the workflow markdown at `workflows/better-stale-bot.md`. The defaults are 60 days before stale, 7 days before close, and a 100 issue detailed scan window. Write caps live in the YAML frontmatter under `safe-outputs`. Recompile after a frontmatter change.

The model reads each selected thread and drafts an issue-specific comment.

> [!IMPORTANT]
> GitHub Agentic Workflows is in public preview. Review the editable `.md` source and compiled `.lock.yml`, test in a sandbox repository, and start with conservative limits before enabling this on a repository with real issues.

**What it does**

- Inspect the 100 oldest eligible candidates, then process low-engagement issues first
- Summarize the thread and note whether it looks resolved
- Apply `Stale` label, comment, close after `days-before-close`, remove `Stale` label on non-bot activity
- Remove `Stale` instead of closing when an exempt label is present (`agentic-workflows`, `pinned`, `security`, `help wanted` by default)
- Bound each write type with compiled `safe-outputs`
- Comments in the issue title's language through prompting
- Keep advisory run context in cache-memory while treating current GitHub state as authoritative

## Table of contents

- [Repository layout](#repository-layout)
- [Installation](#installation)
- [Before the first run](#before-the-first-run)
- [Testing and validation](#testing-and-validation)
- [Migrating from GitHub's stale bot](#migrating-from-githubs-stale-bot)
- [Migrating from Dosu's stale bot](#migrating-from-dosus-stale-bot)
- [Customization](#customization)
- [No-op posted as an issue](#no-op-posted-as-an-issue)
- [Operations and troubleshooting](#operations-and-troubleshooting)
- [Contributing](#contributing)
- [Audit](#audit)
- [Additional Resources](#additional-resources)
- [License](#license)

## Repository layout

This repository keeps two copies of the Agentic Workflow markdown.

| Location | Role |
| --- | --- |
| `workflows/` | **Distribution source**. The template installed by `gh aw add`. Edit this copy first. |
| `.github/workflows/better-stale-bot.md` | **Repository test copy**. Keep it identical to the distribution source. |
| `.github/workflows/better-stale-bot.lock.yml` | **Generated workflow**. Compile it with `gh aw` and never edit it by hand. |
| `.github/aw/actions-lock.json` | **Generated action pins**. Commit compiler updates. |

The two Markdown workflow files must remain byte-for-byte identical. See [CONTRIBUTING.md](CONTRIBUTING.md) for the synchronization and validation commands.

## Installation

**Prerequisites**

- [GitHub Actions](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository) enabled
- [GitHub CLI](https://cli.github.com/)
- An AI account from [Copilot](https://github.com/features/copilot), [Claude](https://www.anthropic.com/), or [Codex](https://openai.com/api/)
- An existing `Stale` label in the target repository. The `add-labels` tool requires it.

The current repository checks use GitHub CLI 2.96.0 and `gh-aw` v0.83.4.

Authenticate the GitHub CLI with repository and workflow access.

```bash
gh auth login --scopes repo,workflow
gh extension install github/gh-aw
```

See the [authentication reference](https://github.github.com/gh-aw/reference/auth/). The wizard in Option A can help configure a secret. Option B assumes you configure authentication yourself.

| Engine | Authentication |
| --- | --- |
| Claude | Repository or organization secret `ANTHROPIC_API_KEY` |
| Codex | Repository or organization secret `OPENAI_API_KEY` |
| Copilot | Prefer organization billing with `permissions.copilot-requests: write`. Otherwise use a fine-grained PAT in `COPILOT_GITHUB_TOKEN`. |

The workflow defaults to Claude Haiku with `engine: claude` and top-level `model: haiku`. Use `ANTHROPIC_API_KEY` unless you change the engine and recompile. The template runs daily. Change the schedule before the first run if needed.

Issue titles, bodies, comments, timeline data, and reaction data can be sent to the selected model provider. Review provider terms and repository data policy before enabling the workflow.

### Option A using the setup wizard

Run from a clone of the repository where you want the bot (you need write access so the wizard can open a PR if needed).

```bash
# 1. Authenticate with GitHub
gh auth login

# 2. Add the workflow (configures auth, adds files, and can open a PR)
# Add --no-secret if authentication is already configured at the repo or org level.
gh aw add-wizard dosu-ai/better-stale-bot/better-stale-bot

# 3. Pull locally if the wizard created and merged a PR
git pull
```

On a repository that has not used `gh-aw` before, initialization also adds authoring support files such as `.github/agents/agentic-workflows.md`, `.github/skills/agentic-workflows/SKILL.md`, `.github/mcp.json`, and `.github/workflows/copilot-setup-steps.yml`. Review and commit those generated files with the workflow.

### Option B using manual setup

In a local clone of the target repository, download the [distribution markdown](https://github.com/dosu-ai/better-stale-bot/blob/main/workflows/better-stale-bot.md), compile it, review it, and push it.

```bash
# 1. Add your engine secret in repo Settings → Secrets and variables → Actions
# ANTHROPIC_API_KEY (Claude), COPILOT_GITHUB_TOKEN (Copilot), or OPENAI_API_KEY (Codex)

# 2. Ensure the required label exists
gh label create Stale --color BFD4F2 \
  --description "Inactive issue awaiting confirmation" --force

# 3. Download and compile
mkdir -p .github/workflows
curl -fsSL -o .github/workflows/better-stale-bot.md \
  https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/workflows/better-stale-bot.md
gh aw compile better-stale-bot

# 4. Validate, review, and push
gh aw validate better-stale-bot --strict
git add .github/ .gitattributes
git commit -m "Add better-stale-bot workflow"
git push
```

## Before the first run

The default policy can post up to 30 stale comments, add 30 labels, remove 30 labels, and close 30 issues in a single run. These are separate caps, not a combined 30-issue limit.

Before enabling or dispatching the workflow

1. Review every open issue that is already older than `days-before-stale`.
2. Confirm the exact exempt-label names and create the `Stale` label.
3. Confirm the engine authentication with `gh aw secrets bootstrap --non-interactive`.
4. Lower every write cap to `1` for the first controlled run, then recompile.
5. Review the compiled `.lock.yml`, especially permissions and safe-output settings.
6. Run in a sandbox repository first. Restore the intended caps only after reviewing the comments, labels, and closure behavior.

Dispatch with `gh aw run better-stale-bot`, or use the **Actions** tab. `gh aw run --dry-run` only previews dispatch configuration. It does not execute the model or prove behavior.

## Testing and validation

For a no-write trial, use `gh aw trial`. It captures safe outputs instead of applying them.

```bash
gh aw trial dosu-ai/better-stale-bot/better-stale-bot \
  --logical-repo OWNER/REPOSITORY \
  --delete-host-repo-after
```

Trial execution still needs valid engine authentication. For a full behavioral test, use a private sandbox with only purpose-built issues and cover the matrix in [CONTRIBUTING.md](CONTRIBUTING.md#behavioral-test-matrix).

Validate local changes with the commands below.

```bash
diff -u workflows/better-stale-bot.md .github/workflows/better-stale-bot.md
gh aw validate better-stale-bot --strict
gh aw validate --dir workflows --strict
gh aw compile better-stale-bot
```

## Migrating from GitHub's stale bot

If you use [actions/stale](https://github.com/actions/stale) (or another workflow that wraps it), disable or remove the stale workflow and any schedule so two bots do not compete on items.

Then follow [Installation](#installation). Inactivity, the close delay, the candidate scan limit, and exempt labels live under `## Configuration`. Write caps live in frontmatter.

**Issues only by policy**

- Frontmatter grants issue read access.
- Comment writes opt out of pull requests and discussions.
- The prompt only lists, ranks, and acts on issues.

The v0.83.4 compiler still grants `pull-requests: write` in the generated lock because the shared label handlers support issues and pull requests. The policy does not ask the agent to act on pull requests. Review this framework permission before deployment.

Supporting pull requests requires a separate policy review.

- Widen YAML frontmatter `permissions` (for example `pull-requests`)
- Grant the GitHub `tools` the agent will need for PRs
- Add or adjust `safe-outputs` for any new write types (comments or state changes on PRs)
- Update the markdown instructions so the agent lists, ranks, and mutates PRs instead of (or in addition to) issues

Run `gh aw compile` after frontmatter edits

## Migrating from Dosu's stale bot

To hand this work to a coding agent, paste the prompt below. Replace the example values first.

```
Migrate this repository from Dosu's hosted stale bot to better-stale-bot by following
https://raw.githubusercontent.com/dosu-ai/better-stale-bot/main/dosu_migration.md

Carry over these values.
- days_before_stale equals 90
- days_before_close equals 7
- operations_per_run equals 25
- exempt_labels contains `agentic-workflows`

Use the Claude engine. Install non-interactively, map my settings, then stop before
committing so I can review. Remind me to add the ANTHROPIC_API_KEY secret and turn
off the Dosu stale bot.
```

To migrate by hand, follow the steps below instead.

- Turn off the previous automation (Dosu's deployment stale bot and/or workflows like `actions/stale`) so two bots do not compete on items.
- Follow [Installation](#installation)
- Map old settings using `## Configuration` and frontmatter in `better-stale-bot.md`.
  - Edit thresholds, the scan limit, and exempt labels in the Configuration table.
  - Treat the old combined operation cap and the new per-type caps as different controls. Start each new cap at `1`, test the workflow, then raise them deliberately.

| Setting | Dosu stale bot defaults | better-stale-bot defaults |
| --- | --- | --- |
| `days-before-stale` | 90 | 60 |
| `days-before-close` | 7 | 7 |
| Write caps | 25 combined operations | 30 for each write type |
| Exempt labels | none | `agentic-workflows`, `pinned`, `security`, `help wanted` |

- Codex uses `engine: codex` with an optional top-level `model`. Add `OPENAI_API_KEY` and recompile. See [engines](https://github.github.com/gh-aw/reference/engines/).

## Customization

- Rename the workflow by renaming the Markdown file and running `gh aw compile <name>`. Remove the old generated lock file after review.
- Frontmatter controls the engine, top-level model, schedule, tools, and write caps. Recompile after any frontmatter change.
- The Markdown body takes effect on the next run. It controls thresholds, the scan limit, exempt labels, and behavior.
- Imported workflows keep a `source` field. Use `gh aw update better-stale-bot`, review the diff, and recompile to pull upstream changes.

## No-op posted as an issue

The default exempt list includes `agentic-workflows` because Agentic Workflows can open a repository issue such as `[aw] No-Op Runs` to track `noop` runs (when the agent reports that there is nothing to do). That issue is labeled `agentic-workflows`, and without an exemption this stale bot could keep summarizing, labeling, or closing it even though Agentic Workflows created it.

If you do not want a daily no-op comment, add the setting below and run `gh aw compile better-stale-bot`.

```yaml
safe-outputs:
  noop:
    report-as-issue: false
```

Keep `agentic-workflows` exempt while any framework issue reporting is enabled. Disabling no-op issue reporting alone does not disable missing-data, threat-detection, or failure reports. Remove the exemption only after confirming that no framework report can create an issue.

## Operations and troubleshooting

- Billing comes from the selected AI engine. Cost grows with the scan limit and thread size.
- Use `gh aw status`, `gh aw logs`, `gh aw audit`, and `gh aw outcomes` when reviewing a run.
- The scan limit bounds work on large repositories. Old issues outside the window roll into later runs.
- A recent bot update can delay an issue because the first pass uses GitHub's `updated` filter. The detailed check is conservative and will not mark a borderline issue stale.
- Safe-output caps are independent. They do not form one total operation cap.
- Separate comment and label writes are not atomic. The prompt checks current state before each write and avoids duplicate stale comments after a partial run.
- To stop the bot, disable `better-stale-bot.lock.yml` in Actions. Revert the install commit if you also want to remove it.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before changing the workflow. Pull requests must keep both source copies in sync, pass current `gh aw` validation, and include the generated lock changes.

## Audit

Read [AUDIT.md](AUDIT.md) for the adversarial review, external research, target repository test, fixed findings, and open work.

## Additional Resources

- [GitHub Agentic Workflows Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Creating Workflows with AI Agents](https://github.github.com/gh-aw/setup/creating-workflows/)
- [CLI Commands](https://github.github.com/gh-aw/setup/cli/)
- [Engine & Auth Setup](https://github.github.com/gh-aw/reference/engines/)
- [Safe Outputs Reference](https://github.github.com/gh-aw/reference/safe-outputs/)
- [Authoring Prompt (`create.md`, raw)](https://raw.githubusercontent.com/github/gh-aw/main/create.md)

## License

Apache License 2.0. See [LICENSE](LICENSE).
