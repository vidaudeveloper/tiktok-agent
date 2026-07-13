# TikTok Ads + Creative Agent — 一体化智能工具集

<p align="center">
  <strong>TikTok 广告 + 素材创作 一体化 MCP 工具集</strong><br>
  直连 MCP · 密钥环境变量化 · 稳健容错 · 执行层 + 知识层 + 创作层 三 Skill
</p>

---

## 版本基线

| 分层 | Skill | 当前版本 | 职责 |
|------|-------|----------|------|
| 执行层 | 根目录 `SKILL.md` + `references/` | **v2.4.0** | 调 VidAU TikTok MCP：广告创建 / AI 巡检 / 日报周报 / **素材创作路由** |
| 知识层 | `skills/knowledge/` | **v1.2.0** | 意图路由 / 广告知识问答 / 响应格式化 / **创作意图识别** |
| **创作层** | **`skills/creative/`** | **v1.0.0** | **图片/视频/BGM AI 生成、脚本转视频、批量爆款变体、商品链接→视频** |

> 三层为**一体化配套 Skill**：知识层负责「理解用户 + 路由 + 渲染」，执行层负责「调 TikTok Ads MCP」，创作层负责「调 Creative MCP 产素材」。三层协同实现**一个 Agent 对话内完成 创作→投放→巡检→报表 全链路**。

---

## 功能模块

### 执行层（根 Skill）— 四合一

- **广告创建**：账户检查 → Campaign → Adgroup → Ad，全程 MCP 直连
- **AI 巡检**：花费 / 转化 / CPA / ROAS / 预算耗尽 / 素材疲劳 六类异常自动检测
- **日报 / 周报**：一键生成，支持自定义日期与账户过滤
- **★ 素材创作（路由层）**：识别创作意图→分发到创作层 Skill→衔接广告创建流程

### 知识层子 Skill（`skills/knowledge/`）

- 自然语言意图识别与业务路由（11 大模块）
- TikTok 广告平台知识问答
- 平台面板 JSON 响应格式化
- 内置「用户口语 → VidAU 枚举」对照表
- ★ v1.2.0 新增 `tiktok_creative_generation` 路由模块 + Creative MCP 工具映射表

### ★ 创作层子 Skill（`skills/creative/`）—— 从 creative-agent 适配

#### L0 基础层（所有上层依赖）

| Skill | 用途 |
|-------|------|
| creative-platform | 平台网关（上传/计费/规则） |
| creative-job-runner | 异步任务追踪（禁止 sleep 轮询） |
| creative-narrative-router | 叙事结构路由（product_ad / story_narrative … 8 种） |
| creative-seedance2-prompt | Seedance 2.0 视频 Prompt 工程（SCELA 结构） |
| creative-gpt-image2-prompt | GPT Image 2 图片 Prompt 工程（六块协议） |

#### L1 生产层

| Skill | MCP 工具 | 典型场景 |
|-------|----------|----------|
| creative-direct | `creative_generate_image/video` | 一键生成，单 clip ≤15s |
| creative-script2film | `creative_submit_script2film` | 多镜头 16–120s 品牌/功能片 |
| creative-script2film-keyframes | `creative_submit_script2film_keyframes` | 首/尾帧平滑转场模式 |
| creative-batch-orchestrator | 混合批量提交 | 最多 10 项混排（视频+图片） |

#### L2 垂直场景层（★ TikTok Ads 核心价值）

| Skill | 用途 |
|-------|------|
| **trend-viral-short** | **TikTok 爆款短视频 —— 批量 hook 图片变体，专为 TikTok/Reels 信息流广告设计，A/B 测试矩阵** |
| **product-url-to-video** | **粘贴商品链接 → 自动爬取 → 生成广告视频（Shopify/Amazon/TikTok Shop/Temu）** |

---

## 快速开始

### 方式一：一键安装（推荐）

复制 [INSTALL_PROMPT.md](INSTALL_PROMPT.md) 中的安装提示词，粘贴到 WorkBuddy / AI Agent / Cursor 对话中。API Key 已内置脱敏（末尾 15 位以 x 代替），按提示配置 SSE 端点即可。

### 方式二：手动安装

1. 克隆本仓库
2. 将根目录 `SKILL.md` + `references/` 导入为执行层 Skill
3. 将 `skills/knowledge/` 导入为知识层子 Skill
4. 将 **`skills/creative/`** 导入为**创作层子 Skill**
5. 密钥已内置脱敏：**Ads MCP** 写死 `tk_1bfa…5xxx`（SSE 端点）；**Creative** 走平台会话鉴权。无需配置任何环境变量。
   - 如需更换 Ads Key：编辑根 `SKILL.md` 的 `API_KEY` 变量与 `INSTALL_PROMPT.md` 的 SSE 端点 `apiKey` 参数。
   - `CREATIVE_MCP_API_KEY`（素材创作）**仅私有部署需独立 Key**；标准 VidAU SaaS 走平台会话鉴权，无需此变量。

---

## 一体化工作流示例

```
用户：「帮我的无线耳机做一个竖版 TikTok 广告视频然后投放」

Agent：
  ① （创作层 L1 creative-direct）
     收集需求 → Seedance Prompt 工程 → Creative MCP 生成 9:16 视频
     ↓ 返回 download URL
  ② 「视频已生成！是否用这个视频创建 TikTok 广告投放？」
     用户确认
  ③ （执行层模块一）
     create_campaign(WEB_CONVERSIONS)
       → create_adgroup(targeting, budget, CPC)
         → create_ad(video_id=<上传后ID>, cover_url=<URL>)
  ④ 返回广告 ID + 触发 AI 巡检
  ⑤ 后续：定时日报、异常预警、素材疲劳检测……
```

