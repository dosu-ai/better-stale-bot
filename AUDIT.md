# Repository audit

This audit covers the full repository as reviewed on July 23, 2026. It records the maintainer fit, adversarial findings, external research, validation work, and the test against `dosu-ai/test`.

## Intended user

Better Stale Bot fits maintainers who have a large issue backlog, want issue-specific stale messages, and can own a GitHub Actions workflow plus model billing.

It is a poor fit when a repository cannot send issue content to a model provider, needs a deterministic rules engine, cannot review generated permissions, or lacks a safe canary repository.

## Review method

Three adversarial reviewers examined the repository from different user perspectives.

| Reviewer | Perspective | Main question |
| --- | --- | --- |
| Maintainer onboarding | First install and daily operation | Can a maintainer install this safely without private context |
| Open source distribution | Packaging and project health | Can users trust, update, validate, and contribute to the project |
| Safety and reliability | Destructive behavior and failure handling | Can incomplete data or reruns cause a wrong label, comment, or close |

The review also traced every tracked file, inspected historical workflow runs, compiled the source with `gh-aw` v0.83.1, and checked the target repository before any live action. The source and generated files were recompiled and revalidated with `gh-aw` v0.83.4 on July 27, 2026.

## Changes made

| Finding | Change |
| --- | --- |
| The original GitHub toolset did not guarantee timeline and actor-attributed reaction data | Use the read-only `gh-proxy` mode and require complete pagination |
| Missing data could still lead to a write | Call `missing-data` and stop work on the affected issue |
| Exempt stale issues could reach closure logic | Make exemptions win before analysis and immediately before a write |
| The engagement score rewarded newer old issues and could starve the oldest backlog | Remove age from the score and use oldest qualifying activity as the tie break |
| A full repository scan would not scale | Inspect a bounded window of the 100 oldest eligible issues |
| Cache state could override current GitHub state | Make cache content advisory only |
| Bot identity and qualifying activity were underspecified | Define both terms using actor metadata and explicit events |
| Separate safe outputs can leave a partial write | Recheck state before writes and detect an earlier stale comment on rerun |
| Comment and close targets were implicit | Set an explicit target and opt comments out of pull requests and discussions |
| The close reason used an invalid enum value | Use `not_planned` |
| The workflow used deprecated engine syntax | Use `engine: claude` with top-level `model: haiku` |
| The migration could enable two bots and multiply the old cap | Pause the old bot first and use a manual canary with one write per type |
| Contributors had no validation path | Add `CONTRIBUTING.md` and a compile drift check in CI |
| Installation files conflicted with `.gitignore` | Allow the generated Copilot setup workflow to be committed |

## Compiler security review

The v0.83.4 compiler reports `ANTHROPIC_API_KEY` as a new restricted secret for the distribution source. This is expected because Claude is the selected engine. No secret value is stored in the repository. The generated workflow reads the key from GitHub Actions secrets and excludes it from the general tool environment.

The generated manifest records the actions and container digests used by the workflow. The generated workflow now uses `gh-aw` v0.83.4 and its refreshed runtime dependencies.

## Target repository test

The shared target `dosu-ai/test` is private and uses `banana` as its default branch. It had 1,289 open issues during this audit. About 1,281 matched the default inactivity filter. Three open issues already carried the stale label.

The documented installation command completed in a temporary HTTPS clone.

```bash
gh aw add dosu-ai/better-stale-bot/better-stale-bot
```

The command added the editable source, compiled lock, action pins, agent guidance, MCP configuration, skill guidance, Copilot setup workflow, and generated file attributes. The installed workflow passed `gh aw validate`.

The final local source also installed over the temporary copy and passed strict validation. A `gh aw trial` dry run produced a valid execution plan for `dosu-ai/test` and confirmed that no host repository or target change would be made during the preview.

No branch, issue, label, comment, workflow run, secret, or repository setting was changed in `dosu-ai/test`.

A live run was not safe for two reasons.

- The repository had no supported engine secret
- The default policy could act on a large real backlog and close the three existing stale issues

The next live test should use an isolated private repository with purpose-built issues. Use a manual trigger, a candidate scan limit no higher than 10, and one write per output type.

## External research

Exa research across current primary sources highlighted four distribution gaps. The repository did not state public preview status, validate generated drift in CI, publish tagged releases, or provide a security reporting path. This change fixes the first two. The release and security work remains open.

Current GitHub guidance supports the changes in this audit.

- [GitHub Agentic Workflows](https://github.github.com/gh-aw/) describes the public preview and the editable source plus compiled lock model
- [Creating GitHub Agentic Workflows](https://docs.github.com/en/copilot/how-tos/github-agentic-workflows/creating-github-agentic-workflows) recommends validation and review of generated workflows
- [GitHub Agentic Workflows FAQ](https://github.github.com/gh-aw/reference/faq/) covers permissions, auth, safe outputs, and operational limits
- [GitHub Actions concurrency](https://docs.github.com/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs) confirms that `queue: max` is valid. Actionlint 1.7.12 has not added the field to its local schema.
- [Maintaining GitHub Actions](https://docs.github.com/en/actions/how-tos/create-and-publish-actions/release-and-maintain-actions) supports tagged releases and clear upgrade paths for reusable automation

The repository now documents public preview status, engine auth, data exposure, source and lock ownership, validation, canary controls, update steps, rollback, and generated permissions.

## Open work

| Priority | Improvement | Why it remains open |
| --- | --- | --- |
| P0 | Run the full behavioral matrix in an isolated fixture repository | The shared test repository is unsafe and has no engine auth |
| P1 | Reduce the compiled pull request write permission | The v0.83.4 shared label handlers still request it |
| P1 | Publish a signed tag and release | Install examples currently track mutable `main` |
| P1 | Add a security policy and private reporting route | Private vulnerability reporting is not enabled |
| P1 | Run the new validation workflow on GitHub | Local checks cannot prove hosted CI settings |
| P1 | Add deterministic pre-write policy checks | Eligibility and exemption checks still depend on model compliance |
| P2 | Run prompt injection and hostile issue content fixtures | The policy and test matrix now cover hostile text but no live fixture has run |
| P2 | Decide whether to pin a dated model version | The `haiku` alias can change behavior over time |
| P2 | Add release notes or a changelog | Users need a concise record of policy and auth changes |

## Release gate

Do not call the project production proven until all P0 work is complete. A release candidate should also resolve or explicitly accept each P1 item.
