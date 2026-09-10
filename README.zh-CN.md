# Codex 代码审查 Skill — 借鉴 Claude Code 的多智能体评审流程

**在 OpenAI Codex 中按可选强度执行代码评审，并支持修复与 GitHub PR 行内评论。**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Codex-Agent_Skill-111827)](SKILL.md)

[English](README.md) | [简体中文](README.zh-CN.md)

支持审查未提交改动、分支差异、PR 和指定路径。`low` 到 `max` 控制评审深度、代理数量和复核强度；`--fix` 应用修复，`--comment` 将结果发布为 GitHub PR 行内评论。结果包含代码位置、触发条件和实际影响。

适合希望在 Codex 中使用 Claude Code 风格代码评审流程的开发者，也支持对照 `AGENTS.md`、`CLAUDE.md` 或技术需求文档（TRD）检查实现。默认只读，评审结果直接返回聊天。

**开始使用：**[安装 skill](#1-安装)，在 Codex 中打开要评审的仓库，然后运行：

```text
$claude-like-codex-code-review [low|medium|high|xhigh|max] [--fix] [--comment] [<pr#>|<branch>|<path>]
```

完整的多智能体评审流程需要支持子代理调度及模型选择的 Codex 环境。兼容性要求及降级方式见[模型与用量](#模型与用量)。

[快速开始](#快速开始) · [评审流程](#评审流程) · [模型与用量](#模型与用量) · [常见问题](#常见问题)

## 为什么做这个项目

一次有用的 code review，应该回答三个问题：**哪里会出错、什么情况下触发、证据在哪里。**

这个 skill 把这些要求放进执行流程：

- **按强度扩展评审。** `low` 为单模型快速检查，默认 `high` 使用四路独立评审，`xhigh` 和 `max` 扩展搜索与复核。
- **发现之后，再找反证。** 候选问题交给新的代理复核，检查已有保护措施和触发前提。
- **只报告达到当前档位阈值的问题。** 合并重复根因，按档位过滤证据不足的发现，附上位置、触发条件和影响。
- **能审改动，也能审整个模块。** 全量模式会检查已有缺陷；明确指定 TRD 时，对照业务约束审查实现。
- **默认只读，结果回到聊天。** 修改代码、保存报告、创建 PR、发布评论和运行测试，都需要明确的附加要求。

## 快速开始

### 1. 安装

克隆本仓库，并在仓库根目录安装 skill：

```bash
git clone https://github.com/jerryyeungisjingyang/claude-like-codex-code-review.git
cd claude-like-codex-code-review
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review"
cp SKILL.md "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/SKILL.md"
cp -R agents "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
cp -R references "${CODEX_HOME:-$HOME/.codex}/skills/claude-like-codex-code-review/"
```

以上命令会安装到用户级 skills 目录；已有同名版本时会更新它。重新开启 Codex 会话，在技能列表中确认 `claude-like-codex-code-review` 已被发现。

完整流程需要支持子代理调度及模型选择的 Codex 环境。当前模型配置面向提供 `gpt-5.6-luna`、`gpt-5.6-sol` 的环境；其他环境的降级方式见下文。

### 2. 开始评审

在需要评审的代码仓库中调用 Skill。所有参数都可省略，未指定强度时默认为 `high`：

```text
$claude-like-codex-code-review [low|medium|high|xhigh|max] [--fix] [--comment] [<pr#>|<branch>|<path>]
```

**检查当前未提交的改动：**

```text
$claude-like-codex-code-review low
```

**检查当前分支相对默认基线的改动：**

```text
$claude-like-codex-code-review high
```

**检查指定分支：**

```text
$claude-like-codex-code-review high feature/login
```

**深度检查指定目录：**

```text
$claude-like-codex-code-review xhigh src/example-service/
```

需要对照 TRD 或强调特定风险时，可在命令后继续用自然语言说明；无法解析为档位、flag 或 target 的内容会作为评审标准，而不是位置参数。

**评审已有 PR：**

```text
$claude-like-codex-code-review high 123
```

**评审 PR 并发布行内评论：**

```text
$claude-like-codex-code-review medium --comment 123
```

**评审并修复本地代码：**

```text
$claude-like-codex-code-review high --fix src/auth/
```

PR 数据通过环境中已有的 GitHub 连接器或 `gh` 读取。缺少权限或历史数据时，评审会说明未覆盖的部分。

### 3. 参数说明

| 参数 | 作用 |
| --- | --- |
| `low` | 单模型快速检查，最多返回 4 项直接证实的问题。 |
| `medium` | 两路独立检查，逐项复核，最多返回 6 项。 |
| `high` | 默认档位；四路独立检查，逐项复核，最多返回 10 项。 |
| `xhigh` | 六路发现、双重复核，最多返回 15 项。 |
| `max` | 八路发现、双重复核和缺口扫描，最多返回 20 项。 |
| `--fix` | 在评审后修改本地工作区并运行范围合理的现有验证；不提交、不推送。 |
| `--comment` | 将保留的问题逐项发布为目标 GitHub PR 的行内评论。 |
| `PR / branch / path` | 只允许一个目标，例如 `123`、`feature/login` 或 `src/auth/`。 |

`--comment` 必须能唯一解析到一个开放 PR。目标不是 PR 时，Skill 会尝试从目标分支或当前分支解析 PR；无法唯一确定时会停止并要求 PR 编号。

保存报告、创建提交、推送或创建 PR 仍需单独明确要求。`--fix` 与 `--comment` 只授权各自对应的动作。

## 评审流程

```text
确认范围与代码版本
        ↓
收集规范 · 整理模块摘要
        ↓
按 low / medium / high / xhigh / max 调度发现代理
        ↓
合并重复根因
        ↓
独立复核每个候选 · 主动寻找反证
        ↓
按当前档位阈值保留发现
        ↓
复核代码版本 · 在聊天中返回结论
```

| 默认 `high` 评审角度 | 关注的问题 |
| --- | --- |
| 项目规范 × 2 | 各自独立检查适用的 `AGENTS.md`、`CLAUDE.md`，以及用户指定的 TRD。 |
| bug 检查 × 2 | 各自独立查找功能错误，核实触发条件、影响与已有保护措施。 |

历史变更、历史 PR 评论和代码注释按需作为证据，不再单独设置必跑的评审路线。

默认 `high` 使用四路独立上下文，按环境并发上限分批执行。`medium` 减少到两路，`xhigh` 和 `max` 增加 bug 搜索角度并使用双重复核。发现阶段不传递其他评审者的结论。

评分表示证据充分程度，不是“真实概率”，严重程度单独判断。`high`、`xhigh` 和 `max` 的最低分为 80；`medium` 提高到 90；`low` 只报告可以直接证实的问题。没有问题达到阈值时，也会明确返回结果和覆盖限制。

## 你会得到什么

以下为输出结构示例，不是本项目的实测结果：

```text
[P1] 已过期的会话仍然有效

位置：src/session.py:42
触发：请求携带的会话已经超过有效期。
影响：本应被拒绝的请求仍然获得访问权限。
证据：过期校验使用 now < expires_at，而非 now >= expires_at。
复核：授予访问权限前，没有其他过期校验。
评分：90
验证：静态调用链核查，未运行测试。
```

每项发现包含可定位的文件链接、触发条件、实际影响与关键证据。测试是否执行、哪些数据无法访问，会如实说明。

## 模型与用量

| 档位 | 发现流程 | 复核流程 |
| --- | --- | --- |
| `low` | 当前协调模型单路检查 | 协调模型仅保留直接证实的问题 |
| `medium` | Sol：规则 low ×1、bug medium ×1 | Luna low ×1 |
| `high` | Sol：规则 low ×2、bug high ×2 | Luna medium ×1 |
| `xhigh` | Sol：规则 medium ×2、bug xhigh ×4 | Luna high ×2 |
| `max` | Sol：规则 high ×2、bug max ×6，追加缺口扫描 | Luna max ×2 |

主协调者保持当前会话模型。如果当前使用 Astra，任务调度和最终汇总仍由 Astra 完成，子代理按上表选择模型。希望协调部分也使用 Sol，可以在调用前切换会话模型。

多代理会产生额外的代码读取、推理和汇总消耗。分级模型用于控制分工成本，不承诺固定节省比例，也不代表总 token 数比单代理更少。

模型 ID 和调度规则写在 [SKILL.md](SKILL.md) 中。指定模型不可用时，优先在可用的 Luna / Sol 中替代；全部不可用或缺少代理工具时，说明限制并降级为当前模型顺序检查，不将降级结果称为四路独立评审。

## 常见问题

### 会自动改代码、写 Markdown 或创建 PR 吗？

默认不会。`--fix` 明确授权修复本地代码，`--comment` 明确授权发布 PR 评论；两者都不授权提交、推送或创建 PR。未提供 flag 时只读取并在聊天中回答。

### 在已有的长对话中使用，会受之前结论影响吗？

主协调者仍能看到当前对话。skill 要求子代理以 `fork_turns: "none"` 启动，只接收评审范围、规范路径和必要资料，减少继承既有判断的影响。这不能完全消除协调者在选取资料和汇总时的偏差。希望进一步减少历史上下文影响，可以在新会话中给出明确范围后调用。

### 能保证没有 bug，或者与 Claude 的结果一致吗？

不能。模型、代码范围和可访问证据都会影响结果。独立复核用于过滤误报；静态评审仍可能遗漏运行时问题，也不能替代测试和人工审查。

### 必须有 PR 才能用吗？

不需要。可以检查工作区改动、分支差异或指定模块的全量代码。所需历史 PR 证据无法获取时，会说明覆盖限制。

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
