---
layout: post
title: 我的AI开发工作流
date: 2026-06-28 00:00:00
author: admin
comments: true
categories: [AI, Productivity]
tags: [AI Agent, 工作流, 自动化, 想法管理]
---

我最近高强度使用 AI 进行编程，工作流经历了几次迭代，发出来分享一下。

<!-- more -->

---

* 目录
{:toc}

---

## 背景

我正在用 AI 开发一个软件。和很多人一样，一开始我的工作方式很简单：想到一个 idea，告诉 AI，让它写代码。但随着项目变复杂，这个方式很快就撑不住了。于是我开始不断调整和 AI 协作的流程，前前后后经历了三个阶段。

这条进化线大概是这样的：

**第一阶段：Plan and Execution + Spec Driven。** 先让 AI 做计划，再执行；后来更进一步，先写规格说明（Spec），再让 AI 按规格生成代码。这一步解决的是"AI 不知道要做什么"的问题。

**第二阶段：Matt Pocock 的七阶段工作流。** 我学到了 Matt Pocock 开源的一套 Agent Skills，把整个开发流程从 idea 到交付串了起来。grill 需求、写 PRD、拆 issue、TDD 实现、诊断、验收，每一步都有对应的 skill。这一步解决的是"流程太松散、质量不可控"的问题。

**第三阶段：加两个新 skill，解决瓶颈。** Matt 的流程很好，但我用了一段时间后发现一个新问题：grill 这一步太重了。我有很多 idea，每个都去 grill 一遍根本来不及。于是我加了两个新 skill（`priority-triage` 和 `idea-review-packet`），在 grill 之前做一层轻量分流。这一步解决的是"想法太多、讨论太贵"的问题。

下面来详细讲讲每个阶段。

## 第一阶段：Plan and Execution 与 Spec Driven

最早我用的是最简单的 Plan and Execution 模式。流程大概是：描述需求 → AI 出计划 → 我确认 → AI 执行。这在 AI 编码工具里很常见，比如 Cursor 的 Plan Mode、Claude Code 的 `/plan` 都是这个思路。思路就是把"想"和"做"分开：先让 AI 分析代码库、拆解任务、生成步骤，确认之后再动手写代码。

但 Plan and Execution 的问题在于，计划的质量取决于 prompt 的质量。你说得越模糊，AI 计划得越跑偏。于是很自然地，我开始用 Spec Driven（规格驱动开发）：先写一份结构化的规格说明，定义清楚需求、边界、验收标准，然后才让 AI 生成代码。像 GitHub Spec Kit、Kiro 这些工具都是这个思路。

这一步把我的开发质量提升了不少。但 Spec Driven 也有它的问题：把 spec 文档当开发品去维护，成本很高。一份详尽的 spec 写出来动辄几千字，人的认知带宽是有限的。仔细看，很累；不仔细看，那和 vibe coding 也没区别。而且它只管"单个功能怎么做"，没有管"整个项目怎么做"。

## 第二阶段：Matt Pocock 的七阶段工作流

把整个流程串起来的，是 Matt Pocock 开源的一套 Agent Skills。他是一位 TypeScript 专家，把自己日常和 Claude Code 协作的流程拆成了 21 个可复用的 skill，每个 skill 解决一个具体环节的问题。

他的流程是七个阶段，我画了一个图：

```text
Idea
  ↓
Grill（深度追问）
  ├─ grill-me：通用需求深访
  └─ grill-with-docs：工程级深访，更新 CONTEXT.md 和 ADR
  ↓
Prototype（快速原型验证）
  ↓
to-prd（把讨论结果写成 PRD）
  ↓
to-issues（把 PRD 拆成可独立执行的 vertical slice issues）
  ↓
Sandcastle Execution（AFK 自动执行）
  ↓
Review / QA（自动 review + 人工验收）
  ↓
反馈进入下一轮
```

这个流程有几个设计值得说一下：

- **Grill 是核心。** 在写任何代码之前，AI 会像一位挑剔的架构师一样连续追问，从功能范围、边界条件、错误处理到测试策略，直到所有模糊地带被扫清。
- **PRD 是"目的地文档"。** 不是写一份厚重的需求文档，而是把 grill 过程中达成的共识总结成一份简洁的 PRD，包含问题描述、解决方案、用户故事和验收标准。
- **Issue 按 vertical slice 拆分。** 不是按水平层（先做后端、再做前端、再做测试），而是每个 issue 都是一个端到端的功能切片，能独立验证。
- **执行用 Sandcastle。** 把 issue 喂给 agent，让它自动实现、跑测试、修 bug，人只需要在关键节点验收。

更详细的内容到youtube 上搜素他的名字查看他的完整视频。

我把这套流程用起来之后，开发效率确实上了一个台阶。代码质量更稳定了，需求偏差更少了，AI 产出的东西也更可预期了。

但用了一段时间，我发现了一个新的瓶颈。

## 第三阶段：Grill 瓶颈和两个新 Skill

Matt 的流程里，grill 是几乎所有事情的起点。`grill-me` 或 `grill-with-docs` 会连续追问几十个问题，一次深度讨论动辄半小时。这一步很有价值，它确实能帮你把模糊的想法搞清楚。但问题是：我的 idea 太多了。

我每天会冒出各种想法，有的是新功能，有的是优化点，有的是 bug 修复，有的是架构重构。这些 idea 的质量参差不齐。有的价值很高但边界模糊，有的边界很清楚但价值一般，有的根本还没想清楚值不值得做。如果每个 idea 都进 grill，我一天大部分时间都在和 AI 讨论需求，代码一行没写。

