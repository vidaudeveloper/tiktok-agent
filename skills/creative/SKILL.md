---
name: vidau-tiktok-creative
version: 1.0.0
category: advertising
description: >-
  VidAU Creative Agent 素材创作子 Skill —— 为 TikTok Ads Specialist 提供一体化素材生成能力。
  支持图片/视频/BGM 一键生成、脚本转视频、批量爆款变体、商品链接转视频。
  通过 Creative MCP（creative.vidau.ai）执行，与 TikTok Ads MCP（tiktok.vidau.ai）协同工作：
  创作产出（video_url / image_url / download URL）可直接传入 ads-tiktok-mcp 的 create_ad 实现创作→投放闭环。
related_skills: [ads-tiktok-mcp, vidau-tiktok-agent-knowledge]
---

# VidAU TikTok Creative — 素材创作子 Skill

**本 Skill 是 `ads-tiktok-mcp`（TikTok Ads Specialist）的素材创作扩展。**

> **核心价值**：用户在同一个 Agent 对话中，既能「帮我做个广告视频」又能「用这个视频创建广告投放」——**一个 Agent，全链路搞定**。

## 与 TikTok Ads 的协作关系

```
用户说"帮我做个产品广告视频"
        ↓
  ┌─ Creative Skill（本 Skill）────────────┐
  │ 1. 收集需求（产品/卖点/风格/时长/比例） │
  │ 2. Prompt 工程（Seedance / GPT-Image-2）│
  │ 3. 调 Creative MCP 生成                │
  │ 4. 返回 video_url / image_url          │
  └──────────────────┬─────────────────────┘
                     ↓ 产出物
  ┌─ Ads Skill（ads-tiktok-mcp）────────────┐
  │ 5. 用产出的 URL 创建 Ad                 │
  │    create_ad({ video_id, cover_url })   │
  │ 6. 投放 + 巡检 + 日报                   │
  └─────────────────────────────────────────┘
```

### 两个 MCP 端点

| 用途 | MCP 端点 | 环境变量 | Skill |
|------|----------|----------|-------|
| 广告管理 | `https://tiktok.vidau.ai/api/mcp/sse?apiKey=tk_1bfa…5xxx` | 内置脱敏 Key（无需 env） | ads-tiktok-mcp |
| 素材创作 | `https://creative.vidau.ai/mcp` | 平台会话鉴权（无独立 Key；私有部署可选 `CREATIVE_MCP_API_KEY`） | **本 Skill** |

> Ads 用内置脱敏 API Key（SSE 端点 `tk_1bfa…5xxx`）；Creative 走平台会话鉴权（`vidau_user_id`），标准部署无需独立 Key，两个端点不同。

## 前置条件

```bash
# Creative MCP 走平台会话鉴权（登录后自动注入 vidau_user_id），标准 VidAU SaaS 无需 Key
# 仅私有部署明确要求独立 Key 时，才设置：
# export CREATIVE_MCP_API_KEY="cr_xxx"
```

所有 Creative MCP 调用逻辑统一在本 Skill 的 `references/creative-mcp-client.md`（容错/重试/校验），禁止各子 skill 自行实现请求。

> 会话开始建议先调用 `creative_mcp_healthcheck()` 验证 key 与连通性。

## 技能文件结构