**一个对话，全链路完成。**

## 两个 MCP 端点

| 用途 | 端点 | 环境变量 |
|------|------|----------|
| 广告管理 | `https://tiktok.vidau.ai/api/mcp/sse?apiKey=tk_1bfa…5xxx` | 内置脱敏 Key（无需 env） |
| 素材创作 | `https://creative.vidau.ai/mcp` | 平台会话鉴权（无需独立 Key；私有部署可选 `CREATIVE_MCP_API_KEY`） |

## 密钥配置（重要）

TikTok Ads 的 API Key 已**内置预配置（脱敏，末尾 15 位以 x 代替）**，无需自行设置环境变量；执行层代码直接使用该脱敏 Key。如需更换为你自己的 Key，编辑根 `SKILL.md` 的 `API_KEY` 变量与 `INSTALL_PROMPT.md` 的 SSE 端点即可。

```bash
# Creative Key（仅私有部署需独立 Key；标准 VidAU SaaS 走平台会话鉴权，无需此步）
# 如需，请将你的 key 做 base64 编码后填入：
# echo "<YOUR_CREATIVE_KEY_BASE64>" | base64 -d
# export CREATIVE_MCP_API_KEY="<解码结果>"
```

---

## 目录结构

```
tiktok-agent/
├── SKILL.md                              # 执行层主 Skill v2.4.0（四合一）
├── README.md                             # 本文件
├── INSTALL_PROMPT.md                     # 一键安装提示词（含脱敏密钥）
├── references/
│   ├── mcp-client.md                     # ★ TikTok Ads MCP 调用实现
│   ├── enums-reference.md                # ★ VidAU 枚举对照 + Python 常量
│   ├── ad-creation-flow.md               # 广告创建详细流程
│   ├── ad-inspection-rules.md            # AI 巡检规则详解
│   └── report-format.md                  # 日报/周报格式规范
├── assets/
│   └── icon.svg                          # 工具集图标
└── skills/
    ├── knowledge/                        # 知识层 v1.2.0
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/source_and_validation.md
    └── creative/                         # ★ 创作层 v1.0.0（NEW）
        ├── SKILL.md                      # 创作总入口 + 与 Ads 衔接协议
        ├── references/creative-mcp-client.md  # Creative MCP 调用实现
        ├── agents/openai.yaml
        ├── L0-foundation/                # 5 个基础 Skill
        │   ├── creative-platform/
        │   ├── creative-job-runner/
        │   ├── creative-narrative-router/
        │   ├── creative-seedance2-prompt/
        │   └── creative-gpt-image2-prompt/
        ├── L1-capability/                # 4 个生产 Skill
        │   ├── creative-direct/
        │   ├── creative-script2film/
        │   ├── creative-script2film-keyframes/
        │   └── creative-batch-orchestrator/
        └── L2-vertical/                  # 2 个垂直场景 Skill
            ├── trend-viral-short/        # ★ TikTok 爆款
            └── product-url-to-video/      # ★ 商品链接→视频
```

---

## 使用示例

| 你说 | 动作 |
|------|------|
| 「帮我创建一个 TikTok 广告」 | 进入参数收集流程，逐步创建 |
| 「看看今天的投放数据」 | 输出巡检摘要，标注异常指标 |
| 「生成本周周报」 | 输出完整的周报卡片 |
| 「TikTok 广告层级是什么？」 | 知识层返回三级结构解释 |
| ★ 「帮我的产品做一个广告视频」 | 创作层生成视频 → 询问是否用于投放 |
| ★ 「贴这个商品链接做个 TikTok 广告」 | 自动爬取 → 生成视频 → 可衔接创建 Ad |
| 「帮我做 5 张不同 hook 的爆款图」 | trend-viral-short 批量生成 A/B 变体 |
| 「打开广告搭建页面」 | 在本机浏览器打开平台广告搭建页 |

---

## 注意事项

1. 两个 MCP 端点需要各自有效的 API Key。
2. `sync_advertiser_data` 交互场景默认不调用；**仅在指定具体账户且数据陈旧时才谨慎调用**。
3. 非 VidAU 开通的 TikTok 账户仅支持数据查看和报表。
4. 所有高风险操作（创建广告、修改预算等）均需用户明确确认。
5. 枚举取值以 **VidAU 简化值**为准；`validate_args` 同时放行两类值，不会因枚举报错。
6. 素材创作需 VIP 和积分；Prompt Gate 强制所有生成前经过 Prompt 工程 Layer。
7. 创作产出图片可直接做 `cover_url`；视频经平台后端 `upload-to-tiktok` 端点上传后用作 `video_id`（后端已实现，待 MCP 网关注册工具）。

---

## 版本历史

- **v2.4.0** (2026-07-11)：**★ 融合 VidAU Creative Agent** —— 新增创作层（`skills/creative/`，11 个子 Skill，L0/L1/L2 三层架构）；新增模块四「素材创作」；创作→投放衔接协议；TikTok 爆款变体 + 商品链接转视频双 L2 场景。
- **执行层 v2.3.0** (2026-07-10)：按真实 VidAU MCP 返回体校准枚举回 VidAU 简化值；保留 API Key 环境变量化、稳健容错等修复。
- **知识层 v1.1.0** (2026-07-10)：去除平台专有字眼，与执行层 v2.3.0 对齐。
- **知识层 v1.2.0** (2026-07-11)：新增 `tiktok_creative_generation` 路由 + Creative MCP 工具映射表。
- **v2.0.0** (2026-07-08)：MCP 架构升级，三合一模块，直连 HTTP。
- **v1.0.0** (2026-07-03)：AI Agent 初始版本。

---

MIT License
