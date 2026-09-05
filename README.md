# claude-like-codex-code-review

**Bring Claude code-review's five independent review tracks to Codex.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Codex-Agent_Skill-111827)](SKILL.md)

[English](#english) | [简体中文](#简体中文)

<a id="english"></a>

Claude Code's `code-review` plugin is great. Independent agents inspect the code, then verify each candidate finding and filter out false positives. I wanted that workflow in Codex, too.

So I built a Codex version.

One command starts five independent review tracks, follows code and history, checks project rules, and brings evidence-backed findings back to your chat. Review PRs, branch changes, or entire modules with Luna / Terra / Sol handling different parts of the work.

```text
/claude-like-codex-code-review Review the entire payment service against docs/TRD/. No comparison branch.
```

[Quick start](#quick-start) · [How it works](#how-it-works) · [Models and usage](#models-and-usage) · [FAQ](#faq)

## Why this project

A useful code review should answer three questions: **What breaks? What triggers it? Where is the evidence?**

This skill puts those questions into the workflow:

- **Five perspectives, independent discovery.** Check project rules, obvious bugs, historical changes, prior PR comments, and code comments separately.
- **Find an issue, then look for counterevidence.** Fresh agents verify candidates against existing safeguards and trigger conditions.
- **Report findings that meet the threshold.** Merge duplicate root causes and keep findings scoring at least 80, with locations, triggers, and impact.
- **Review a diff or an entire module.** Full reviews include existing defects. Specify a TRD to check implementation against business requirements.
- **Read-only by default, results in chat.** Editing code, saving reports, creating PRs, posting comments, and running tests require explicit additional requests.

## Quick start

### 1. Install

Download or clone this repository, then run from its root directory:

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review"
cp SKILL.md "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/SKILL.md"
cp -R agents "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
```

These commands install the skill into your user-level skills directory and update an existing copy with the same name. Start a new Codex session and confirm that `claude-like-codex-code-review` appears in the skill list.

The full workflow requires a Codex environment with subagent orchestration and model selection. The current configuration targets environments offering `gpt-5.6-luna`, `gpt-5.6-terra`, and `gpt-5.6-sol`; fallback behavior is described below.

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
/claude-like-codex-code-review Review all of biz/service/qrpay/ against docs/TRD/. No comparison branch. Focus on money movement, idempotency, and error recovery.
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
Five independent review tracks
Rules · Bugs · History · PR feedback · Comments
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
| Project rules | Does the implementation violate applicable `AGENTS.md`, `CLAUDE.md`, or user-specified TRDs? |
| Obvious bugs | Is there a functional error with a concrete trigger and impact? |
| Historical causality | Did a later commit undo a safeguard? Was a migration missed? |
| Historical PR comments | Do previously reported issues still exist or recur? |
| Comments versus implementation | Do locks, states, idempotency, and recovery behave as their comments describe? |

The five tracks use independent contexts and run in batches within the environment's concurrency limit. Discovery agents do not receive other reviewers' conclusions. Verification agents read the code themselves to determine whether each candidate holds up.

**80 is a review threshold, not an “80% probability of being correct.”** Scores describe the strength of evidence; severity is assessed separately. If no finding meets the threshold, the review still reports its outcome and coverage limitations.

## What you get

This is an illustrative output format, not a measured result from this project:

```text
[P1] Duplicate refund events enter the cumulative amount calculation

Location: webhook.go:94
Trigger: Two refund notifications with the same event_hash are processed in sequence.
Impact: The second notification may count the refund again during the limit check,
        incorrectly routing it to manual review.
Evidence: The caller continues calculating the total after the event insert reports
          that the event already exists.
Counterevidence: A later unique index prevents duplicate credits, but does not prevent
                 the earlier incorrect limit check.
Score: 90
Verification: Static call-chain inspection; no tests executed.
```

Each finding includes a file link, trigger, practical impact, and key evidence. The review states which tests ran and which data could not be accessed.

## Models and usage

| Work | Default model | Reasoning effort |
| --- | --- | --- |
| Scope, rule paths, summary, final baseline check | GPT-5.6 Luna | low |
| Rules, obvious bugs, historical PR comments, and code comments | GPT-5.6 Terra | medium |
| Historical causality | GPT-5.6 Sol | medium |
| Independent candidate scoring | GPT-5.6 Luna | medium |
| Necessary rechecks for conflicting evidence or complex findings | GPT-5.6 Sol | medium |

The coordinator keeps your current session model. If you use Astra, Astra still orchestrates the review and summarizes the results; subagents use the models above. Switch the session model before invoking the skill if you want Sol or Terra to coordinate as well.

Multiple agents add code-reading, reasoning, and synthesis costs. Model tiers help manage the cost of different assignments; they do not promise a fixed saving or fewer total tokens than a single-agent review.

Model IDs and dispatch rules live in [SKILL.md](SKILL.md). If a specified model is unavailable, the workflow first tries an available Luna / Terra / Sol alternative. If none is available, or agent tools are missing, it discloses the limitation and falls back to sequential review by the current model. It does not label that fallback as five independent reviews.

## FAQ

### Will it edit code, write Markdown, or create a PR automatically?

Not by default. A standalone invocation authorizes reading and a chat response. Tests, builds, and dependency installation are also excluded by default. Additional actions require an explicit request, and the same boundary applies to subagents.

### Can earlier conversation affect a review?

The coordinator still sees the current conversation. The skill requires subagents to start with `fork_turns: "none"`, receiving only the review scope, rule paths, and necessary materials to reduce inherited assumptions. This does not eliminate coordinator bias when selecting materials or summarizing results. For less influence from earlier conversation, invoke the skill in a fresh session with an explicit scope.

### Does it guarantee bug-free code or the same findings as Claude?

No. Models, scope, and available evidence affect the outcome. Independent verification helps filter false positives, but static review can still miss runtime issues and does not replace testing or human review.

### Do I need a PR?

No. Review workspace changes, branch differences, or an entire specified module. If historical PR data is unavailable, that track is marked as uncovered.

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

---

<a id="简体中文"></a>

## 简体中文

[English](#english) | [简体中文](#简体中文)

**把 Claude code-review 的五路独立评审流程，带到 Codex。**

Claude Code 的 `code-review` 插件很好用。多个代理分别检查代码，再逐项核实候选问题、过滤误报，这套流程值得在 Codex 里也用起来。

所以，我做了一个 Codex 版本。

一个命令，启动五路独立评审，追踪代码与历史，核对项目规范，最后把有证据的问题带回聊天。支持 PR、分支改动和指定模块全量审查，使用 Luna / Terra / Sol 分工。

```text
/claude-like-codex-code-review 对照 docs/TRD/，审查整个支付服务，没有对比分支。
```

[快速开始](#快速开始) · [评审流程](#评审流程) · [模型与用量](#模型与用量) · [常见问题](#常见问题)

## 为什么做这个项目

一次有用的 code review，应该回答三个问题：**哪里会出错、什么情况下触发、证据在哪里。**

这个 skill 把这些要求放进执行流程：

- **五个角度，独立发现。** 分别检查规范、明显 bug、历史变更、历史 PR 评论和代码注释。
- **发现之后，再找反证。** 候选问题交给新的代理复核，检查已有保护措施和触发前提。
- **只报告达到阈值的问题。** 合并重复根因，保留置信评分至少 80 的发现，附上位置、触发条件和影响。
- **能审改动，也能审整个模块。** 全量模式会检查已有缺陷；明确指定 TRD 时，对照业务约束审查实现。
- **默认只读，结果回到聊天。** 修改代码、保存报告、创建 PR、发布评论和运行测试，都需要明确的附加要求。

## 快速开始

### 1. 安装

下载或克隆本仓库，在仓库根目录执行：

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review"
cp SKILL.md "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/SKILL.md"
cp -R agents "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
```

以上命令会安装到用户级 skills 目录；已有同名版本时会更新它。重新开启 Codex 会话，在技能列表中确认 `claude-like-codex-code-review` 已被发现。

完整流程需要支持子代理调度及模型选择的 Codex 环境。当前模型配置面向提供 `gpt-5.6-luna`、`gpt-5.6-terra`、`gpt-5.6-sol` 的环境；其他环境的降级方式见下文。

### 2. 开始评审

在需要评审的代码仓库中，向 Codex 输入：

**检查当前未提交的改动：**

```text
/claude-like-codex-code-review 审查当前工作区所有未提交改动。
```

**检查当前分支相对 main 的改动：**

```text
/claude-like-codex-code-review 对比 main，审查当前分支的改动。
```

**对照 TRD，全量检查指定服务：**

```text
/claude-like-codex-code-review 对照 docs/TRD/，全量审查 biz/service/qrpay/，没有对比分支，重点看资金流、幂等和异常恢复。
```

**评审已有 PR：**

```text
/claude-like-codex-code-review 审查 PR #123，只在聊天中给出结论。
```

PR 数据通过环境中已有的 GitHub 连接器或 `gh` 读取。缺少权限或历史数据时，评审会说明未覆盖的部分。

### 3. 按需追加动作

单独调用 skill，默认只检查并回答。需要其他动作时，把要求写清楚：

```text
/claude-like-codex-code-review 审查当前分支，并将报告保存到 review.md。
```

```text
/claude-like-codex-code-review 审查当前改动，并运行相关测试验证候选问题。
```

保存报告不会同时授权创建 PR；运行测试也不会同时授权修改代码。同一任务中已经明确给出的授权继续有效。

## 评审流程

```text
确认范围与代码版本
        ↓
收集规范 · 整理模块摘要
        ↓
┌────────┬────────┬────────┬────────┬────────┐
│ 项目规范 │ 明显 bug │ 历史因果 │ PR 评论 │ 注释实现 │
└────────┴────────┴────────┴────────┴────────┘
        ↓
合并重复根因
        ↓
独立复核每个候选 · 主动寻找反证
        ↓
保留评分 ≥ 80 的发现
        ↓
复核代码版本 · 在聊天中返回结论
```

| 评审角度 | 关注的问题 |
| --- | --- |
| 项目规范 | 实现是否违反适用的 `AGENTS.md`、`CLAUDE.md`，以及用户指定的 TRD？ |
| 明显 bug | 是否存在能明确说明触发条件和影响的功能错误？ |
| 历史因果 | 过去的修复是否被后续提交撤回？迁移是否遗漏？ |
| 历史 PR 评论 | 以前指出的问题是否仍然存在或再次出现？ |
| 注释与实现 | 锁、状态、幂等和恢复逻辑，是否与代码注释描述一致？ |

五路使用独立上下文，按环境的并发上限分批执行。发现阶段不传递其他评审者的结论；复核阶段要求自行读取代码，检查候选是否成立。

**80 分是评审阈值，不是“80% 的真实概率”。** 评分表示证据充分程度，严重程度单独判断。没有达到阈值的问题时，也会明确返回结果和覆盖限制。

## 你会得到什么

以下为输出结构示例，不是本项目的实测结果：

```text
[P1] 重复退款事件进入累计金额计算

位置：webhook.go:94
触发：同一 event_hash 的两个退款通知先后进入处理逻辑。
影响：第二次处理可能将同一笔退款重复纳入额度判断，误转人工复核。
证据：事件写入返回“已存在”后，调用方仍继续计算累计金额。
复核：后续唯一索引可阻止重复入账，但不能阻止前面的额度误判。
评分：90
验证：静态调用链核查，未运行测试。
```

每项发现包含可定位的文件链接、触发条件、实际影响与关键证据。测试是否执行、哪些数据无法访问，会如实说明。

## 模型与用量

| 工作 | 默认模型 | 推理强度 |
| --- | --- | --- |
| 范围整理、规范路径收集、摘要、最终基线检查 | GPT-5.6 Luna | low |
| 规范、明显 bug、历史 PR 评论、注释四路评审 | GPT-5.6 Terra | medium |
| 历史因果评审 | GPT-5.6 Sol | medium |
| 候选问题独立评分 | GPT-5.6 Luna | medium |
| 证据冲突或复杂问题的必要二次复核 | GPT-5.6 Sol | medium |

主协调者保持当前会话模型。如果当前使用 Astra，任务调度和最终汇总仍由 Astra 完成，子代理按上表选择模型。希望协调部分也使用 Sol 或 Terra，可以在调用前切换会话模型。

多代理会产生额外的代码读取、推理和汇总消耗。分级模型用于控制分工成本，不承诺固定节省比例，也不代表总 token 数比单代理更少。

模型 ID 和调度规则写在 [SKILL.md](SKILL.md) 中。指定模型不可用时，优先在可用的 Luna / Terra / Sol 中替代；全部不可用或缺少代理工具时，说明限制并降级为当前模型顺序检查，不将降级结果称为五路独立评审。

## 常见问题

### 会自动改代码、写 Markdown 或创建 PR 吗？

默认不会。独立调用只授权读取和聊天回答，也不默认运行测试、构建或安装依赖。附加动作需要用户明确要求，这个边界同样适用于子代理。

### 在已有的长对话中使用，会受之前结论影响吗？

主协调者仍能看到当前对话。skill 要求子代理以 `fork_turns: "none"` 启动，只接收评审范围、规范路径和必要资料，减少继承既有判断的影响。这不能完全消除协调者在选取资料和汇总时的偏差。希望进一步减少历史上下文影响，可以在新会话中给出明确范围后调用。

### 能保证没有 bug，或者与 Claude 的结果一致吗？

不能。模型、代码范围和可访问证据都会影响结果。独立复核用于过滤误报；静态评审仍可能遗漏运行时问题，也不能替代测试和人工审查。

### 必须有 PR 才能用吗？

不需要。可以检查工作区改动、分支差异或指定模块的全量代码。没有历史 PR 数据时，对应维度会标记为未覆盖。

### 这是 Claude 或 OpenAI 的官方项目吗？

这是受 Claude Code `code-review` 插件启发的独立项目，与 Anthropic、OpenAI 均无隶属或背书关系。运行评审使用 Codex 环境中的模型和工具，无需调用 Claude。

### 评审使用什么语言？

技能指令使用英文编写，评审结果默认使用中文；需要英文结果时，在提示中明确提出即可。

## 参与改进

欢迎提交可复现的漏报、误报和兼容性问题。最有帮助的反馈包含评审范围、实际使用的模型、最小代码示例，以及预期与实际结论。提交前请移除密钥和私有业务数据。

流程的完整定义在 [SKILL.md](SKILL.md)，展示名称和默认提示在 [agents/openai.yaml](agents/openai.yaml)。

## 致谢

感谢 [Claude Code 的 code-review 插件](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review) 提供的流程灵感。本项目围绕 Codex 的代理调度、模型分工和本地评审场景编写。

## 开源协议

[MIT](LICENSE) © 2026 jerry_yang