```
skills/creative/
├── SKILL.md                              # ★ 本文件 — 总入口、工具清单、模块总览、与 Ads 协作
├── references/
│   └── creative-mcp-client.md            # ★ Creative MCP 调用实现（与 ads mcp-client 对齐）
├── agents/
│   └── openai.yaml                      # Agent 接口配置
├── L0-foundation/                        # 基础层 — 所有上层依赖
│   ├── creative-platform/               # 平台网关（上传/计费/规则）
│   ├── creative-job-runner/             # 异步任务追踪
│   ├── creative-narrative-router/       # 叙事结构路由
│   ├── creative-seedance2-prompt/       # Seedance 2.0 视频 Prompt 工程
│   └── creative-gpt-image2-prompt/      # GPT Image 2 图片 Prompt 工程
├── L1-capability/                        # 生产能力层
│   ├── creative-direct/                 # 一键生成（同步，单 clip ≤15s）
│   ├── creative-script2film/            # 脚本转视频（参考图模式，多镜头 16-120s）
│   ├── creative-script2film-keyframes/  # 脚本转视频（首尾帧模式）
│   └── creative-batch-orchestrator/     # 批量编排（最多 10 项混排）
└── L2-vertical/                          # 垂直场景层
    ├── trend-viral-short/               # ★ TikTok 爆款短视频（批量图片变体）
    └── product-url-to-video/            # 商品链接→广告视频
```

## 触发关键词

| 模块 | 关键词 |
|------|--------|
| 素材创作（通用） | 做个视频、生成视频、创作素材、AI生图、做个海报、生成图片、帮我设计、素材创作、创意视频、广告素材 |
| 一键生成 | 一键生成、直接生成、快速出图、快速出视频、单张图片、单个视频 |
| 脚本转视频 | 脚本转视频、文案转视频、分镜视频、多镜头、长视频、30秒视频、60秒视频、品牌片 |
| 批量创作 | 批量生成、批量出图、A/B测试素材、多个版本、变体、爆款视频 |
| TikTok 爆款 | 爆款视频、TikTok爆款、带货短视频、hook视频、引流视频、趋势素材 |
| 商品链接转视频 | 这个链接做个视频、商品页转视频、产品URL转广告 |

> 说明：当用户同时表达「创作素材」+「用于广告投放」时，本 Skill 自动进入**创作→投放衔接流程**（见下方§衔接协议）。

## Creative MCP 工具清单

### 图片生成

| MCP 工具 | 用途 | 同步/异步 |
|----------|------|-----------|
| `creative_generate_image` | 单张图片生成 | 同步（~1-2min） |
| `creative_submit_batch_variants` | 批量图片变体（5 张起） | 异步（job 追踪） |

### 视频生成

| MCP 工具 | 用途 | 同步/异步 |
|----------|------|-----------|
| `creative_generate_video` | 文字→视频（无参考图） | 同步（~2-5min） |
| `creative_image_to_video` | 参考图→视频（reference 模式） | 同步（~2-5min） |
| `creative_first_frame_to_video` | 首/尾帧→视频 | 同步（~2-5min） |
| `creative_submit_script2film` | 完整脚本→视频（多镜头 reference 模式） | 异步（10-30min） |
| `creative_submit_script2film_keyframes` | 完整脚本→视频（首尾帧模式） | 异步（10-30min） |

### BGM / 音频

| MCP 工具 | 用途 |
|----------|------|
| `creative_generate_bgm` | 生成背景音乐 |
| `creative_mux_bgm_into_video` | 将 BGM 混入视频 |

### 脚本 / 其他

| MCP 工具 | 用途 |
|----------|------|
| `creative_generate_script` | 从 brief 生成 Final Video Spec 脚本（免费，不扣币） |
| `creative_estimate` | 估算所需积分和时间（免费） |

### 任务管理

| MCP 工具 | 用途 |
|----------|------|
| `creative_get_job` | 查询单个任务状态 |
| `creative_list_jobs` | 查询任务列表 |
| `creative_cancel_job` | 取消任务 |
| `creative_get_upload_instructions` | 获取 S3 预签名上传信息 |
| `creative_upload_reference` | 上传参考图（base64 兜底） |
| `creative_list_models` | 列出可用模型 |

### 批量编排

| MCP 工具 | 用途 |
|----------|------|
| `creative_submit_workflow` | 统一提交入口（direct_video / direct_image / batch_variants） |
| `creative_submit_batch_variants` | 批量图片变体提交 |

## 业务模块总览

### 模块 A：一键生成（L1 creative-direct）

