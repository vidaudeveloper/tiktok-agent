---
name: creative-direct
description: 一键图片/视频生成（同步，单 clip ≤15s；音频默认开启）。**最适合 TikTok 广告快速素材生产。**
metadata:
  layer: L1-capability
  requires: [creative-platform, creative-job-runner, creative-seedance2-prompt, creative-gpt-image2-prompt]
  tags: [image, video, sync, one-click]
---

# Creative Direct — 同步一键生成

用于单张广告图或 **单个 clip ≤15s** 的产品短视频 —— 无需分镜脚本。

> **Prompt Gate（必须）**：在任何下方 MCP 调用前，加载 **creative-gpt-image2-prompt**（图片）或 **creative-seedance2-prompt**（视频），产出 paste-ready prompt，然后将其作为 MCP `prompt` 传入。**绝不要**用用户原始文本。

> **时长路由**：用户想要 **>15s** / 30s / 60s / 多镜头 / 分镜 → 使用 **creative-script2film**（以 `creative_generate_script` 开始）；**不要**强行把长内容塞进本 Skill。

> **任务追踪**：任何生成调用前先加载 **creative-job-runner**；即使同步任务也提供实时状态。

## 视频 Skill 选择

| 需求 | Skill | MCP |
|------|-------|-----|
| 参考图、产品一致性要求高 | **creative-script2film** | `creative_submit_script2film` |
| 首/尾帧转场、可控运镜 | **creative-script2film-keyframes** | `creative_submit_script2film_keyframes` |
| 单个短 clip | **本 Skill** | `creative_generate_video` / `creative_image_to_video` / `creative_first_frame_to_video` |

## 图片生成

1. 告知用户："正在生成图片，约 1–2 分钟…"
2. **加载 creative-gpt-image2-prompt** — 从用户意图 + 参考图产出生产级 `prompt`
3. **当用户有本地/附加参考图时**（`@image` 等）：
   - **优先** `creative_get_upload_instructions` → 本地 curl/终端 PUT 到 S3 → 使用 `upload.file_url`
   - 兜底（无本地终端）：`creative_upload_reference`（`content_base64`）
4. `creative_generate_image`：
   - `prompt`：**creative-gpt-image2-prompt 的输出**（不是用户原始文本）
   - `aspect_ratio`：`9:16` | `1:1` | `16:9`
   - `reference_urls`：可选 — 来自上传步骤的 `file_url`（或已有 HTTPS URL）
5. 读取 `tracking.user_message`；返回 `artifacts[0].urls.download` + 本地保存提示

## 视频生成

1. 告知用户："正在生成视频，约 2–5 分钟…"
2. **加载 creative-seedance2-prompt** — 产出生产级 `prompt`（含 reference roles、camera、audio 规则）
3. 有用户参考图时 → **`creative_image_to_video`**（Seedance **reference-to-video** 模式，`reference_image` role —— 不是 first/last frame）：
   - `prompt`：**creative-seedance2-prompt 的输出**
   - `reference_image_urls`：产品/人才/场景/风格等（最多 9 张）
   - 或单个 `reference_image_url`
4. 无参考图时 → `creative_generate_video`（text-to-video），prompt 用步骤 2 的产出
5. 交付 artifacts + `tracking.user_message`

## 可选：BGM

对于带背景音乐的单个短 clip：

1. `creative_generate_bgm`（可传 `script` / `brief` / `bgm_hint` 自动生成 prompt）
2. `creative_mux_bgm_into_video` — 混合 `video_url` + `bgm_url`

> script2film 一键交付物**自动混入 BGM**（在 workflow 中）；direct 视频需要上述两步。

## 默认值

- 竖版短视频：`aspect_ratio=9:16`，`duration_sec=5`
- **音频开启**：`generate_audio=true`（默认）；in-shot SFX 来自 Seedance
