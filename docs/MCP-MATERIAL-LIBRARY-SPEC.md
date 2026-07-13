# TikTok Ads Agent — 素材库 MCP 能力需求规格书

> 目标：补齐「创作→素材库→投放」全链路的最后两环。
> 受众：VidAU 后端工程 / MCP 接口负责人。

> **📌 实现状态（2026-07-13 更新）**：本规格书中的 `material_save_from_url` / `material_upload_to_tiktok` **后端 REST 端点已在 `tiktok-ads-agent` 平台源码中实现**（`POST /api/creatives/save-from-url` 与 `POST /api/creatives/[id]/upload-to-tiktok`），并已抽取共享 service `src/lib/creative-upload.ts` 供 `ads/route.ts` 复用。`material_list` / `material_sync_from_tiktok` 此前已存在。剩余工作：MCP 网关（`tiktok.vidau.ai/api/mcp`）将这 4 个端点**注册为 MCP 工具**——注册后 Agent 即可在对话内全自动完成 创作→入库→上传 TikTok→投放。

---

## 背景

### 当前状态

**tiktok.vidau.ai MCP**（10 个工具）：仅覆盖广告管理（Campaign / AdGroup / Ad / Metrics / Alerts），无任何素材相关工具。

**creative.vidau.ai MCP**（18 个工具）：覆盖素材生成（图片 / 视频 / 脚本 / 批量变体），但产出物仅为 download URL，无法回写至 VidAU 素材库或 TikTok CDN。

**VidAU 平台 UI**（已验证存在）：
- 「素材管理」页面，支持三来源：本地上传 / TikTok 同步 / vidau.ai
- 手动「+ 上传素材」「同步」按钮可用
- 素材带来源标签（TikTok / vidau.ai）、ID、分辨率、广告账户绑定

### 断裂链路

```
Creative MCP 生成 → download_url（当前终点）
                    ↓ ❌ 缺少：自动入库到 VidAU 素材库
               VidAU 素材库（vidau.ai 标签）
                    ↓ ❌ 缺少：自动上传到 TikTok CDN
               TikTok CDN → video_id
                    ↓ ✅ 已有：create_ad({video_id})
```

用户痛点：Agent 生成视频后，必须手动切到浏览器 → 上传素材 → 同步 → 复制 ID → 回到对话粘贴。**破坏了「一个 Agent 全搞定」的一体化体验。**

---

## 需求：新增 4 个 MCP 工具

> 建议添加到 **tiktok.vidau.ai** MCP（广告端点），因为素材库和广告账户强关联。
> 或新增独立端点 `material.vidau.ai/mcp`（若架构上更清晰）。

### Tool 1：`material_list`

列出当前账户的素材库内容。

```json
{
  "name": "material_list",
  "description": "List materials in the advertiser's material library, with optional filters by type, source, and pagination.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "advertiser_id": { "type": "string", "description": "Advertiser ID (required)" },
      "type": { "type": "string", "enum": ["VIDEO", "IMAGE", "ALL"], "description": "Filter by type. Default: ALL" },
      "source": { "type": "string", "enum": ["LOCAL", "TIKTOK", "VIDAU_AI", "ALL"], "description": "Filter by source. Default: ALL" },
      "page": { "type": "integer", "default": 1 },
      "page_size": { "type": "integer", "default": 20, "maximum": 50 }
    },
    "required": ["advertiser_id"]
  }
}
```

**返回示例**：
```json
{
  "total": 212,
  "page": 1,
  "items": [
    {
      "id": "cover-v10033g50000d8n...",
      "type": "VIDEO",
      "source": "TIKTOK",
      "resolution": "1080x1920",
      "duration_sec": 15,
      "download_url": "https://cdn.vidau.ai/materials/...",
      "tiktok_video_id": "v10033g50000d8n...",   // 仅 source=TIKTOK 时有值
      "created_at": "2026-07-10T08:30:00Z",
      "advertiser_id": "75469276767852552288"
    }
  ]
}
```

---

### Tool 2：`material_save_from_url`

将外部 URL（如 Creative 产出的 download URL）保存到 VidAU 素材库。

```json
{
  "name": "material_save_from_url",
  "description": "Save a material from an external URL into the VidAU material library. Supports both VIDEO and IMAGE. The material will be tagged with source=VIDAU_AI.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "advertiser_id": { "type": "string", "description": "Advertiser ID (required)" },
      "url": { "type": "string", "format": "uri", "description": "Download URL of the generated material (from Creative MCP output)" },
      "type": { "type": "string", "enum": ["VIDEO", "IMAGE"], "description": "Material type (required)" },
      "name": { "type": "string", "description": "Optional display name for the material" },
      "metadata": {
        "type": "object",
        "properties": {
          "job_id": { "type": "string" },       // Creative job ID for traceability
          "prompt": { "type": "string" }         // Original generation prompt
        }
      }
    },
    "required": ["advertiser_id", "url", "type"]
  }
}
```

**返回示例**：
```json
{
  "material_id": "mat_abc123def456",
  "type": "VIDEO",
  "source": "VIDAU_AI",
  "download_url": "https://cdn.vidau.ai/materials/mat_abc123...",
  "resolution": "1080x1920",
  "duration_sec": 15,
  "status": "READY",
  "created_at": "2026-07-13T16:20:00Z"
}
```

