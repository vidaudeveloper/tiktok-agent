---
name: trend-viral-short
description: 爆款短视频 —— 批量 hook 图片变体，专为 TikTok/Reels 产品广告设计。★ TikTok Ads Specialist 首选创作 Skill
metadata:
  layer: L2-vertical
  requires: [creative-job-runner, creative-platform, creative-gpt-image2-prompt, creative-seedance2-prompt, creative-script2film, creative-script2film-keyframes, creative-direct]
  tags: [trend, batch, ecommerce, image, tiktok-ads]
---

# Trend Viral Short — TikTok 爆款短视频/图片变体

追热点；快速产出多个竖版 **图片** 变体用于 A/B 测试。

> **Prompt Gate**：在 `creative_submit_batch_variants` 前加载 **creative-gpt-image2-prompt** — craft base prompt + variant hooks。视频路径则加载 **creative-seedance2-prompt**。

## 使用时机

- MCN 每日投递、追热点
- 同产品、多个 opening-hook 测试（**图片**变体）
- TikTok 信息流广告素材矩阵
- 需要大量快速测试不同创意方向

## 当用户想要"视频交付物"

本 Skill 默认为 **批量图片**。如需视频，按意图切换 L1 Skill：

| 需求 | Skill | MCP |
|------|-------|-----|
| 多镜头产品短 + 参考 | creative-script2film | `creative_submit_script2film` |
| 多镜头 + keyframe 转场 | creative-script2film-keyframes | `creative_submit_script2film_keyframes` |
| 单个趋势短 clip | creative-direct | `creative_generate_video` / `creative_first_frame_to_video` |

提交前确认意图 —— 不要对视频请求默认走 `batch_variants`。

## Flow（图片变体）

1. 整理 brief：`product`、`trend_tags`（热点关键词）、`hook_idea`（可选）
2. **加载 creative-gpt-image2-prompt** — 产出生产级 base `prompt`（+ variant hook 子句，如需要）
3. `creative_estimate` workflow_type=`batch_variants`, params=`{ count: 5 }`
4. `creative_submit_batch_variants`：
   - `prompt`：**creative-gpt-image2-prompt 的输出**（不是用户原始文本）
   - `count`：默认 **5**
   - `aspect_ratio`：**9:16**（TikTok 竖版）
   - `preset`：**trend_viral_v1**
5. **creative-job-runner** — 立即 push `tracking.user_message`；**不要** sleep/polling
6. 按 variant 编号列出 artifacts；建议投放优先级

## Preset 约束（trend_viral_v1）

- 前 3 秒强视觉 hook
- 产品特写 ≤ 画面 40%
- 无侵权热点、无敏感内容
- 竖版 9:16（TikTok 信息流原生比例）
- 适合轮播/信息流广告位展示

## ★ 与 TikTok Ads 的衔接

本 Skill 产出的 **batch image variants** 可直接用于：

1. **A/B 测试素材矩阵**：同一产品 5 个不同 hook 方向 → 分别创建 5 条 Ad → 对比 CTR/CVR
2. **Cover Image 批量生成**：为多个 Ad 生成不同封面图 → `create_ad.cover_url`
3. **落地页配图**：批量产出后供落地页使用

衔接流程：
```
trend-viral-short 产出 N 个 download URLs
    ↓
用户选择要使用的变体
    ↓
Ads Skill: create_ad({ cover_url: <选中URL>, ... })
    或批量创建 N 条 Ad（每个变体一条）
```
