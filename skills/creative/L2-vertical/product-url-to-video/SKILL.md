---
name: product-url-to-video
description: 商品链接→爬取信息→广告视频/图片（支持 DTC 和主流电商平台）。★ 电商广告主最高效的创作入口
metadata:
  layer: L2-vertical
  requires: [creative-job-runner, creative-platform, creative-seedance2-prompt, creative-gpt-image2-prompt, creative-script2film, creative-script2film-keyframes]
  tags: [ecommerce, product, url, scrape, script2film, bgm, one-click]
---

# Product URL → Video

用户粘贴 **商品页面 URL** 时启用。用 Agent web 工具爬取商品信息，然后调用 VidAU Creative MCP 生成广告素材。

> **Prompt Gate**：任何图片 MCP 前加载 **creative-gpt-image2-prompt**。任何视频 MCP 或脚本视觉丰富化前加载 **creative-seedance2-prompt**。
>
> **适用**：Shopify DTC、Amazon、TikTok Shop、Temu 等任何可访问的商品页。
> **不适用**：社交主页、云盘、YouTube/Bilibili 视频链接——作为普通对话处理。

## 视频 Skill 选择（L2 必读）

提交最终渲染前，从用户意图选择 **L1 video skill**：

| 用户意图/场景 | 加载 Skill | MCP 入口 |
|--------------|-----------|----------|
| 产品短视频；主角必须匹配主图（**默认**） | **creative-script2film** | `creative_submit_script2film` |
| 强调镜头转场、运镜电影感 | **creative-script2film-keyframes** | `creative_submit_script2film_keyframes` |
| 仅需 5–15s demo clip，无需多镜头 | **creative-direct** | `creative_image_to_video` 或 `creative_first_frame_to_video` |
| A/B 测试多个 hook **图片** | **trend-viral-short** | `creative_submit_batch_variants` |

**决策简写**：
- 有产品 hero、必须"看起来像这个 SKU" → **reference**（creative-script2film）
- 想要"平滑转场/故事运镜" → **keyframes**（creative-script2film-keyframes）
- 未指定时，电商默认 → **creative-script2film**

## 触发时机

消息包含 `https://` 且看起来像商品页（含 `product`、`/p/`、`/dp/`、`shop`、`store` 等）或用户说"这个链接的产品"。

## 流程总览

```
1. 爬取商品信息（Agent 本地工具）
2. 展示摘要并请用户确认
3. 估算积分 + 提交生成（MCP）
4. creative-job-runner — 立即发送 tracking.user_message；不 sleep/poll
```

## 1. 爬取商品信息

按顺序尝试；**首次成功即停止**：

### A. `web_extract`（推荐）

```
web_extract(urls=["<product URL>"], format="markdown")
```

提取：`product_name`、`brand`、`price`、`description`、hero image URLs（`og:image` / 最大商品图）。

### B. `execute_code`（结构化解析）

当 `web_extract` 返回不足或失败时。优先 **JSON-LD**（`@type: Product`）、**Open Graph**、`application/ld+json`。

**必填字段**：

| 字段 | 含义 |
|------|------|
| `product_name` | 产品标题 |
| `product_description` | 卖点/描述（截断 ~500 字符 OK） |
| `product_images` | Hero 图 URLs（最多 8 张，优选高清） |
| `price` | 可选，用于展示 |
| `brand` | 可选 |

### 爬取失败

明确告知用户："无法从该 URL 解析商品信息" —— 请用户手动提供产品图片 + 名称/卖点。不要强制 MCP 提交。

## 2. 用户确认

爬取成功后，**生成前展示摘要**：
- 产品名称、品牌、价格（如有）
- Hero 图预览（Markdown 图片或 URL）
- 卖点摘要（2–3 句话）

询问：
1. **输出类型**：短视频交付（默认）/ batch hook 变体 / 单张广告图
2. **比例**：默认 `9:16`（TikTok/Reels）
3. **时长**：默认 30s（script2film）
4. **参考图**：默认爬取的 hero（最多 3 张）

若用户只贴了 URL，默认 → **script2film 30s 竖版产品短片**。

## 3. MCP 提交

### 默认：script2film 交付（reference）

```
creative_submit_script2film:
  script: "<step 3 的 script>"
  reference_image_urls: ["<hero URL>", ...]
  brief: { product: "<product_name>", audience: "<inferred>" }
  aspect_ratio: "9:16"
  target_duration_sec: 30
```

### 备选：batch hook 变体

用户想要 A/B hook 测试 → **trend-viral-short**：

```
creative_submit_batch_variants:
  prompt: "<product name + selling points + trend hook>"
  count: 5
  aspect_ratio: "9:16"
```

### 备选：单张图/单个视频

使用 **creative-direct**：

- 图：`creative_generate_image` + `reference_urls: [<hero>]`
- 参考图视频：`creative_image_to_video` + `reference_image_urls: [<hero>]`

## 4. 任务追踪

提交后立即加载 **creative-job-runner**：
- 发送 `tracking.user_message`；用户可在本线程随时询问进度
- **不要** sleep / `creative_get_job` 循环
- 用户跟进时，一次 `creative_get_job` → 交付 URL + 本地提示（**script2film 默认包含 BGM**）
