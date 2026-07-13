---
name: creative-script2film
description: 长视频多镜头（16s–120s）— 一句话→脚本→参考图生视频 + concat + BGM
metadata:
  layer: L1-capability
  requires: [creative-job-runner, creative-platform, creative-narrative-router, creative-seedance2-prompt]
  tags: [storyboard, async, script2film, reference, bgm, one-click, long-video, script]
---

# Creative Script2Film — 参考图模式多镜头视频

将脚本（或一句话 brief）转化为可交付的短视频。**默认 `video_mode=reference`**：每个镜头使用 Seedance reference-image-to-video（用户参考图 + per-shot keyframe）。

如需首/尾帧转场，请改用 **creative-script2film-keyframes**。

> **Prompt Gate**：在编写或丰富 Final Video Spec 镜头视觉描述前，加载 **creative-seedance2-prompt**。服务端从脚本生成 per-shot video prompts —— 丰富的电影化场景行可提高输出质量。对于此 workflow 以外的直接视频 MCP，Seedance prompt Skill 是**必须的前置**。

## 使用时机（路由优先级）

| 用户意图 | 使用 |
|----------|------|
| **长视频 >15s**（16s–120s）：30s/60s 产品广告、品牌片 | **本 Skill** |
| 一句话/卖点列表，尚无脚本 | **本 Skill**（从 `creative_generate_script` 开始） |
| 多镜头 + 产品/人物参考约束 | **本 Skill** → `creative_submit_script2film` |
| 多镜头 + 平滑镜头间转场（首/尾帧） | **creative-script2film-keyframes** |
| **单个 clip ≤15s**，一镜到底直出 | **creative-direct**（**不要**使用本 Skill） |

**路由规则**：用户说"30 秒视频"、"一分钟短视频"、"多镜头"、"分镜交付物"或 `duration > 15s` → 本 Skill。单段 ≤15s → **creative-direct**。

## Agent 流程（面向用户）

### 0. 收集输入（可以最少）

用户可能只给一句话，例如：

> "帮我做一个 30s 竖版 TikTok 无线耳机广告——突出 ANC 和 30 小时续航。"

脚本语言匹配用户的对话语言（EN 用户 → EN 脚本，ZH 用户 → ZH 脚本等）。

仅在缺失时询问（已提供则跳过）：产品、核心卖点、目标时长（**>15s 默认 30s**）、比例（默认 9:16）、参考图（产品/人才/场景）及每张图的用途。

### 1. 叙事路由（脚本生成前的必须步骤）

遵循 **creative-narrative-router**：

1. 解析意图 → 选 `narrative_structure`（`product_ad` / `story_narrative` / `problem_solution` / …）
2. **Read** 匹配的 `references/*.md`；将 beats 和 forbidden rules 写入 `brief.narrative`
3. 如置信度低，展示 **2–3 个不同结构**的选项；等待用户选择

### 2. 生成脚本（当用户无完整脚本时的必须步骤）

调用 **`creative_generate_script`**（不扣积分）：

```json
{
  "creative_request": "30s vertical TikTok ad for wireless earbuds — highlight ANC and 30h battery",
  "brief": {
    "product": "Wireless earbuds",
    "audience": "US commuters",
    "platform": "TikTok",
    "locale": "en",
    "narrative_structure": "product_ad",
    "secondary_structure": "problem_solution",
    "narrative": {
      "beats": ["pain_hook", "hero_product", "feature_demo", "cta"],
      "constraints": ["No invented SKUs", "No fake stats"]
    }
  },
  "target_duration_sec": 30,
  "aspect_ratio": "9:16",
  "voiceover": true
}
```

**交付给用户**：将返回的 **`spec_markdown`** 完整展示为 Markdown（包括 `# Final Video Spec`、编号的场景概览、**`## Voiceover Copy`** 等）。

**确认前**：加载 **creative-seedance2-prompt** 并丰富每个场景行的清晰 camera、motion 和 diegetic-audio 措辞（voiceover 开启时不写 BGM）。

**等待用户确认**（或在编辑后重新运行 `creative_generate_script`）后再提交最终渲染。

### 3. 估算积分 + 提交

1. `creative_estimate` with `workflow_type=script2film`
2. `creative_submit_script2film`：
```json
{
  "script": "<spec_markdown or user-confirmed script text>",
  "target_duration_sec": 30,
  "aspect_ratio": "9:16",
  "reference_image_urls": ["<user upload URL 1>", "<URL 2>"],
  "voiceover_enabled": true,
  "subtitles_enabled": true,
  "brief": { "product": "...", "audience": "..." },
  "delivery": { "mode": "both" },
  "client_request_id": "<uuid>"
}
```
3. 将返回的 `job_id` 交给 **creative-job-runner** —— **立即**发送 `tracking.user_message` 并**结束本轮**；**不要** sleep 或轮询 `creative_get_job`
4. 当用户在本线程询问进度时查询；完成后 **artifacts[0]** 是服务端 master（**已混入 BGM**）

### 4. 任务失败与重试（内容安全/版权）

Seedance 可能在 **per-shot video gen** 时拦截：

| 类型 | 典型错误 | 含义 |
|------|----------|------|
| **版权/IP** | output video may be related to copyright restrictions | 输出端审核——疑似受保护 IP、品牌外观等 |
| **隐私/真脸** | PrivacyInformation / InputImageSensitiveContentDetected | 输入参考包含可识别真人 |

**Agent 处理方式**：
1. 解释**该镜头**被拦截；其他镜头可能已成功（任务可能仍有部分 artifacts）
2. 请用户**修改该镜头脚本**（更抽象/电影化；避免 IP 名称/品牌/名人）和/或**更换参考图**
3. 用新 `client_request_id` 重新提交完整 script2film
4. 相同 prompt 有时**重试可通过**（随机性）；若反复失败，修改文案/图片

## 时长规划

| 参数 | 含义 |
|------|------|
| `target_duration_sec` | 目标总长度（**16–120s**；>15s 使用本 Skill） |
| `voiceover_enabled` | 静音 Seedance shots; 后续由客户端处理 VO |
| `subtitles_enabled` | 客户端 ffmpeg 烧录字幕 |

**30s 示例**：`target_duration_sec=30`，planner 分配接近 30s（如 4+6+5+7+4+4）——**非固定每镜头 5s**。
