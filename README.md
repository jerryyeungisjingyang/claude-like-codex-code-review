# Code Review for Codex — Claude-inspired Multi-Agent Skill

**A multi-agent code review skill for OpenAI Codex, with four independent reviewers and evidence-based verification.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Codex-Agent_Skill-111827)](SKILL.md)

[English](README.md) | [简体中文](README.zh-CN.md)

Review uncommitted changes, branch diffs, pull requests, or entire modules in Codex. Two reviewers check project rules and two look for bugs; fresh agents then verify candidate findings and look for counterevidence. Results include code locations, trigger conditions, and practical impact.

Use it when you want a Claude Code-inspired review workflow in Codex, including checks against `AGENTS.md`, `CLAUDE.md`, or your technical requirements documents (TRDs). Reviews are read-only by default and return findings in chat.

**Get started:** [Install the skill](#1-install), open the repository you want to review in Codex, and run:

```text
/claude-like-codex-code-review Review all uncommitted changes in the current workspace.
```

The full multi-agent workflow requires a Codex environment with subagent orchestration and model selection. See [models and fallback behavior](#models-and-usage) for compatibility details.

[Quick start](#quick-start) · [How it works](#how-it-works) · [Models and usage](#models-and-usage) · [FAQ](#faq)

## Why this project

A useful code review should answer three questions: **What breaks? What triggers it? Where is the evidence?**

This skill puts those questions into the workflow:

- **Four independent reviewers.** Two Sol low agents check project rules; two Sol high agents look for bugs.
- **Find an issue, then look for counterevidence.** Fresh agents verify candidates against existing safeguards and trigger conditions.
- **Report findings that meet the threshold.** Merge duplicate root causes and keep findings scoring at least 80, with locations, triggers, and impact.
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
```

These commands install the skill into your user-level skills directory and update an existing copy with the same name. Start a new Codex session and confirm that `claude-like-codex-code-review` appears in the skill list.

The full workflow requires a Codex environment with subagent orchestration and model selection. The current configuration targets environments offering `gpt-5.6-luna` and `gpt-5.6-sol`; fallback behavior is described below.

### 2. Run a review

Open the repository you want to review in Codex and enter:

**Review uncommitted changes:**

```text
/claude-like-codex-code-review Review all uncommitted changes in the current workspace.
```

**Review the current branch against main:**

```text
/claude-like-codex-code-review Review the current branch's changes against main.
```

**Review an entire service against its TRDs:**

```text
/claude-like-codex-code-review Review all of src/example-service/ against docs/requirements/. No comparison branch. Focus on state transitions, idempotency, and error recovery.
```

**Review an existing PR:**

```text
/claude-like-codex-code-review Review PR #123. Return findings only in chat.
```

PR data is read through an already available GitHub connector or `gh`. Missing permissions or history are reported as coverage limitations.

### 3. Request additional actions when needed

A standalone invocation reviews code and answers in chat. Specify any additional action explicitly:

```text
/claude-like-codex-code-review Review the current branch and save the report to review.md.
```

```text
/claude-like-codex-code-review Review the current changes and run relevant tests to verify candidate findings.
```

Saving a report does not also authorize creating a PR. Running tests does not also authorize editing code. Explicit authorization already given in the same task remains valid.

## How it works

```text
Confirm scope and code version
              ↓
Collect rules and summarize the module
              ↓
Four independent review tracks
Rules × 2 (Sol low) · Bugs × 2 (Sol high)
              ↓
Merge duplicate root causes
              ↓
Verify each candidate and seek counterevidence
              ↓
Keep findings scoring ≥ 80
              ↓
Recheck the code version and return findings in chat
```

| Review track | Key question |
| --- | --- |
| Project rules × 2 | Each independently checks applicable `AGENTS.md`, `CLAUDE.md`, and user-specified TRDs. |
| Bugs × 2 | Each independently checks functional errors and their concrete triggers, impact, and safeguards. |

History, prior PR discussions, and code comments remain available as supporting evidence when relevant; they are not separate mandatory tracks.

The four tracks use independent contexts and run in batches within the environment's concurrency limit. Discovery agents do not receive other reviewers' conclusions. Each merged candidate is scored once by a fresh Luna medium agent that reads the code and seeks counterevidence. There is no additional Sol recheck.

**80 is a review threshold, not an “80% probability of being correct.”** Scores describe the strength of evidence; severity is assessed separately. If no finding meets the threshold, the review still reports its outcome and coverage limitations.

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

| Work | Default model | Reasoning effort |
| --- | --- | --- |
| Scope, rule paths, summary, final baseline check | GPT-5.6 Luna | medium |
| Two independent rule reviews | GPT-5.6 Sol | low |
| Two independent bug reviews | GPT-5.6 Sol | high |
| Independent candidate scoring | GPT-5.6 Luna | medium |

The coordinator keeps your current session model. If you use Astra, Astra still orchestrates the review and summarizes the results; subagents use the models above. Switch the session model before invoking the skill if you want Sol to coordinate as well.

Multiple agents add code-reading, reasoning, and synthesis costs. Model tiers help manage the cost of different assignments; they do not promise a fixed saving or fewer total tokens than a single-agent review.

Model IDs and dispatch rules live in [SKILL.md](SKILL.md). If a specified model is unavailable, the workflow first tries an available Luna / Sol alternative. If none is available, or agent tools are missing, it discloses the limitation and falls back to sequential review by the current model. It does not label that fallback as four independent reviews.

## FAQ

### Will it edit code, write Markdown, or create a PR automatically?

Not by default. A standalone invocation authorizes reading and a chat response. Tests, builds, and dependency installation are also excluded by default. Additional actions require an explicit request, and the same boundary applies to subagents.

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
