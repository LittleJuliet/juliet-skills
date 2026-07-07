---
name: review-completed-work
description: Use after a requirement, idea, task, feature, bug fix, or issue investigation has been completed and Codex needs to review the resulting code changes against available requirements, plans, bug reports, issue context, and project standards. Review current uncommitted changes by default, or a user-specified git range; if no requirement, bug, or plan context exists, skip spec matching and perform code-quality review only.
---

# Review Completed Work

对已经完成的需求、想法、任务、feature、bug 修复或问题处理做只读 code review。默认检查当前未提交改动；如果用户给了 base/head/ref/range，就按用户指定范围检查。

开场先说明：

> 我正在使用 `review-completed-work`。本轮只做只读 code review，不修改代码。

## 流程

1. **确定 diff 范围**
   - 用户给了 base/head/ref/range：解析并校验该范围。
   - 用户没给范围：默认使用当前未提交改动，覆盖 `git diff HEAD`。
   - 如果默认范围为空，只问一个问题，让用户指定要 review 的范围。
   - 校验命令优先用：`git status --short`、`git rev-parse`、`git diff --stat`、`git diff`、`git log`。

2. **收集需求或问题上下文**
   - 优先使用用户直接提供的需求、bug、问题描述、计划、PRD、issue 文本或文件路径。
   - 再查仓库内明显匹配当前任务或分支的 `docs/`、`specs/`、`.scratch/`、计划文件、bug investigation 文档。
   - commit message 或 branch name 里的 issue/任务引用只能作为辅助线索。
   - 如果没有明确上下文，不追问；跳过需求/问题匹配轴，只做代码质量 review，并在最终结果里说明原因。

3. **收集项目标准**
   - 查 `AGENTS.md`、`README`、`CONTRIBUTING`、`CODING_STANDARDS`、附近模块文档和已有测试模式。
   - repo 明确规则优先；没有规则时，使用下面的基础 smell baseline。
   - 跳过已经由 formatter、linter、type checker 稳定覆盖的纯样式问题。

4. **分配只读 subagent**
   - 始终派一个代码质量 review subagent。
   - 只有存在明确需求、bug、计划或 issue 上下文时，再派一个需求/问题匹配 review subagent。
   - subagent 未完成前必须等待，不能提前中断或汇总。
   - 每个 subagent 都必须只读：禁止改工作区、index、HEAD 或 branch；只能用 `git diff`、`git show`、`git log`、`rg`、测试/构建命令检查。

5. **聚合结果**
   - findings first，按 severity 排序。
   - 不自动修复 review 建议；用户要求修复时，再按 `receiving-code-review` 的方式逐条验证后处理。

## Subagent Prompts

### 代码质量 review

把下面内容作为 prompt，补齐实际范围、标准来源和 diff 命令：

```text
你是 Senior Code Reviewer。请只读 review 已完成改动的代码质量、项目标准符合度、测试覆盖和生产风险。

## 范围

<写入 base/head/ref/range；默认范围写为 current uncommitted changes against HEAD>

## 可用命令

<写入 git diff --stat / git diff / git log 等命令>

## 项目标准来源

<列出已找到的 AGENTS.md、README、CONTRIBUTING、CODING_STANDARDS、附近模块文档、测试模式；如果没有，写 none found>

## 基础 smell baseline

- Mysterious Name：名称不能表达用途。
- Duplicated Code：同形逻辑重复出现。
- Feature Envy：函数过度读取别的对象数据。
- Data Clumps：同一组字段或参数反复一起传递。
- Primitive Obsession：用 primitive/string 伪装领域概念。
- Repeated Switches：同一类型上重复出现相同分支。
- Shotgun Surgery：一个逻辑变化分散修改太多位置。
- Divergent Change：一个模块被多个无关原因修改。
- Speculative Generality：为未出现的需求添加抽象、参数或 hook。
- Message Chains：调用方依赖过长访问链。
- Middle Man：只做转发的薄封装。
- Refused Bequest：继承关系中大量拒绝父级行为。

repo 明确规则优先；baseline 都是 judgement calls，不是硬性违规。

## 只读规则

禁止修改文件、index、HEAD、branch。禁止运行 formatter 或 fixer。可以运行只读检查、测试和构建命令；如果命令会重写 repo-tracked 文件，先不要运行。

## 输出

按 severity 输出 findings：

- Critical：bug、安全、数据丢失、核心功能坏掉。
- Important：需求缺口、错误处理、架构、测试覆盖、兼容性风险。
- Minor：小规模可维护性、命名、文档或局部优化。

每条 finding 必须包含：
- severity
- file:line
- 问题
- 影响
- 建议修复

如果没有问题，明确说没有发现代码质量问题，并列出实际检查过的范围和剩余风险。
```

### 需求/问题匹配 review

只有存在明确上下文时才派这个 subagent：

```text
你是 Senior Code Reviewer。请只读 review 已完成改动是否满足需求、计划、bug 报告或问题上下文。

## 范围

<写入 base/head/ref/range；默认范围写为 current uncommitted changes against HEAD>

## 可用命令

<写入 git diff --stat / git diff / git log 等命令>

## 需求、bug 或计划上下文

<写入用户提供内容，或仓库中找到的文档路径与关键内容>

## 只读规则

禁止修改文件、index、HEAD、branch。禁止运行 formatter 或 fixer。可以运行只读检查、测试和构建命令；如果命令会重写 repo-tracked 文件，先不要运行。

## 检查重点

- 是否完整实现需求、计划或 bug 修复目标。
- 是否遗漏明确要求的行为、边界、验收标准或测试。
- 是否引入需求外行为、scope creep 或不必要复杂度。
- 是否把 bug 修在症状层而不是根因层。
- 是否存在看似实现但实际不符合上下文的地方。

## 输出

按 severity 输出 findings：

- Critical：核心需求未达成、bug 未真正修复、安全或数据风险。
- Important：部分需求遗漏、边界条件缺失、测试不能证明修复。
- Minor：文案、文档、命名或验收材料的小缺口。

每条 finding 必须包含：
- severity
- file:line
- 对应需求/问题依据
- 问题
- 影响
- 建议修复

如果没有问题，明确说没有发现需求/问题匹配问题，并列出实际检查过的上下文和剩余风险。
```

## 输出格式

最终回复使用这个结构：

```markdown
## 需求/问题匹配

<如果无上下文，写：未运行。原因：没有找到明确需求、bug、计划或 issue 上下文，本次按约定只做代码质量 review。>

## 代码质量

<findings first；没有问题就明确说明>

## 验证

<列出实际运行的只读命令、测试、构建或无法运行的原因>

## 结论

<Ready / Ready with fixes / Not ready；1-2 句技术判断>
```

不要用“看起来不错”代替证据。不要把 nitpick 标成 Critical。不要在 review 过程中修改代码。
