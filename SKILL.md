---
name: tiktok-ad-toolkit
description: Vidau TikTok 广告智能工具集，包含广告创建(creation)、数据巡检(inspection)、知识问答(knowledge)和投放报告(report)四大核心技能模块。适用于 https://tiktok.vidau.ai/ 平台的 AI 助手场景。
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [tiktok, ads, vidau, toolkit, ad-creation, inspection, knowledge, report]
    related_skills:
      - tiktok-hermes-ad-creation
      - tiktok-hermes-ad-inspection
      - tiktok-hermes-agent-knowledge
      - tiktok-hermes-report
---

# TikTok Ad Toolkit — Vidau 广告智能工具集

## 概述

本工具集是 Vidau 自研 TikTok 广告平台的完整 Skill 集合，包含以下 **4 个核心模块**：

| 子模块 | 路径 | 用途 |
|--------|------|------|
| `skills/creation` | [SKILL.md](skills/creation/SKILL.md) | TikTok 广告创建 — 闭环投放创建工作流 |
| `skills/inspection` | [SKILL.md](skills/inspection/SKILL.md) | TikTok 数据巡检 — 异常诊断与优化建议 |
| `skills/knowledge` | [SKILL.md](skills/knowledge/SKILL.md) | TikTok 知识库 — 总控路由与智能问答 |
| `skills/report` | [SKILL.md](skills/report/SKILL.md) | TikTok 投放报告 — 日报/周报自动生成 |

## 平台信息

- **主平台**: https://tiktok.vidau.ai/
- **AI 助手入口**: https://tiktok.vidau.ai/Ai 或默认首页
- **广告创建页面**: https://tiktok.vidau.ai/campaigns/ads
- **账户管理页面**: https://tiktok.vidau.ai/accounts

## 目录结构

```
tiktok-ad-toolkit/
├── SKILL.md                          # 工具集主入口（本文件）
├── README.md                         # 使用说明文档
├── references/
│   └── setup.md                      # 安装与配置指南
├── assets/
│   └── icon.svg                      # 工具集图标
└── skills/
    ├── creation/                     # 广告创建模块
    │   ├── SKILL.md
    │   └── references/
    │       └── source_and_validation.md
    ├── inspection/                   # 数据巡检模块
    │   ├── SKILL.md
    │   └── references/
    │       └── inspection_workflow.md
    ├── knowledge/                    # 知识库模块
    │   ├── SKILL.md
    │   └── references/
    │       └── source_and_validation.md
    └── report/                       # 投放报告模块
        ├── SKILL.md
        └── references/
            ├── report_contract.md
            ├── report_workflow_cn.md
            └── test_scenarios.md
```

## 核心原则

### 数据真实性
- **禁止编造**任何业务结果或性能指标（花费、转化、CPA、ROAS 等）。
- 缺失字段必须标记为 `unknown` / `insufficient_data`，不得编造。
- 仅当后端工具或页面状态明确确认时，才能声明操作成功。

### 风险控制
- 高风险操作（提交发布、修改预算/出价、暂停删除、外部通知等）**必须用户明确确认**。
- 非 Vidau 开通的账户即使已授权，也仅支持数据查看/报表/监控，不支持创建/修改/发布广告。
- UI 点击同一语义目标最多 2 次，每次点击后必须读取实际页面状态。

### 输出规范
- Hermes AI 助手模式下输出**有效 JSON**，不使用 Markdown 包裹。
- 用户可见内容放入 `assistant_display`（knowledge）或 `ui_payload`（report/inspection）。
- 禁止向普通用户暴露完整原始 JSON 或内部实现细节。

## 使用方式

1. 将本工具集上传至 Hermes 管理页面。
2. 用户在 `https://tiktok.vidau.ai/` 的 **AI助手** 面板中输入自然语言请求。
3. Hermes 根据用户意图自动路由到对应的子模块：
   - 创建/搭建广告 → `skills/creation`
   - 查看数据/巡检/异常诊断 → `skills/inspection`
   - 知识问答/开户/授权/导航 → `skills/knowledge`
   - 生成日报/周报 → `skills/report`

## 模块间协作

- **创建成功后** → 自动询问是否开启巡检 (`inspection`)
- **知识库** 作为总控路由器，负责意图识别和子模块调度
- **报告模块** 依赖 `knowledge` 模块的账户选择和数据同步规则
- 所有模块共享统一的数据验证原则和 JSON 输出协议

## 相关资源

- 详细安装配置请参阅：[references/setup.md](references/setup.md)
