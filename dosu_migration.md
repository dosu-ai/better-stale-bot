# Migrate from Dosu stale bot

This guide is written for a coding agent with shell and GitHub CLI access. A person can follow the same steps.

Dosu's hosted stale bot is scheduled for removal on August 1, 2026. Better Stale Bot runs in the customer repository through GitHub Actions.

The migration must not change policy by accident. Stop when a setting is unknown.

## Required inputs

- The target repository in `owner/repo` form
- `days_before_stale`
- `days_before_close`
- `operations_per_run`
- The exact exempt label list, including an explicit empty list when no labels are exempt
- The AI engine
- The owner of the engine bill and secret

Do not treat an unknown value as an empty value.

## Important differences

| Setting | Dosu | Better Stale Bot |
| --- | --- | --- |
| Stale delay | 90 days by default | 60 days by default |
| Close delay | 7 days by default | 7 days by default |
| Work limit | One combined operation cap | One cap for each write type |
| Candidate scan | Managed by Dosu | 100 oldest eligible issues by default |
| Exempt labels | Customer setting | Four defaults in the template |
| Runtime | Hosted by Dosu | Customer GitHub Actions |

The old combined operation cap cannot be copied exactly. A value of 25 on four new write types can allow up to 100 writes. Use `1` for every write type during the canary. Raise each cap only after the first run is reviewed.

Body settings take effect on the next run. Frontmatter settings require `gh aw compile`.

## Step 1

Check the local and remote state.

```bash
gh auth status
gh extension install github/gh-aw
gh extension upgrade github/gh-aw
gh repo view OWNER/REPOSITORY
git status --short
```

Confirm that GitHub Actions is enabled.

Inventory the current stale state before installing anything.

```bash
gh issue list --repo OWNER/REPOSITORY \
  --state open \
  --label Stale \
  --limit 1000
```

Record any issue that could close on the first run.

## Step 2

Install the workflow without prompts.

```bash
gh aw add dosu-ai/better-stale-bot/better-stale-bot
```

The command adds the editable Markdown source and compiled lock file. A first time `gh-aw` setup can also add the files below.

- `.github/agents/agentic-workflows.md`
- `.github/skills/agentic-workflows/SKILL.md`
- `.github/mcp.json`
- `.github/workflows/copilot-setup-steps.yml`
- `.gitattributes`

Review every generated file.

## Step 3

Set the workflow to manual-only mode for the canary.

```yaml
on: workflow_dispatch
```

Keep the default detailed scan limit at 100 unless the repository needs a smaller canary.

```markdown
| `candidate-scan-limit` | Maximum Bucket B issues to inspect in detail per run after the oldest eligible issues are selected | 100 |
```

Map the stale and close delays in `## Configuration`.

Use the exact exempt label list supplied by the customer.

If the list is empty, disable the no-op issue before removing `agentic-workflows`.

```yaml
safe-outputs:
  noop:
    report-as-issue: false
```

Do not keep `pinned`, `security`, or `help wanted` unless the customer chooses them.

## Step 4

Use one write per type for the canary.

```yaml
safe-outputs:
  add-comment:
    max: 1
    target: "*"
    issues: true
    pull-requests: false
    discussions: false
  add-labels:
    max: 1
    target: "*"
    allowed: ["Stale"]
  remove-labels:
    max: 1
    target: "*"
    allowed: ["Stale"]
  close-issue:
    max: 1
    target: "*"
  noop:
```

If no-op issue reporting is disabled, keep `report-as-issue: false` under `noop`.

## Step 5

Set the engine with current syntax.

Claude Haiku

```yaml
engine: claude
model: haiku
```

Copilot

```yaml
engine: copilot
```

Codex

```yaml
engine: codex
```

Copilot organization billing also needs the permission below.

```yaml
permissions:
  contents: read
  issues: read
  copilot-requests: write
```

## Step 6

Make sure the `Stale` label exists.

```bash
gh label create Stale \
  --repo OWNER/REPOSITORY \
  --color BFD4F2 \
  --description "Inactive issue awaiting confirmation" \
  --force
```

Compile and validate.

```bash
gh aw compile better-stale-bot
gh aw validate better-stale-bot --strict
gh aw secrets bootstrap \
  --repo OWNER/REPOSITORY \
  --non-interactive
```

Review `.github/workflows/better-stale-bot.lock.yml`. Confirm the engine, schedule, permissions, write caps, and required secret.

## Step 7

Configure engine authentication.

| Engine | Authentication |
| --- | --- |
| Claude | `ANTHROPIC_API_KEY` |
| Codex | `OPENAI_API_KEY` |
| Copilot | Organization billing or `COPILOT_GITHUB_TOKEN` |

Never echo or hardcode a secret. Let the customer enter it through GitHub or an interactive `gh aw secrets set` prompt.

## Step 8

Pause Dosu's hosted stale bot before the new workflow can run.

Do not leave both systems active.

Commit the manual-only canary on a branch and open a pull request.

```bash
git add .github/ .gitattributes
git commit -m "chore: migrate stale issue automation"
git push
```

Merge after reviewing the generated lock file.

## Step 9

Run one manual canary.

```bash
gh aw run better-stale-bot --repo OWNER/REPOSITORY
gh run list \
  --repo OWNER/REPOSITORY \
  --workflow better-stale-bot.lock.yml \
  --limit 1
```

Inspect the run with a real run ID.

```bash
gh aw logs better-stale-bot --repo OWNER/REPOSITORY
gh aw audit RUN_ID --repo OWNER/REPOSITORY
gh aw outcomes RUN_ID --repo OWNER/REPOSITORY
```

Check the issue comment, label, close reason, language, and duplicate behavior.

## Step 10

Enable the production schedule only after approval.

```yaml
on:
  schedule: daily
```

Choose each write cap separately. Do not reuse the old combined cap without review.

Recompile, validate, and merge the production change.

```bash
gh aw compile better-stale-bot
gh aw validate better-stale-bot --strict
```

## Failure rules

- Missing timeline data means no close.
- Missing reaction actor data means no computed engagement score.
- Missing engine authentication means no live run.
- A threat detection failure means no writes should be trusted as applied.
- Cache state is advisory. Current GitHub state wins.
- A missing or uncertain customer setting stops the migration.

## Completion report

Report the items below when the work is done.

1. The repository and pull request
2. The old and new stale delays
3. The old and new close delays
4. The candidate scan limit
5. The canary caps
6. The production caps
7. The exempt label list
8. The selected engine and auth owner
9. The canary run ID and outcome
10. Confirmation that Dosu's old bot is paused
