# Code Review for Codex — Claude-inspired Multi-Agent Skill

**Configurable code review for OpenAI Codex, with optional fixes and GitHub PR inline comments.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Codex-Agent_Skill-111827)](SKILL.md)

[English](README.md) | [简体中文](README.zh-CN.md)

Review uncommitted changes, branch diffs, pull requests, or selected paths in Codex. Effort levels from `low` through `max` control review depth, reviewer count, and verification strength. Use `--fix` to apply local fixes and `--comment` to publish GitHub PR inline comments.

Use it when you want a Claude Code-inspired review workflow in Codex, including checks against `AGENTS.md`, `CLAUDE.md`, or your technical requirements documents (TRDs). Reviews are read-only by default and return findings in chat.

**Get started:** [Install the skill](#1-install), open the repository you want to review in Codex, and run:

```text
$claude-like-codex-code-review [low|medium|high|xhigh|max] [--fix] [--comment] [<pr#>|<branch>|<path>]
```

The full multi-agent workflow requires a Codex environment with subagent orchestration and model selection. See [models and fallback behavior](#models-and-usage) for compatibility details.

[Quick start](#quick-start) · [How it works](#how-it-works) · [Models and usage](#models-and-usage) · [FAQ](#faq)

## Why this project

A useful code review should answer three questions: **What breaks? What triggers it? Where is the evidence?**

This skill puts those questions into the workflow:

- **Scale the review by effort.** `low` is a quick single-model pass, the default `high` profile uses four independent reviewers, and `xhigh`/`max` broaden discovery and verification.
- **Find an issue, then look for counterevidence.** Fresh agents verify candidates against existing safeguards and trigger conditions.
- **Report findings that meet the selected threshold.** Merge duplicate root causes, filter weak evidence by effort profile, and include locations, triggers, and impact.
- **Review a diff or an entire module.** Full reviews include existing defects. Specify a TRD to check implementation against business requirements.
- **Read-only by default, results in chat.** Editing code, saving reports, creating PRs, posting comments, and running tests require explicit additional requests.

## Quick start

### 1. Install

Clone this repository and install the skill from its root directory:

```bash
git clone https://github.com/jerryyeungisjingyang/claude-like-codex-code-review.git
cd claude-like-codex-code-review
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review"
cp SKILL.md "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/SKILL.md"
cp -R agents "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
cp -R references "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
```

These commands install the skill into your user-level skills directory and update an existing copy with the same name. Start a new Codex session and confirm that `claude-like-codex-code-review` appears in the skill list.

The full workflow requires a Codex environment with subagent orchestration and model selection. The current configuration targets environments offering `gpt-5.6-luna` and `gpt-5.6-sol`; fallback behavior is described below.

### 2. Run a review

Open the repository you want to review in Codex. Every argument is optional and the default effort is `high`:

```text
$claude-like-codex-code-review [low|medium|high|xhigh|max] [--fix] [--comment] [<pr#>|<branch>|<path>]
```

**Review uncommitted changes:**

```text
$claude-like-codex-code-review low
```

**Review the current branch against its default baseline:**

```text
$claude-like-codex-code-review high
```

**Review a named branch:**

```text
$claude-like-codex-code-review high feature/login
```

**Review a selected path deeply:**

```text
$claude-like-codex-code-review xhigh src/example-service/
```

Append ordinary-language review criteria when you need a TRD or special risk focus. Text that is not parsed as an effort, flag, or target remains review guidance.

**Review an existing PR:**

```text
$claude-like-codex-code-review high 123
```

**Review a PR and publish inline comments:**

```text
$claude-like-codex-code-review medium --comment 123
```

**Review and fix local code:**

```text
$claude-like-codex-code-review high --fix src/auth/
```

PR data is read through an already available GitHub connector or `gh`. Missing permissions or history are reported as coverage limitations.

### 3. Arguments

| Argument | Behavior |
| --- | --- |
| `low` | One quick pass; report at most 4 directly demonstrated findings. |
| `medium` | Two independent discovery tracks with verification; report at most 6 findings. |
| `high` | Default; four independent tracks with verification; report at most 10 findings. |
| `xhigh` | Six discovery tracks and two-vote verification; report at most 15 findings. |
| `max` | Eight discovery tracks, two-vote verification, and a gap sweep; report at most 20 findings. |
| `--fix` | Apply retained findings locally and run reasonably scoped existing verification; do not commit or push. |
| `--comment` | Publish retained findings as inline comments on the resolved GitHub PR. |
| `PR / branch / path` | Accept one target such as `123`, `feature/login`, or `src/auth/`. |

`--comment` requires exactly one open PR. When the target is not a PR, the skill resolves it from the target branch or current branch and stops for a PR number if the result is missing or ambiguous.

Saving a report, creating a commit, pushing, or opening a PR still requires a separate explicit request. `--fix` and `--comment` authorize only their corresponding actions.

## How it works

```text
Confirm scope and code version
              ↓
Collect rules and summarize the module
              ↓
Discovery tracks selected by low / medium / high / xhigh / max
              ↓
Merge duplicate root causes
              ↓
Verify each candidate and seek counterevidence
              ↓
Keep findings meeting the selected threshold
              ↓
Recheck the code version and return findings in chat
```

| Default `high` review track | Key question |
| --- | --- |
| Project rules × 2 | Each independently checks applicable `AGENTS.md`, `CLAUDE.md`, and user-specified TRDs. |
| Bugs × 2 | Each independently checks functional errors and their concrete triggers, impact, and safeguards. |

History, prior PR discussions, and code comments remain available as supporting evidence when relevant; they are not separate mandatory tracks.

The default `high` profile uses four independent contexts and runs them in batches within the environment's concurrency limit. `medium` uses two tracks, while `xhigh` and `max` add bug-discovery angles and two-vote verification. Discovery agents do not receive other reviewers' conclusions.

Scores describe evidence strength rather than statistical probability; severity is assessed separately. `high`, `xhigh`, and `max` use an 80-point floor, `medium` uses 90, and `low` reports only directly demonstrated issues. The review still reports its outcome and limitations when no finding survives.

## What you get

This is an illustrative output format, not a measured result from this project:

```text
[P1] Expired sessions remain valid

Location: src/session.py:42
Trigger: A request presents a session whose expiration time is in the past.
Impact: The request is accepted even though the session should be rejected.
Evidence: The expiration check compares now < expires_at instead of now >= expires_at.
Counterevidence: No later expiration check exists before access is granted.
Score: 90
Verification: Static call-chain inspection; no tests executed.
```

Each finding includes a file link, trigger, practical impact, and key evidence. The review states which tests ran and which data could not be accessed.

## Models and usage

| Level | Discovery | Verification |
| --- | --- | --- |
| `low` | Coordinator-only pass | Keep directly demonstrated findings only |
| `medium` | Sol: rules low ×1, bugs medium ×1 | Luna low ×1 |
| `high` | Sol: rules low ×2, bugs high ×2 | Luna medium ×1 |
| `xhigh` | Sol: rules medium ×2, bugs xhigh ×4 | Luna high ×2 |
| `max` | Sol: rules high ×2, bugs max ×6, then a gap sweep | Luna max ×2 |

The coordinator keeps your current session model. If you use Astra, Astra still orchestrates the review and summarizes the results; subagents use the models above. Switch the session model before invoking the skill if you want Sol to coordinate as well.

Multiple agents add code-reading, reasoning, and synthesis costs. Model tiers help manage the cost of different assignments; they do not promise a fixed saving or fewer total tokens than a single-agent review.

Model IDs and dispatch rules live in [SKILL.md](SKILL.md). If a specified model is unavailable, the workflow first tries an available Luna / Sol alternative. If none is available, or agent tools are missing, it discloses the limitation and falls back to sequential review by the current model. It does not label that fallback as four independent reviews.

## FAQ

### Will it edit code, write Markdown, or create a PR automatically?

Not by default. `--fix` explicitly authorizes local fixes and `--comment` explicitly authorizes PR comments; neither authorizes commits, pushes, or PR creation. Without those flags, the skill reads code and answers in chat only.

### Can earlier conversation affect a review?

The coordinator still sees the current conversation. The skill requires subagents to start with `fork_turns: "none"`, receiving only the review scope, rule paths, and necessary materials to reduce inherited assumptions. This does not eliminate coordinator bias when selecting materials or summarizing results. For less influence from earlier conversation, invoke the skill in a fresh session with an explicit scope.

### Does it guarantee bug-free code or the same findings as Claude?

No. Models, scope, and available evidence affect the outcome. Independent verification helps filter false positives, but static review can still miss runtime issues and does not replace testing or human review.

### Do I need a PR?

No. Review workspace changes, branch differences, or an entire specified module. If needed historical PR evidence is unavailable, the review reports that limitation.

### Is this an official Claude or OpenAI project?

This is an independent project inspired by Claude Code's `code-review` plugin. It is not affiliated with or endorsed by Anthropic or OpenAI. Reviews use the models and tools in your Codex environment; no Claude invocation is required.

### What language does the review use?

The skill instructions are written in English. Review responses default to Chinese; ask for English in your prompt if preferred.

## Contributing

Reproducible missed bugs, false positives, and compatibility reports are welcome. Useful reports include the review scope, actual models used, a minimal code example, and expected versus actual findings. Remove secrets and private business data before sharing.

The complete workflow is defined in [SKILL.md](SKILL.md). Display metadata and the default prompt live in [agents/openai.yaml](agents/openai.yaml).

## Acknowledgments

Thanks to [Claude Code's code-review plugin](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review) for the workflow inspiration. This project adapts the approach to Codex agent orchestration, model tiers, and local code review.

## License

[MIT](LICENSE) © 2026 jerry_yang