**使用场景（关键衔接点）**：
```
Step 1: creative_generate_video(...) → artifacts[0].urls.download = "https://..."
Step 2: material_save_from_url({
           url: "<download URL>",
           type: "VIDEO",
           advertiser_id: "xxx"
         })
       → 返回 material_id + 内部 cdn URL
Step 3: material_upload_to_tiktok({ material_id }) → 拿到 tiktok_video_id
Step 4: create_ad({ video_id: <tiktok_video_id> }) ✅
```

---

### Tool 3：`material_upload_to_tikTok`

将 VidAU 素材库中的素材上传到 TikTok CDN，获得可用于 `create_ad` 的 `video_id`。

```json
{
  "name": "material_upload_to_tiktok",
  "description": "Upload a material from VidAU library to TikTok CDN. Returns the TikTok video_id or image_id needed for create_ad. Only materials not yet uploaded (source != TIKOK) can be uploaded.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "advertiser_id": { "type": "string", "description": "Advertiser ID (required)" },
      "material_id": { "type": "string", "description": "Material ID from material_list or material_save_from_url (required)" }
    },
    "required": ["advertiser_id", "material_id"]
  }
}
```

**返回示例**：
```json
{
  "material_id": "mat_abc123def456",
  "tiktok_video_id": "v10033g50000d8n...",     // 可直接传入 create_ad.video_id
  "tiktok_image_id": null,                       // 如果是图片则有值
  "upload_status": "COMPLETED",
  "uploaded_at": "2026-07-13T16:22:00Z"
}
```

**或失败时**：
```json
{
  "error": "UPLOAD_FAILED",
  "reason": "Video duration exceeds TikTok limit (240s > 60s max)",
  "suggestion": "Trim video to 5-60 seconds before uploading"
}
```

---

### Tool 4：`material_sync_from_tiktok`（可选，已有类似能力）

从 TikTok 广告账户拉取已在 TikTok 侧的素材，同步到 VidAU 库。这对应 UI 上的「同步」按钮功能。

```json
{
  "name": "material_sync_from_tiktok",
  "description": "Sync existing materials from TikTok ad account into VidAU material library. Equivalent to clicking 'Sync' on the material management page.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "advertiser_id": { "type": "string", "description": "Advertiser ID (required)" },
      "sync_type": { "type": "string", "enum": ["INCREMENTAL", "FULL"], "default": "INCREMENTAL" }
    },
    "required": ["advertiser_id"]
  }
}
```

---

## 完整一体化流程图（实现后）

```
用户：「帮我的无线耳机做一个 TikTok 广告视频然后投放」

① [Creative MCP] creative_direct Skill 收集需求
② [Creative MCP] creative_generate_video(prompt={产品卖点, 9:16, 15s})
   → 返回 artifacts[0].urls.download = "https://cdn.creative.vidau.ai/output/xxx.mp4"

③ ★ [TikTok Ads MCP] material_save_from_url(
       url="<download>",
       type="VIDEO",
       advertiser_id="xxx"
   )
   → 入库 VidAU 素材库，source=VIDAU_AI，返回 material_id

④ ★ [TikTok Ads MCP] material_upload_to_tiktok(
       advertiser_id="xxx",
       material_id="mat_xxx"
   )
   → 上传到 TikTok CDN，返回 tiktok_video_id = "v10033g..."

⑤ [TikTok Ads MCP] create_campaign + create_adgroup + create_ad({
       video_id: "v10033g...",    ← 来自 Step ④
       cover_url: "...",           ← 来自图片生成（直接用 URL）
       ...
   })

⑥ [TikTok Ads MCP] 启动巡检 + 日报

→ 用户零手动操作，全链路自动化 ✅
```

---

## 优先级建议

| 优先级 | 工具 | 理由 |
|--------|------|------|
| **P0** | `material_save_from_url` | 创作→入库的核心桥接，没有它后续都走不通 |
| **P0** | `material_upload_to_tiktok` | 拿 video_id 的唯一途径，投放必需 |
| **P1** | `material_list` | 查看库存、选择已有素材复用 |
| **P2** | `material_sync_from_tiktok` | 锦上添花，UI 已可手动 |

---

## 技术备注

1. **端点位置**：建议挂在 `tiktok.vidau.ai/api/mcp/tools` 下（与现有 10 个广告工具同域同鉴权），无需新部署服务。
2. **鉴权**：沿用现有 `apiKey` 查询参数机制（`?apiKey=tk_xxx`），无需改动。
3. **异步考虑**：`material_upload_to_tiktok` 涉及 TikTok CDN 上传，可能需要 10-60 秒。建议内部异步处理 + 轮询模式，MCP 层暴露为同步调用（内部等待完成）或返回 job_id 让客户端轮询。
4. **幂等性**：同一 URL 重复 save 应去重；同一 material 重复 upload 应返回已有的 tiktok_video_id 而非报错。
5. **错误码**：明确区分「素材格式不支持」「TikTok 审核拒绝」「余额不足」「网络超时」等，方便 skill 层做用户友好的引导。

---

*Spec version: 1.0 · Author: VidAU TikTok Ads Agent Team · Date: 2026-07-13*
