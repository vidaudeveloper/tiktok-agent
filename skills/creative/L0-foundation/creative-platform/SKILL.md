---
name: creative-platform
description: VidAU Creative 平台网关 —— 参考图上传、计费规则、预生成校验。所有生成操作的前置网关。
metadata:
  layer: L0-foundation
  requires: []
  tags: [foundation, platform, upload, billing]
---

# Creative Platform Gateway

## 计费

- 生成类操作（同步 + 异步）需要 **VIP** 并 **扣除积分**。
- 非 VIP → 分享购买链接（从 MCP 错误信息中获取）。
- 积分不足 → 引导用户充值。
- **免费操作**：`creative_estimate`、`creative_get_upload_instructions`、`creative_upload_reference`、`creative_generate_script`、`creative_list_models`、`creative_mux_bgm_into_video`。

## Prompt Gate（所有生成 MCP 调用的强制前置）

**任何图片和视频 MCP 调用都必须先经过 Prompt Skill**，禁止将用户原始文本作为 `prompt` 直接传入：

| MCP 工具 / 输出 | 必须先加载 |
|------------------|-----------|
| `creative_generate_image`, `direct_image`, `batch_variants` | **creative-gpt-image2-prompt** |
| `creative_generate_video`, `creative_image_to_video`, `creative_first_frame_to_video`, `direct_video` | **creative-seedance2-prompt** |
| script2film Final Video Spec（每个镜头的视觉描述） | **creative-seedance2-prompt** |

工作流：加载 Prompt Skill → 产出 paste-ready prompt → 用该 prompt 调用下游 Skill 或 MCP。

## 标准流程

1. （可选）调用 `creative_estimate` 估算时间和积分
2. 调用 `creative_generate_*` / `creative_submit_*`

## 本地参考图上传（推荐方式）

图片/视频 MCP 工具仅接受 **HTTPS URL**（`reference_urls`），不接受原始文件字节。

**当用户有本地终端可用时：**

1. `creative_get_upload_instructions` — 获取 S3 预签名 PUT URL + curl 示例
2. 在 **用户机器上**，通过终端/curl PUT 文件（`Content-Type` 按响应要求）
3. 上传完成后，使用返回的 `upload.file_url`
4. 将其传入 `creative_generate_image.reference_urls` 或 `creative_image_to_video.reference_image_urls`

**不要**在远程 MCP 中使用 `local_path`（会 ENOENT）。**不要**默认通过 MCP 发送大 base64；仅当无本地终端可用时才使用 `creative_upload_reference` 作为兜底。

## 注意事项

- 单张参考图最大约 **25MB**；支持 jpg/png/webp/gif/bmp