这里的问题不在于 AI 不够聪明，而是流程缺少上游筛选。我需要在 grill 之前加一层轻量判断，把 idea 先分流。

于是我自己写了两个新 skill 来解决这个问题。

### Priority Triage：批量筛选

`priority-triage` 负责批量看我的 workbench（所有 idea 的存放地）。它不做深入设计，不写 PRD，也不重新实现想法。它只回答一个问题：**现在最值得推进的是哪些？每个候选应该走哪条路？**

它会根据价值、边界清晰度、风险、依赖关系，把候选分到五条路线：

| 路线 | 说明 |
|------|------|
| Direct PRD | 价值清楚、边界清楚、验收清楚，直接写 PRD |
| Deep Grill | 价值高，但边界、风险、术语或验收还需要深挖 |
| Need Info | 信息不足，需要先补最少关键事实 |
| Park | 价值不明、时机不对、范围过宽，先暂停 |
| Cleanup | 已完成、重复、过期，只需要整理 |

这样一来，grill 就从默认步骤变成了昂贵资源。只有真正需要深挖的候选才进入 `grill-with-docs`，其他候选要么直接进 PRD，要么先补信息，要么直接暂停。

### Idea Review Packet：单点决策材料

`idea-review-packet` 是夹在 `priority-triage` 和 `grill-with-docs` / `to-prd` 之间的一个技能。它的目标不是产出完整方案，而是产出一页高内聚的判断材料，让我能快速决定下一步。

一个好的 review packet 会回答这些问题：

- 它解决的具体问题是什么？
- 谁会受益？
- 当前系统已经有什么能力？缺口是什么？
- 它触碰哪些边界？
- 最小可做切片是什么？
- 如何证明它做成了？
- 如果要 Deep Grill，应该重点追问哪几个问题？

有了这两个 skill，我的流程变成了这样：

```text
Workbench Inbox
  ↓
Priority Triage（批量筛选，分路线）
  ↓
Idea Review Packet（单点决策材料）
  ↓
路线分流
  ├─ Park / Cleanup / Need Info
  ├─ Direct PRD → to-issues → Sandcastle Execution
  └─ Deep Grill → to-prd → to-issues → Sandcastle Execution
        ↓
      Verifier / Tests / QA
        ↓
      Human Acceptance
        ↓
      反馈进入 Workbench
```

这个流程最大的变化是：grill 从"每个 idea 都要走"变成了"只给高价值、高不确定性的 idea 走"。大部分 idea 在 `priority-triage` 和 `idea-review-packet` 这两层就被分流了，只有少数需要深挖的才进入深度讨论。

## 这和 Loop 有什么关系

最近 AI coding 领域经常讨论 loop。我的理解是，loop 的重点不是让 agent 无限运行，而是把一组重复发生的判断和动作变成可观察、可暂停、可验证的循环。

一个健康的 loop 不是这样的：

```text
idea -> agent 自动写代码 -> 我看结果
```

而是这样的：

```text
idea -> 筛选 -> 判断包 -> 规格化 -> 执行 -> 验证 -> 人类验收 -> 反馈进入下一轮
```

这个 loop 有几个关键条件。

**第一，它有状态。** 一个 idea 不是一段自由文本，而是处在某个阶段：captured、screened、reviewed、prd-ready、executing、verify-failed、accepted 或 parked。

**第二，它有预算。** 不是每个 idea 都值得深挖，不是每个失败都值得无限修复。比如一个候选最多 cheap review 一次，Deep Grill 最多两轮，执行失败最多修复三轮。超过预算就进入人工判断。

**第三，它有人类闸口。** 人不应该陪 agent 做所有重复劳动，但应该保留几个关键控制点：是否值得做、边界是否正确、验收标准是否成立、最终产物是否符合预期。

**第四，它有验收证据。** 执行结束后，不能只给我一个 diff。它应该给我一个 final acceptance packet，里面包含目标、改动范围、测试结果、风险、手动验收方式和建议结论。

## 展望：下一步的瓶颈可能是 QA

目前这套流程在我这里跑得还不错，但我已经能看到下一个瓶颈了：QA（质量验收）。

你想，当上游的筛选、规格化、执行都自动化之后，最花时间的环节就变成了"确认 AI 写出来的东西对不对"。现在我的 QA 还是比较手动的：看 diff、跑测试、手动点点功能。但随着 idea 吞吐量提高，QA 会越来越成为瓶颈。

我接下来想探索的方向有几个：

- 让 verifier 更智能，不只是跑测试，而是能对照 PRD 和验收标准逐项检查。
- 把人工 QA 的精力集中在"只能人判断"的部分。比如交互体验、视觉一致性、业务逻辑的正确性，这些交给人工；而"可以自动检查"的部分完全交给 agent。
- 引入 eval 和 golden case，让回归测试有据可依。

这可能需要再写一两个新 skill，或者把现有的 verifier 和 review 流程升级。流程优化这件事，大概没有终点。

## 总结

回头看这条工作流的进化，其实就是在做一件事：把判断成本从人转移到流程。

第一阶段加了 Plan 和 Spec，让 AI 知道要做什么。第二阶段引入了 Matt Pocock 的七阶段工程纪律，把松散的流程收紧。第三阶段在 grill 前面加了轻量分流，因为想法太多，深度讨论太贵。

以前和 AI 协作就是不断发 prompt，现在更像是在设计一条小型 idea 加工线。想法扔进去，经过筛选、规格化、验证，出来的是能直接判断要不要做的交付候选。我越来越觉得，长期效率不取决于某一次 AI 写得多快，而取决于这个系统能不能持续把模糊想法变成可靠结果。