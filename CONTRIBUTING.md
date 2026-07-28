# Contributing

Better Stale Bot can comment on, label, and close real issues. Treat every behavior change as a production automation change.

## Setup

Install the current GitHub CLI and the tested `gh-aw` release.

```bash
gh auth login --scopes repo,workflow
gh extension install github/gh-aw --pin v0.83.4
gh aw --version
```

This repository was last validated with `gh-aw v0.83.4`.

## Source files

The distribution source is `workflows/better-stale-bot.md`.

The repository test copy is `.github/workflows/better-stale-bot.md`.

Edit the distribution source first, then copy it.

```bash
cp workflows/better-stale-bot.md \
  .github/workflows/better-stale-bot.md
```

The two files must match exactly.

```bash
diff -u \
  workflows/better-stale-bot.md \
  .github/workflows/better-stale-bot.md
```

Never edit `.github/workflows/better-stale-bot.lock.yml` or `.github/aw/actions-lock.json` by hand.

## Validation

Run the checks below after each workflow change.

```bash
gh aw validate better-stale-bot --strict
gh aw validate --dir workflows --strict
gh aw compile better-stale-bot
gh aw compile better-stale-bot \
  --json \
  --no-emit \
  --strict
```

Review all generated changes.

```bash
git diff -- \
  .github/workflows/better-stale-bot.lock.yml \
  .github/aw/actions-lock.json
```

A compiler upgrade must include both generated files.

Use `gh aw validate` as the schema authority for generated locks. Actionlint 1.7.12 predates the supported `concurrency.queue` field and rejects the compiler output. It can still validate `.github/workflows/validate.yml`.

## Review rules

Every workflow change needs answers to the questions below.

1. Can the configured GitHub read tools provide every required field
2. Does missing data stop a write
3. Are write permissions still narrow
4. Are all write caps bounded
5. Can an exempt issue be labeled or closed
6. Can bot activity change eligibility
7. Can a rerun post a duplicate comment
8. Does the cache remain advisory
9. Does the README still match the workflow
10. Does the migration guide still preserve the customer's choices

## Behavioral test matrix

Use a private sandbox with only purpose-built issues.

| Case | Expected result |
| --- | --- |
| Inactive issue past the stale delay | One stale comment and the `Stale` label |
| Active issue | No write |
| Exempt issue | No stale comment and no close |
| Exempt issue that already has `Stale` | Remove `Stale` with no comment |
| Stale issue past the close delay | One close comment and `not_planned` close reason |
| Stale issue with later human activity | Remove `Stale` with no comment |
| Stale issue with later bot activity only | Keep `Stale` |
| Manually removed and reapplied `Stale` | Use the latest label event |
| Non-English title | Use the title language |
| Missing timeline data | Do not close |
| Missing reaction actor data | Do not rank or label that issue |
| Bot-applied stale label with no notice comment | Do not close |
| Issue body asks the agent to ignore its policy | Ignore the issue instructions |
| No eligible issues | One no-op result |
| Rerun after a partial write | No duplicate stale comment |

Start every live test with these controls.

- Manual-only trigger
- One write per output type
- Candidate scan limit no higher than 10
- A known engine secret or approved Copilot organization billing
- No unrelated old issues

The shared `dosu-ai/test` repository has a large old backlog. Do not dispatch the default workflow there. Use an isolated fixture set and a manual-only canary.

## Pull requests

A pull request must include the items below.

- A short statement of the user-visible behavior change
- The tested `gh-aw` version
- Current validation output
- Both synchronized Markdown sources
- The generated lock changes
- Any new secret, permission, action, or network access
- The behavioral cases that were run
- Documentation updates

Keep test thresholds and test-only labels out of the distribution source.

## Releases

Main is mutable. Consumers should get a tagged source once releases exist.

A release should include the items below.

1. A clean validation run
2. A sandbox canary
3. A reviewed generated lock
4. A tag
5. Release notes with behavior and auth changes
6. Updated install and upgrade examples