**最适合 TikTok 广告的快速素材生产。**

- 单张图片 ≤15s 短视频 → 直接输出
- 用户给参考图 → 图生视频；没参考图 → 文生视频
- 可选加 BGM
- **产出物**：`artifacts[0].urls.download` → 直接传入 `create_ad.video_id` / `cover_url`

### 模块 B：脚本转视频（L1 script2film / keyframes）

**多镜头广告视频（16s–120s）。**

- 一句话/卖点列表 → 自动生成脚本 → 多镜头视频
- Reference 模式（产品必须像参考图）或 Keyframe 模式（平滑转场）
- 自动配 BGM
- **适合**：品牌片、功能展示、故事型广告

### 模块 C：批量创作（L1 batch-orchestrator + L2 vertical）

**A/B 测试 / 爆款矩阵。**

- `trend-viral-short`：TikTok 专属——批量生成带强 hook 的竖版图片/短视频变体，专为 TikTok/Reels 信息流广告设计
- `product-url-to-video`：粘贴商品链接 → 自动爬取信息 → 生成广告视频
- 最多 10 项混合编排（视频+图片混排）

### 模块 D：Prompt 工程层（L0 foundation）

**所有生成前的必经门槛。**

- `creative-seedance2-prompt`：视频 prompt → Seedance 2.0 生产级提示词工程
- `creative-gpt-image2-prompt`：图片 prompt → GPT Image 2 六块协议
- `creative-narrative-router`：叙事结构路由（product_ad / story_narrative / problem_solution …）
- **规则**：任何生成类 MCP 调用前，必须先加载对应 Prompt Skill，产出 paste-ready prompt 后才传入 MCP。**禁止将用户原始文本直接作为 prompt 传给 MCP。**

## ★ 创作→投放衔接协议（与 ads-tiktok-mcp 协同）

当用户意图同时包含「创作」+「投放」时：

### 衔接流程

```
Step 1: Creative Skill 生成素材
  → 输出: { video_url, image_url, download_url, artifacts }

Step 2: 用户确认使用该素材投放
  → Agent 将产出物映射为 ads-tiktok-mcp 参数:

  | Creative 产出 | ads create_ad 参数 | 当前状态 |
  |--------------|-------------------|----------|
  | video_url (download) | `video_id` | ✅ 后端已实现 upload-to-tiktok 端点；🔧 待 MCP 网关注册 `material_upload_to_tiktok` 工具 |
  | image_url (download) | `cover_url` | ✅ 封面图可直接用 URL |
  | artifacts[0].urls.download | `creative_id` | ✅ 后端已实现 save-from-url 端点；🔧 待 MCP 网关注册 `material_save_from_url` 工具 |

Step 3: 进入 ads-tiktok-mcp 广告创建流程
  → 复用已有的 Campaign → Adgroup → Ad 创建链路
  → 素材步骤自动填入 Step 2 的产出物
```

### 🔄 自动化路径（后端已实现，待 MCP 网关注册工具）

> **需求规格 & 实现状态**：详见 `docs/MCP-MATERIAL-LIBRARY-SPEC.md`
>
> 平台后端已实现以下能力的 REST 端点（`tiktok-ads-agent` 仓库）；当 MCP 网关将它们注册为 MCP 工具后，视频素材可实现**全自动衔接**：

| 新增工具（MCP） | 对应后端端点 | 用途 | 替代手动操作 |
|----------------|-------------|------|-------------|
| `material_save_from_url` | `POST /api/creatives/save-from-url` | Creative 下载 URL → 入库 VidAU 素材库（标记 vidau.ai 来源） | 手动上传素材 |
| `material_upload_to_tiktok` | `POST /api/creatives/[id]/upload-to-tiktok` | VidAU 库素材 → 上传 TikTok CDN 拿 `video_id` | 手动同步到 TikTok |
| `material_list` | `GET /api/creatives`（已有） | 查看素材库存、选择已有素材复用 | 手动浏览素材页 |
| `material_sync_from_tiktok` | `POST /api/creatives` 拉取（已有） | 从 TikTok 拉取已有素材入库 | 手动点「同步」按钮 |

