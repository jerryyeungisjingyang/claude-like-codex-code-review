# Codex 代码审查 Skill — 借鉴 Claude Code 的多智能体评审流程

**在 OpenAI Codex 中执行四路独立代码评审，并逐项核实候选问题的证据。**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Codex-Agent_Skill-111827)](SKILL.md)

[English](README.md) | [简体中文](README.zh-CN.md)

支持审查未提交改动、分支差异、PR 和完整模块。两路评审检查项目规范，两路查找 bug；候选问题再交给独立代理核实、寻找反证。结果包含代码位置、触发条件和实际影响。

适合希望在 Codex 中使用 Claude Code 风格代码评审流程的开发者，也支持对照 `AGENTS.md`、`CLAUDE.md` 或技术需求文档（TRD）检查实现。默认只读，评审结果直接返回聊天。

**开始使用：**[安装 skill](#1-安装)，在 Codex 中打开要评审的仓库，然后运行：

```text
/claude-like-codex-code-review 审查当前工作区所有未提交改动。
```

完整的多智能体评审流程需要支持子代理调度及模型选择的 Codex 环境。兼容性要求及降级方式见[模型与用量](#模型与用量)。

[快速开始](#快速开始) · [评审流程](#评审流程) · [模型与用量](#模型与用量) · [常见问题](#常见问题)

## 为什么做这个项目

一次有用的 code review，应该回答三个问题：**哪里会出错、什么情况下触发、证据在哪里。**

这个 skill 把这些要求放进执行流程：

- **四路独立发现。** 两路 Sol low 检查规则，两路 Sol high 查找 bug。
- **发现之后，再找反证。** 候选问题交给新的代理复核，检查已有保护措施和触发前提。
- **只报告达到阈值的问题。** 合并重复根因，保留置信评分至少 80 的发现，附上位置、触发条件和影响。
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
```

以上命令会安装到用户级 skills 目录；已有同名版本时会更新它。重新开启 Codex 会话，在技能列表中确认 `claude-like-codex-code-review` 已被发现。

完整流程需要支持子代理调度及模型选择的 Codex 环境。当前模型配置面向提供 `gpt-5.6-luna`、`gpt-5.6-sol` 的环境；其他环境的降级方式见下文。

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
/claude-like-codex-code-review 对照 docs/requirements/，全量审查 src/example-service/，没有对比分支，重点看状态流转、幂等和异常恢复。
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
两路规则检查（Sol low）· 两路 bug 检查（Sol high）
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
| 项目规范 × 2 | 各自独立检查适用的 `AGENTS.md`、`CLAUDE.md`，以及用户指定的 TRD。 |
| bug 检查 × 2 | 各自独立查找功能错误，核实触发条件、影响与已有保护措施。 |

历史变更、历史 PR 评论和代码注释按需作为证据，不再单独设置必跑的评审路线。

四路使用独立上下文，按环境的并发上限分批执行。发现阶段不传递其他评审者的结论；每个去重后的候选由独立的 Luna medium 代理读取代码、寻找反证并评分一次，不追加 Sol 二次复核。

**80 分是评审阈值，不是“80% 的真实概率”。** 评分表示证据充分程度，严重程度单独判断。没有达到阈值的问题时，也会明确返回结果和覆盖限制。

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

| 工作 | 默认模型 | 推理强度 |
| --- | --- | --- |
| 范围整理、规范路径收集、摘要、最终基线检查 | GPT-5.6 Luna | medium |
| 两路独立规则检查 | GPT-5.6 Sol | low |
| 两路独立 bug 检查 | GPT-5.6 Sol | high |
| 候选问题独立评分 | GPT-5.6 Luna | medium |

主协调者保持当前会话模型。如果当前使用 Astra，任务调度和最终汇总仍由 Astra 完成，子代理按上表选择模型。希望协调部分也使用 Sol，可以在调用前切换会话模型。

多代理会产生额外的代码读取、推理和汇总消耗。分级模型用于控制分工成本，不承诺固定节省比例，也不代表总 token 数比单代理更少。

模型 ID 和调度规则写在 [SKILL.md](SKILL.md) 中。指定模型不可用时，优先在可用的 Luna / Sol 中替代；全部不可用或缺少代理工具时，说明限制并降级为当前模型顺序检查，不将降级结果称为四路独立评审。

## 常见问题

### 会自动改代码、写 Markdown 或创建 PR 吗？

默认不会。独立调用只授权读取和聊天回答，也不默认运行测试、构建或安装依赖。附加动作需要用户明确要求，这个边界同样适用于子代理。

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
