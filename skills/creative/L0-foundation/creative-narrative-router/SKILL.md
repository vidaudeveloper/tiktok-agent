---
name: creative-narrative-router
description: 从用户意图路由叙事结构（product_ad / story_narrative / problem_solution 等）；在 `creative_generate_script` 前加载匹配的参考文件
metadata:
  layer: L0-foundation
  requires: []
  tags: [foundation, narrative, script, routing, storyboard]
---

# Creative Narrative Router

在 **`creative_generate_script`** 之前，选择一个 **`narrative_structure`**，**Read** 匹配的参考文件，然后将节奏点、约束和场景韵律注入 `brief.narrative`。

**不要**对所有视频使用通用的 HOOK→CLIMAX 模板。结构跟随用户意图；细节存在于参考文件中（按需加载）。

## 语言与区域

| 规则 | 说明 |
|------|------|
| **Skill 文档** | 英文（本文件 + references） |
| **用户对话** | 匹配用户的语言（EN、ES、JP、ZH 等） |
| **脚本输出** | `creative_generate_script` 根据 `brief.locale` 设置输出语言——不强制中文 |

## 运行时机

| 场景 | 是否需要 |
|------|---------|
| `creative-script2film` / `-keyframes` — 生成脚本 | **必须** |
| `product-url-to-video` — 提交前 | **必须** |
| `creative-direct` — 单个 clip ≤15s | 可选 |
| 用户已提供完整 Final Video Spec | 跳过 router → 直接提交 |

## 流程（3 步）

### Step 1 — 解析意图

从用户消息 + `brief` 中提取：

| 字段 | 含义 |
|------|------|
| `content_goal` | sell / brand / story / promo / mood / explain |
| `has_concrete_product` | 命名了 SKU、产品参考图或 `brief.product` |
| `platform` | TikTok、Reels、YouTube Shorts、… |
| `target_duration_sec` | 16–120，默认 30 |
| `voiceover` | 默认 `false` |
| `user_explicit_structure` | 用户明确指定类型（如 "品牌片"、"故事型"、"痛点广告"） |

### Step 2 — 选择 `narrative_structure`

**优先级**（高 → 低）：

1. 用户明确选择了某个结构
2. 关键词推断（见下表）——适用于多种语言；不确定时优先显式 `brief.narrative_structure`
3. 低置信度 → 展示 **2–3 个不同**结构的选项；等待用户选择
4. 仍不清楚 → 有产品/SKU 时选 `product_ad`，否则有痛点描述时选 `problem_solution`

#### 可用结构

| `narrative_structure` | 用户信号（英文示例） | 推断提示 | 参考 |
|----------------------|---------------------|---------|------|
| `product_ad` | "产品广告", "功能展示", "SKU", "UGC 卖货" | `brief.product`, 产品参考图 | `references/product-ad.md` |
| `story_narrative` | "故事", "迷你电影", "情感", "非硬卖" | 故事意图且非纯卖货 | `references/story-narrative.md` |
| `problem_solution` | "痛点", "前后对比", "省时间", "困扰" | brief 中有问题描述 | `references/problem-solution.md` |
| `brand_film` | "品牌片", "品牌故事", "价值观", "宣言" | 品牌调性，弱 CTA | `references/brand-film.md` |
| `event_promo` | "促销", "新品发布", "大促", "限量" | 促销/折扣/活动 | `references/event-promo.md` |
| `mood_film` | "氛围", "美学", "感官", "极简文案" | 感觉主导，弱脚本 | `references/mood-film.md` |
| `knowledge_explainer` | "讲解", "原理", "教程", "科普" | 教学/解释主题 | `references/knowledge-explainer.md` |
| `character_showcase` | "人物展示", "模特", "lookbook" | 人物主导，无 SKU 焦点 | `references/character-showcase.md` |

> **服务端**：`creative_generate_script` 读取 `brief.narrative_structure` 并注入节奏点；若省略，服务端以相同的关键词规则推断。响应包含 `narrative_structure` / `secondary_narrative_structure`。

#### 组合结构

电商常用 **`product_ad` + `problem_solution`**：痛点 hook → 产品揭晓。
设主结构 `product_ad`，`secondary_structure: problem_solution`，**两者参考文件都 Read**。

### Step 3 — 加载参考 → 调用 MCP

1. **Read** 所选结构的参考文件（路径相对于本 SKILL.md）
2. 总结 **beats**、**forbidden**、**scene_types_hint** 到 `brief.narrative`
3. 调用 **`creative_generate_script`**

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
      "constraints": ["No invented SKUs", "No fake stats"],
      "scene_types_hint": ["hero_product", "feature_closeup", "lifestyle", "cta_card"]
    }
  },
  "target_duration_sec": 30,
  "aspect_ratio": "9:16",
  "voiceover": false
}
```

## 多选项推荐（低置信度时）

每行必须使用**不同的** `narrative_structure`：

| 选项 | 结构 | 一句话卖点 | 为什么 |
|------|-------|-----------|--------|
| A | product_ad | 30s竖版，3个卖点快切 | 有产品参考图+清晰SKU |
| B | problem_solution | 通勤噪音→ANC耳机 | 用户强调了痛点 |
| C | story_narrative | 一天生活场景，耳机自然出现 | 用户想要软性植入 |

选择后 → Read 参考 → `creative_generate_script`。

## 不要编造产品

如果用户**没给**产品名称且**没给**产品参考图：
- **不要**选择 `product_ad` 并编造手表、手机等
- 优先 `story_narrative`、`problem_solution` 或 `character_showcase`（用占位符）
- 或询问："你有产品名称或参考图片吗？"

## 下游依赖

- **creative-script2film** / **creative-script2film-keyframes** — 脚本生成前需要本 Skill
- **product-url-to-video** — 通常 `product_ad`（+ 可选 `problem_solution`）
