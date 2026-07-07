---
name: project-bug-investigation
description: Use when the user reports a bug, broken behavior, failing workflow, confusing project issue, or asks why something is not working; investigate the project deeply, build or request evidence, ask at most one blocking question at a time, and stop only when the root cause or a concrete blocker is identified.
---

# Project Bug Investigation

找出项目 bug 或异常行为的根因。默认只做调查和证据收集，不直接改代码。

## 核心规则

- 先调查，再下结论。没有证据支撑根因或 blocker 前，不提出修复方案。
- 项目事实自己查。只有本地无法发现或无法安全验证的信息，才问用户。
- 每次最多问一个阻塞问题。问题必须是最能改变下一步调查路径的那个。
- 不猜测。证据不足时，明确缺少哪一个 artifact、权限或环境。
- 默认不编辑文件。只有用户明确要求或批准时，才添加临时 instrumentation 或实施修复。

## 问题接收

把用户报告整理成一个具体调查目标：

- 现象：用户实际看到什么。
- 期望行为：本来应该发生什么。
- 实际行为：报错、错误输出、变慢路径、缺失数据或失败流程。
- 上下文：日志、截图、时间点、账号、环境、输入、payload、版本或最近发布。
- 影响范围：影响谁或什么，以及是否可复现。
- 已尝试动作：已经试过的修复、绕过方案、命令或检查。

如果用户已经提供足够线索，直接开始调查，不要先问模板化问题。

## 项目调研

为相关项目区域建立最小但有用的地图：

- 先读本地说明：`AGENTS.md`、`CONTEXT.md`、README、ADR 和附近文档。
- 定位和现象相关的入口、模块、配置、测试、命令和日志。
- 优先用 `rg` 搜索错误文案、UI 标签、路由名、API 路径、命令、函数名、配置 key 和 schema 字段。
- 可用时，用 `git diff`、`git status` 和相关 commit 检查最近本地变化。
- 项目较大时，把只读调查拆成独立路径，例如代码链路、测试覆盖、日志和近期变更；依赖 delegated work 前必须等待其完成。

## 反馈闭环

创建或找到一个能捕获当前 bug 的紧反馈信号：

- 如果允许编辑，优先使用已有 failing test，或最小的新测试 seam。
- 否则使用最窄的可运行闭环：curl、CLI command、browser automation、trace replay、throwaway harness、differential run 或 bisect command。
- 闭环应当 red-capable、确定性强；对于 flaky bug，要有高复现率；同时足够快、可由 agent 自动运行。
- 可行时至少运行一次闭环，并记录命令和关键输出。
- 如果无法建立闭环，列出已尝试内容，并请求唯一缺失的 artifact 或权限：log dump、HAR、payload、带时间戳的录屏、数据样本、账号、环境，或添加临时 instrumentation 的许可。

## 证据地图

深入单个组件前，先追踪真实执行链路：

- 写出链路，例如 UI -> API -> service -> database，command -> parser -> executor，或 CI -> build -> deploy。
- 在每个边界记录观察到的 input、output、config、environment、state 和 error shape。
- 将节点标记为 confirmed、excluded 或 unverified。
- 对多组件系统，先定位 data、state 或 behavior 首次偏离的边界，再调查这个边界。

## 假设

对非平凡 bug，生成 1-5 个候选原因，并按证据强弱排序。

每个假设都必须可证伪：

```text
如果 <原因> 成立，那么 <探针或改变条件> 会让 <可观察结果> 发生。
```

一次只测试一个预测。如果用户的领域知识会显著改变排序，只问一个聚焦问题，得到回答后继续。

## 追踪根因

不要停在报错出现的位置。向后追踪，直到找到最初触发点：

- 哪段代码直接产生了现象？
- 是谁调用了它，传入了什么？
- 错误 input、state、config、permission、environment 或 timing condition 最早在哪里出现？
- 哪个更早的步骤本应产生不同的值或 invariant？
- 候选根因能否同时解释现象、反馈闭环结果和证据地图？

根因是系统第一次变错的位置，不是最深处那个发现错误的 stack frame。

## 输出格式

找到原因时，输出：

- 根因：一句精确描述。
- 证据链：3-6 条观察如何指向该原因。
- 复现或验证：可用时给出 command、test、script、request 或 manual path。
- 关键排除项：只列出已排除的重要候选。
- 建议修复点：file、function、module、config、data 或 external owner；保持聚焦。
- 剩余风险：只在仍有重要内容未验证时列出。

尚未找到原因时，输出：

- 当前 blocker：唯一缺失的 artifact、权限、环境或决策。
- 已经检查过什么。
- 下一个单一问题或请求的 artifact。

## 停止条件

满足以下条件时停止调查：

- 已找到根因，并且有证据支撑。
- 问题已隔离到 external system、缺失权限、环境缺口，或只有用户侧数据才能继续调查。
- 下一步只需要用户提供一个 artifact 或决策。
- 用户要求从调查切换到实施。
