# TikTok Ads + Creative Agent — 安装提示词（v2.4.0 一体化版）

将此提示词**完整复制粘贴**到 WorkBuddy / AI Agent / Cursor 等 AI Agent 对话中，即可自动完成 skills 安装：

```
请帮我安装 TikTok Ads + Creative Agent 一体化工具集（v2.4.0），包含以下步骤：

## Step 1: 安装主 Skill（ads-tiktok-mcp，仓库根目录）

从 GitHub 仓库读取主 skill 文件：
https://github.com/vidaudeveloper/tiktok-agent/blob/main/SKILL.md

这是一个基于 MCP (Model Context Protocol) 的 TikTok 广告管理 + 素材创作一体化 skill，支持：
- 广告创建（create_campaign / create_adgroup / create_ad）
- AI 广告巡检（花费、转化、CPA、ROAS、预算、素材疲劳六大异常检测）
- 日报/周报生成
- ★ 素材创作（图片/视频/BGM/脚本/批量变体/爆款短视频/商品链接转视频）

该 skill 已内置脱敏 API Key（末尾 15 位以 x 代替），不暴露真实 Key、不写日志。
调用逻辑统一在 references/mcp-client.md（TikTok Ads）和 skills/creative/references/creative-mcp-client.md（Creative）（JSON-RPC 2.0 容错、重试退避、并发限流、参数预校验）。

## Step 2: 安装知识库子 Skill（vidau-tiktok-agent-knowledge，skills/knowledge）

从 GitHub 仓库读取知识库子 skill：
https://github.com/vidaudeveloper/tiktok-agent/tree/main/skills/knowledge

这个 skill 负责：
- TikTok 广告平台的自然语言意图识别与业务路由
- 11 大业务模块覆盖（开户、授权、同步、报表、素材管理、素材创作★、搭建、预警等）
- AI Agent AI 助手面板的 JSON 响应格式（ui_actions / tool_requests / cards / charts / action_buttons）
- 缺失槽位检测与用户引导
- ✅ 已内置 MCP 执行层关联（related_skills: [ads-tiktok-mcp, vidau-tiktok-creative]），自动复用两步滤波法等策略
- ✅ v1.2.0 新增 tiktok_creative_generation 路由模块，支持创作→投放一体化

## Step 3: ★ 安装素材创作子 Skill（vidau-tiktok-creative，skills/creative）

从 GitHub 仓库读取素材创作子 skill：
https://github.com/vidaudeveloper/tiktok-agent/tree/main/skills/creative

这是 VidAU Creative Agent 的 TikTok Ads 适配版，包含三层架构 11 个子 Skill：
- L0 基础层：平台网关、任务追踪、叙事路由、Seedance 视频 Prompt、GPT-Image-2 图片 Prompt
- L1 生产层：一键生成（同步≤15s）、脚本转视频 reference 模式、首尾帧模式、批量编排
- L2 垂直场景：TikTok 爆款批量变体（trend-viral-short）、商品链接→视频（product-url-to-video）

核心价值：用户在同一 Agent 中「做个广告视频」→「用这个视频创建投放」，全链路一体。

## Step 4: 配置 MCP 连接（鉴权说明）

### 广告管理 MCP（tiktok.vidau.ai）
API Key 已内置预配置（脱敏，末尾 15 位以 x 代替），直接使用下方 SSE 端点配置即可，无需设置环境变量。

### ★ 素材创作 MCP（creative.vidau.ai）
> **重要**：Creative MCP **通常不需要独立的 API Key**——它走的是你 Agent 平台的 **VidAU 会话鉴权**（与 Ads MCP 同一套登录态）。
> 只需在 MCP 配置里把端点指向 `https://creative.vidau.ai/mcp` 并确认已连接即可。
>
> **仅当你的私有部署明确要求独立 Key 时**，才需要设置 `CREATIVE_MCP_API_KEY`（填入其 base64 串）；标准 VidAU SaaS 部署无需此步。

标准 MCP 端点配置参考（在你的 Agent MCP 设置中）：

{
  "mcpServers": {
    "tiktok-ads-agent": {
      "url": "https://tiktok.vidau.ai/api/mcp/sse?apiKey=tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
    },
    "vidau-creative": {
      "url": "https://creative.vidau.ai/mcp"
    }
  }
}

> 在你已有的 Agent 系统里（截图中的 Creative Designer / TikTok Ads Specialist），这两个 MCP 通常已经随平台登录态自动连接，无需额外配置 Key。

## Step 5: 验证安装

