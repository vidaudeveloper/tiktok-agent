---
name: creative-batch-orchestrator
description: 批量编排 —— 每批最多 10 项，混合 Skill/MCP，并行提交和追踪
metadata:
  layer: L1-capability
  requires: [creative-job-runner, creative-platform, creative-seedance2-prompt, creative-gpt-image2-prompt, creative-direct, creative-script2film, creative-script2film-keyframes]
  tags: [batch, orchestrator, multi-skill, video, async]
---

# Creative Batch Orchestrator

将**多个独立生成任务**归为一批。**允许混合 Skill**（script2film、keyframes、direct video、batch image variants 等）——并行提交、统一追踪和交付。

> **必须前置**：加载 **creative-job-runner**（多任务 UI 追踪）和 **creative-platform**（上传 + preflight）。
> **Prompt Gate**：每个涉及图片/视频 MCP 的 item，加载 **creative-gpt-image2-prompt** 或 **creative-seedance2-prompt** 并在提交前 craft prompt —— 不使用原始文本。
> **典型规模**：**10 项/批**（硬上限 10；更大请求请拆分）。

## 使用时机

- A/B 同产品跨渲染模式（reference vs keyframes vs direct）
- 多个产品/脚本一次运行
- 混合批量：3× script2film + 5× direct_video + 2× batch_variants images
- 运营日常投递：提交批量，后台追踪 —— 无需逐个等待

## 不适用时机

- 单个任务 → 直接使用匹配的 L1/L2 Skill，不需要 batch wrapper
- 相同 prompt、N 个图片变体 → **trend-viral-short** + `creative_submit_batch_variants`（一个 job）
- 用户要即时结果、无需任务列表 → **creative-direct** sync MCP（不在 batch 中）

## Batch manifest 格式

组织为 **items array**（1–10）：

```yaml
batch_label: "Summer sale batch-A"
items:
  - label: "SKU-A reference render"
    skill: creative-script2film
    input:
      script: "..."
      target_duration_sec: 30
      aspect_ratio: "9:16"
      reference_image_urls: ["https://..."]
      brief: { product: "...", audience: "..." }

  - label: "Hero image"
    skill: creative-direct-image
    input:
      prompt: "..."
      aspect_ratio: "9:16"

  - label: "Hook image variants x5"
    skill: trend-viral-short
    input:
      prompt: "..."
      count: 5
      aspect_ratio: "9:16"
      preset: trend_viral_v1
```

每个 item **必须有唯一 `label`**（用于交付表）。每个 item 生成唯一 `client_request_id`（UUID）——幂等，避免重复扣费。

## Skill → MCP 映射（严格按 submit）

| `skill` 字段 | MCP 工具 | workflow_type |
|-------------|----------|---------------|
| `creative-script2film` | `creative_submit_script2film` | `script2film` |
| `creative-script2film-keyframes` | `creative_submit_script2film_keyframes` | `script2film` |
| `creative-direct-video` | `creative_submit_workflow` | `direct_video` |
| `creative-direct-image` | `creative_submit_workflow` | `direct_image` |
| `trend-viral-short` | `creative_submit_batch_variants` | `batch_variants` |
| `product-url-to-video` | 爬取后 → 以上任意 L1 MCP | 按 `workflow` |

**所有 batch items 都是异步 job**（含 `job_id`）；通过 `creative_get_job` / `creative_list_jobs` / `creative_cancel_job` 追踪。

## 标准流程

### 1. 校验 batch
- `items.length` **1–10**；>10 → 拆分并告知用户
- 每个 `label` 非空；`skill` 在映射表中
- 缺失 script/prompt/URL → 询问用户；**不要**提交不完整的 items

### 2. 估算积分
- 每项调用 `creative_estimate`（对齐 workflow_type）
- 汇总表格供用户确认后提交

### 3. 并行提交
- **全部异步**：并行 fire 所有 `creative_submit_*` / `creative_submit_workflow`（每项唯一 `client_request_id`）
- 提交后发送摘要：`Submitted N async jobs` + 每个 `job_id`；告知用户可在本线程随时询问进度

### 4. 追踪
- **不要** sleep / `creative_get_job` 循环；按提交发送 `tracking.user_message`
- 用户询问时，**一次** `creative_list_jobs` 或对特定 `job_id` 调用 `creative_get_job`
- 全部终态时 → 交付 batch 结果表：

```
| # | label | skill | job_id | status | artifact |
|---|-------|-------|--------|--------|----------|
| 1 | SKU-A | script2film | uuid-1 | ✅ | https://... |
| 2 | Trend direct | direct_video | uuid-2 | ✅ | https://... |
```