**实现后的完整自动流程**：
```
creative_generate_video() → download URL
  → material_save_from_url(url, type=VIDEO)     // ★ 新
    → material_upload_to_tiktok(material_id)      // ★ 新
      → tiktok_video_id
        → create_ad({video_id})                   // ✅ 已有
```

### 🔧 当前临时方案（MCP 工具就绪前）

在上述 MCP 工具上线前，按以下步骤操作：

**图片素材（全自动）**：
1. Creative 生成图片 → 得到 image_url
2. 直接传入 `create_ad(cover_url=image_url)` ✅

**视频素材（半自动，需用户一次操作）**：
1. Creative 生成视频 → 得到 video download URL + 预览展示给用户
2. 用户确认使用 → **Agent 引导用户将视频上传至 TikTok 广告管理平台**：
   - 方式 A：打开 TikTok Ads Manager → 素材管理 → 「+ 上传素材」→ 粘贴 URL 或选择文件
   - 方式 B：在 VidAU 平台素材管理页点「+ 上传素材」，填入 Creative 产出的 download URL
3. 上传完成后用户将 `video_id`（或素材 ID）粘贴回对话
4. Agent 执行 `create_ad({video_id: <用户提供的ID>}, ...)`

### 衔接示例对话

> **用户**：「帮我的无线耳机做一个 TikTok 广告视频然后投放」
>
> **Agent**：
> 1. （Creative Skill）收集需求 → 生成 9:16 竖版 15s 产品视频 → 返回 download URL
> 2. 「视频已生成！是否用这个视频创建 TikTok 广告投放？」
> 3. 用户确认 → （Ads Skill）进入参数收集（账户/目标/定向/预算...）
> 4. `create_ad({ video_id: <上传后ID>, cover_url: <封面URL>, ... })`
> 5. 返回广告 ID + 触发巡检

### ⚠️ 当前限制

| 限制 | 原因 | 解决方案 |
|------|------|----------|
| Creative 生成的视频不能直接经 MCP 自动用作 `video_id`（需 MCP 工具注册） | TikTok Ads 要求视频先上传到 TikTok CDN | 后端已实现 `upload-to-tiktok` 端点；MCP 工具注册后 Agent 自动上传，注册前引导用户经面板/浏览器上传一次 |
| 图片 URL 可直接用作 `cover_url` | 封面接受 HTTPS URL | ✅ 无缝衔接 |

## 计费说明

| 操作 | 是否扣币 | 备注 |
|------|----------|------|
| `creative_estimate` | ❌ 免费 | 建议每次生成前调用 |
| `creative_generate_script` | ❌ 免费 | 脚本生成不扣币 |
| `creative_get_upload_instructions` | ❌ 免费 | 获取上传预签名 |
| `creative_upload_reference` | ❌ 免费 | 参考图上传 |
| `creative_list_models` | ❌ 免费 | 列模型 |
| `creative_mux_bgm_into_video` | ❌ 免费 | BGM 混合 |
| 图片/视频生成 | ✅ 扣 VIP 积分 | 需要 VIP 账户 |
| `creative_generate_bgm` | ✅ 扣 VIP 积分 | BGM 生成 |

非 VIP 或余额不足 → 分享购买链接，**不要编造结果**。

## 定时任务

- 无独立定时任务（创作由用户按需触发）。
- 创作完成后可衔接 Ads Skill 的定时巡检/日报。

## 版本基线

- **v1.0.0**（初始）：从 vidau-creative-agent 适配为 TikTok Ads Specialist 子 Skill；去除 hermes 字眼；新增与 ads-tiktok-mcp 的创作→投放衔接协议；保留完整的三层架构（L0/L1/L2）。