安装完成后，依次执行以下验证：
1. 输入 "TikTok 广告层级是什么？" → 应返回知识库模块的正确层级结构说明
2. 输入 "查看投放数据" → 应返回当前账户的巡检摘要
3. 输入 "帮我创建一条 TikTok 广告" → 应进入参数收集流程，提示缺失必填项
4. 输入 "生成 TikTok 日报" → 应生成完整的日报卡片
5. ★ 输入 "帮我的产品做一个 TikTok 广告视频" → 应进入素材创作流程（收集需求 → Prompt 工程 → Creative MCP 生成 → 返回视频 URL）
6. ★ 输入 "贴个商品链接帮我做广告视频" → 应自动爬取商品信息并生成视频
```

---

## 注意事项

- **密钥已脱敏**：API Key 末尾 15 位以 `x` 代替，真实 Key 不会出现在本文件；请勿外传到不可信环境。
- ⚠️ **密钥轮换提醒**：本仓库历史版本中曾以明文存放过 TikTok Ads Key。如你关心泄露风险，建议到 VidAU 后台轮换（revoke）该 Key，并重新生成混淆串替换。
- 直连 MCP 方式不需要登录 VidAU，但需要有效的 API Key（仅 Ads MCP 需要；Creative MCP 走平台会话鉴权）。
- **两个 MCP 端点**：Ads 用 `tiktok.vidau.ai`（需 Key），Creative 用 `creative.vidau.ai`（通常无需独立 Key，随平台登录态自动鉴权）。
- 巡检和日报默认不自动同步数据（sync_advertiser_data 较慢），仅当指定账户且数据陈旧时才谨慎调用；请在平台手动刷新作为兜底。
- 所有高风险操作（创建广告、修改预算等）均需你明确确认后才会执行。
- 素材创作需 VIP 账户和足够积分；非 VIP 会返回购买链接。
- 创作产出的图片 URL 可直接用作 `cover_url`；视频经平台后端 `upload-to-tiktok` 端点上传至 TikTok CDN 得到 `video_id`，后端已实现、待 MCP 工具注册后 Agent 自动衔接（注册前需用户经面板上传一次）。

---

## 快速验证命令

安装后，在对话中输入以下任一命令测试：

| 测试命令 | 预期响应 |
|----------|----------|
| `查看投放数据` | 列出已授权账户，过滤有 Campaign 的账户，展示巡检摘要 |
| `生成今日日报` | 生成包含指标、环比变化、问题/优秀广告、优化建议的日报 |
| `帮我创建 TikTok 广告` | 进入参数收集流程，提示缺失的必填字段 |
| `TikTok 广告系列和广告组有什么区别` | 返回三级广告结构的清晰解释 |
| ★ `帮我的无线耳机做一个竖版广告视频` | 进入素材创作流程 → 生成 9:16 视频 → 返回 download URL |
| ★ `把这个链接做成 TikTok 广告视频 https://...` | 自动爬取商品信息 → 生成产品广告视频 |

---

## 常见问题

### Q: 提示 "API Key 无效" 怎么办？
A: 先用下方 SSE 配置中的脱敏 Key 验证连通性；若提示无效，请到 VidAU 后台核对你的真实 Key 是否有效/未过期（真实 Key 由你自行安全保管）。**注意：仅 Ads MCP 使用 API Key；Creative MCP 走平台登录会话鉴权（`vidau_user_id`），无独立 Key。**

### Q: 创作时提示余额不足？
A: Creative MCP 生成类操作需要 VIP 和积分。非 VIP → 按提示链接开通/充值。

### Q: 生成的视频怎么用到广告里？
A: 图片 URL 可直接作为 `create_ad.cover_url`；视频经平台后端 `upload-to-tiktok` 端点（已实现的 REST 端点，对应 `material_upload_to_tiktok`）上传到 TikTok CDN 得到 video_id 后传入 `create_ad.video_id`。MCP 网关注册该工具后 Agent 全自动衔接；注册前需用户经面板上传一次视频。

### Q: 巡检显示"数据不足"？
A: 先在平台手动执行一次数据同步（点击"同步实体数据"），然后重新巡检。

### Q: 如何切换到浏览器模式？
A: 直连 MCP 是最快路径。如需文件上传/OAuth 等操作，回复"打开广告面板"即可使用浏览器。

---

*最后更新: 2026-07-11 · v2.4.0（执行层 ads-tiktok-mcp v2.4.0 + 知识层 v1.2.0 + 创作层 vidau-tiktok-creative v1.0.0）*
*一体化能力：广告管理 + AI 巡检 + 日报/周报 + 图片/视频/BGM 创作 + 爆款变体 + 商品链接转视频*
