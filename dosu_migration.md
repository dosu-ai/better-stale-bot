# Dosu Stale Bot → better-stale-bot Migration Guide

A prompt for a coding agent (Claude Code, Copilot, Codex, or similar with shell + `gh`
CLI access) to migrate a repository off Dosu's hosted stale bot onto
[better-stale-bot](https://github.com/dosu-ai/better-stale-bot), the open-source AI stale
issue bot built on [GitHub Agentic Workflows](https://github.github.com/gh-aw/).

A human can also follow these steps directly.

---

## Context

Dosu's hosted stale bot is being removed from the Dosu app on **August 1, 2026**. The
replacement is **better-stale-bot**: it reads each issue thread end to end, summarizes
status, applies a `Stale` label with a tailored comment, closes issues after a grace
period, and removes `Stale` when a non-bot user re-engages. It runs as a scheduled GitHub
Actions workflow in the target repository — the customer owns and hosts it.

Your job is to install it and carry the customer's existing Dosu settings across so
behavior does not silently change.

## Inputs

Collect (or ask the user for) the following. Every field is optional — fall back to the
Dosu default when a value is unknown.

- `repo` — target repository in `owner/repo` form
- `dosu_config` — the user's current Dosu stale bot settings:
  - `days_before_stale` (Dosu default: **90**)
  - `days_before_close` (Dosu default: **7**)
  - `operations_per_run` (Dosu default: **25**)
  - `exempt_labels` (Dosu default: none)
- `ai_engine` — `claude`, `copilot`, or `codex` (default: `claude`, model Haiku)

## Configuration mapping (read this before editing anything)

better-stale-bot's defaults are **not** the same as Dosu's. If you install and leave the
defaults, a customer on Dosu's 90-day policy silently drops to 60 days. Map explicitly:

| Setting | Dosu default | better-stale-bot default | Where it lives |
| --- | --- | --- | --- |
| `days-before-stale` | 90 | 60 | Markdown body → `## Configuration` |
| `days-before-close` | 7 | 7 | Markdown body → `## Configuration` |
| Per-run caps | 25 (one combined cap) | 30 (per output type) | YAML frontmatter → `safe-outputs` |
| Exempt labels | none | `agentic-workflows`, `pinned`, `security`, `help wanted` | Markdown body → `## Configuration` |

Two things to keep straight, because they decide whether you must recompile:

- **Markdown body** (`## Configuration`: thresholds, exempt labels) is imported at
  runtime. Edits take effect on the next scheduled run with **no recompile**.
- **YAML frontmatter** (`safe-outputs` caps, `engine`, schedule) is compiled into the
  `.lock.yml`. Any edit here **requires `gh aw compile`**.

Note the cap model differs: Dosu used one `operations_per_run` number; better-stale-bot
caps each output type independently (e.g. 30 comments *and* 30 label adds *and* 30 closes
per run). When mapping `operations_per_run`, set each relevant `max:` to that value.

---

## Migration steps

### Step 1 — Validate prerequisites

```bash
gh auth status                       # must be authenticated with write access to `repo`
gh extension install github/gh-aw    # no-op if already installed
gh extension upgrade github/gh-aw    # ensure it's current
gh repo view {repo}                  # confirms the repo exists and is reachable
```

Also confirm GitHub Actions is enabled for the repo (Settings → Actions → General). If
it's disabled the workflow will never run.

### Step 2 — Install better-stale-bot

From a local clone of the target repo, use the **non-interactive** command (best for an
autonomous agent — it downloads the distribution markdown and compiles the lock file
without prompting):

```bash
gh aw add dosu-ai/better-stale-bot/better-stale-bot
```

This creates:
- `.github/workflows/better-stale-bot.md` — editable source (frontmatter + instructions)
- `.github/workflows/better-stale-bot.lock.yml` — the compiled workflow that actually runs
- `.gitattributes` — marks the `.lock.yml` as generated

> **Human alternative:** `gh aw add-wizard dosu-ai/better-stale-bot/better-stale-bot`
> is interactive — it prompts for the engine secret and can open a PR. Do **not** use the
> wizard from an unattended agent; it will hang waiting for input. Use `gh aw add` above.

### Step 3 — Map Dosu configuration

Open `.github/workflows/better-stale-bot.md`.

**A. Thresholds and exempt labels — markdown body, `## Configuration`:**

- Set `days-before-stale` default to `{dosu_config.days_before_stale}` (preserves the
  customer's 90-day policy instead of dropping to 60).
- Set `days-before-close` default to `{dosu_config.days_before_close}`.
- **Exempt labels — replicate Dosu, but always include `agentic-workflows`:**
  - If the user supplied `exempt_labels`: set the exempt list to **exactly those labels
    plus `agentic-workflows`**. Do not carry over the other better-stale-bot defaults
    (`pinned`, `security`, `help wanted`) — the customer didn't have them under Dosu.
  - If the user supplied **no** exempt labels: keep all four better-stale-bot defaults
    (`agentic-workflows`, `pinned`, `security`, `help wanted`).
  - Use label names exactly as they appear on GitHub.
- **Why `agentic-workflows` is non-negotiable:** GitHub Agentic Workflows opens a
  repository issue (`[aw] No-Op Runs`) labeled `agentic-workflows` to track `noop` runs.
  If it isn't exempt, the bot will summarize, label, and eventually close its own no-op
  issue. Only drop it if you also disable no-op-as-issue by adding `report-as-issue: false`
  under `safe-outputs.noop` in the frontmatter (then recompile) — see the repo's
  "No-op posted as an issue" section.

**B. Per-run caps — YAML frontmatter, `safe-outputs`:**

Only edit if `operations_per_run` differs from 30. Set each output type's `max:` to the
mapped value. The correct key is `allowed:` (not `allowed-labels:` — that key does not
exist and will fail compilation):

```yaml
safe-outputs:
  add-comment:
    max: {operations_per_run}
  add-labels:
    max: {operations_per_run}
    allowed: ["Stale"]
  remove-labels:
    max: {operations_per_run}
    allowed: ["Stale"]
  close-issue:
    max: {operations_per_run}
  noop:
```

**C. Engine — YAML frontmatter, `engine:` (only if `ai_engine` is not `claude`):**

```yaml
# Claude (default) — Haiku:
engine:
  id: claude
  model: haiku

# Copilot:
engine: copilot

# Codex:
engine: codex
```

### Step 4 — Recompile (only if you touched frontmatter)

If you changed **anything in the YAML frontmatter** in Step 3 (caps or engine), recompile:

```bash
gh aw compile better-stale-bot
```

If you changed **only** the markdown body (thresholds, exempt labels), skip this — the
existing `.lock.yml` imports the body at runtime.

### Step 5 — Configure the engine secret

The secret cannot be set by the agent's own credentials — it must be the customer's key.
Add it in the repo (Settings → Secrets and variables → Actions → New repository secret):

| Engine | Required secret |
| --- | --- |
| `claude` | `ANTHROPIC_API_KEY` |
| `copilot` | `COPILOT_GITHUB_TOKEN` |
| `codex` | `OPENAI_API_KEY` |

You may run `gh secret set {SECRET_NAME} --repo {repo}` to set it interactively, but **do
not paste, echo, or hardcode the key value** — let the user supply it at the prompt or via
the GitHub UI.

### Step 6 — Commit and push

```bash
git add .github/ .gitattributes
git commit -m "chore: migrate from Dosu stale bot to better-stale-bot"
git push
```

Prefer a branch + PR if the repo protects `main`. Do not commit unrelated untracked files.

### Step 7 — Verify

`gh aw compile` adds a `workflow_dispatch` trigger to the lock file, so you can trigger a
manual run immediately (no need to wait for the daily schedule):

```bash
gh aw run better-stale-bot                    # manual dispatch
gh run list --workflow=better-stale-bot.lock.yml --limit=1   # check status
```

Confirm the run succeeds in the **Actions** tab. After this the bot runs daily on schedule
with no further action. If a run fails, inspect it with `gh aw logs` / `gh aw audit` and
recheck the secret.

### Step 8 — Turn off the Dosu stale bot

Tell the user to disable Dosu's hosted stale bot in their Dosu app settings so the two
bots don't act on the same issues. It will be removed automatically on August 1, 2026
regardless, but disabling it now avoids double-processing during the overlap.

---

## Error handling

- `gh aw` command fails → `gh extension upgrade github/gh-aw`, then retry.
- Compile fails → validate YAML frontmatter syntax; the most common mistake is using
  `allowed-labels:` instead of `allowed:`, or bad indentation under `safe-outputs`.
- Run fails immediately → the engine secret is usually missing or misnamed; confirm it
  matches the table in Step 5 for the selected engine.
- No `workflow_dispatch` option in the Actions tab → the `.lock.yml` wasn't committed, or
  compile didn't run; re-check Steps 4 and 6.

## Notes

- Config in the **markdown body** takes effect on the next run with no recompile; **YAML
  frontmatter** changes require `gh aw compile`.
- Caps are per output type, not one combined counter like Dosu's `operations_per_run`.
- Between runs the bot writes a short summary to cache-memory and reads it next run to
  avoid reprocessing the same issues.
- Comments are generated in the language of the issue title.
- The shipped workflow targets **issues only**, not pull requests.

## Output — summarize when done

1. Repository migrated
2. Config applied — `days-before-stale`, `days-before-close`, per-run caps, exempt labels
   (call out any delta from the customer's old Dosu values)
3. Engine selected
4. Whether the manual verification run succeeded
5. Remaining manual steps for the user (add API key secret, disable Dosu stale bot)
