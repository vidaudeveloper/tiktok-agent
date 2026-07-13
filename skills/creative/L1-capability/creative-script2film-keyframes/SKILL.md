---
name: creative-script2film-keyframes
description: 脚本转视频（首/尾帧模式）—— per-shot keyframes 驱动 Seedance 转场动画
metadata:
  layer: L1-capability
  requires: [creative-job-runner, creative-platform, creative-narrative-router, creative-seedance2-prompt, creative-script2film]
  tags: [storyboard, async, script2film, keyframes, first-frame, one-click]
---

# Creative Script2Film — 首/尾帧模式视频

共享 **creative-script2film** 的完整 workflow（reference-image-to-video），但 per-shot video 使用 **first-frame / last-frame** 模式替代 reference 模式。

> **Prompt Gate**：同 **creative-script2film** —— 加载 **creative-seedance2-prompt` 在提交前丰富 Final Video Spec 镜头视觉和帧间运动语言。

## 使用时机

| 场景 | 推荐 Skill |
|------|-----------|
| 产品/人物必须紧密匹配参考图；多参考约束 | **creative-script2film**（reference） |
| 平滑镜头间转场；可控运动 | **本 Skill**（first_last_frame） |
| 单镜头从静态关键帧展开动作；无需结束帧 | 本 Skill + `video_mode: first_frame` |
| 单个短 clip（非多镜头交付物） | **creative-direct** + `creative_first_frame_to_video` |

## Flow（与 reference 版本共享 script2film workflow）

同 **creative-script2film** 服务端流水线 —— **仅 `video_mode` 不同**（本 Skill 默认 `first_last_frame`）。

1. `creative_estimate` with `workflow_type=script2film`
2. **`creative_submit_script2film_keyframes`**（默认 `video_mode=first_last_frame`）：
```json
{
  "script": "<user script>",
  "target_duration_sec": 30,
  "aspect_ratio": "9:16",
  "reference_image_urls": ["<product hero image>"],
  "brief": { "product": "..." },
  "client_request_id": "<uuid>"
}
```
3. **creative-job-runner** —— 提交后**立即**发送 `tracking.user_message`；**不要** sleep/轮询；**artifacts[0]** 为最终视频（除非 `skip_bgm: true`）

## 首/尾帧原理（与 reference 版本的唯一差异）

| 阶段 | Reference 版本 | 首/尾帧版本（本 Skill） |
|------|---------------|------------------------|
| Key elements + element refs | ✅ 相同 | ✅ 相同 |
| Per-shot keyframes | element anchors + woven prompt | ✅ 相同 |
| Per-shot video | Seedance **reference**（element imgs + keyframe） | Seedance **first_last_frame**（首/尾 keyframes only） |

Per-shot video：
- **First frame** = 该 shot 的 keyframe（已被 element anchors 约束）
- **Last frame** = 下一个 shot 的 keyframe（最后一个 shot：last frame = 本 shot 的 keyframe）

## 同步单段（非完整交付物）

已有首/尾帧图且只需一个 clip 时：
```
creative_first_frame_to_video:
  prompt: "..."
  first_frame_url: "<first frame url>"
  last_frame_url: "<last frame url>"   # optional; omit for first_frame mode
  duration_sec: 5
  aspect_ratio: "9:16"
```

## 任务失败与重试

同 **creative-script2film**：若镜头因 Seedance **版权/IP** 或**真脸隐私** 拦截，引导用户**修改该镜头 prompt / 更换参考图**，然后用新 `client_request_id` 重新提交。
