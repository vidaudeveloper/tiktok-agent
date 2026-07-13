---
name: creative-seedance2-prompt
description: Seedance 2.0 / 即梦 视频 Prompt 工程 —— 任何视频 MCP 调用前的必经门槛。用于 `creative_generate_video`、`creative_image_to_video`、`creative_first_frame_to_video`、script2film 镜头描述等。
metadata:
  layer: L0-foundation
  requires: []
  tags: [foundation, prompt, seedance, video, jimeng]
---

# Creative Seedance 2.0 Prompt

生产级 **Seedance 2.0 / 即梦** 视频 Prompt 工程。

> **强制门槛**：在任何视频 MCP 调用前**必须加载本 Skill**（`creative_generate_video`、`creative_image_to_video`、`creative_first_frame_to_video`、`creative_submit_workflow` 含 `direct_video`，或丰富脚本镜头视觉描述）。**绝不要**将用户原始文本直接作为 `prompt` 传入。

## 加载时机

| 触发动作 | 行为 |
|----------|------|
| 任何视频 MCP 调用 | 先生成 Seedance prompt → 传给 MCP 的 `prompt` 参数 |
| script2film / keyframes 脚本编写 | 丰富每个镜头的 **visual + motion + audio** 描述到 Final Video Spec |
| 用户说 即梦 / Seedance / 视频提示词 / AI video | 加载并起草或优化 prompts |
| 参考图生视频 | 在 prompt 中包含 reference role 语义 |

## Agent 工作流（必须遵循）

1. **解析意图** — 广告 / 短剧 / 产品展示 / 转场 / 扩展 / 编辑 / MV / 科普
2. **收集约束** — 时长（每 clip 4–15s）、比例、参考资源（URL 已上传）、语言（默认 ZH 用于即梦原生文案；用户说英语时用 EN）
3. **按需 Read 参考资料**
4. **起草 prompt**（使用下方结构）
5. **合规检查** — 参考图中无真人面部；无 IP/品牌/名人名称；无冲突的运镜指令
6. **交付** — 输出一个 copy-ready prompt 块；然后调用下游 Skill / MCP

## Prompt 结构（SCELA + 时间分段）

对于 **>8s** 的 clip，使用分段计时：

```
[主体/人物] + [场景] + [动作/运动] + [运镜] +
[0–Ns 分时段描述] + [转场/特效] + [音效/对白] + [风格/氛围]
```

| 区块 | 内容 |
|------|------|
| **S** Subject | 谁/什么在画面上；关联 reference roles |
| **C** Camera | 推/拉/摇/移/跟/POV/全景 — 每节一个主导运镜 |
| **E** Effect | 粒子、变速、match cut、产品旋转等 |
| **L** Light | 一天中的时间、主光/轮廓光、色调 |
| **A** Audio | 环境音效/环境声/对白语气 — **script2film VO 场景中不要写 BGM**（服务端后续添加 BGM） |

### @ 参考图语法（即梦原生）

当用户素材映射为 references 时，显式标注每个 role：

```
@图片1 作为首帧，产品细节参考 @图片2，运镜参考 @视频1 的慢推
```

VidAU MCP 使用 HTTPS URL — 在 Agent 文本中将 references 描述为 `reference_image_urls[0] as product hero` 并将相同语义编织进 prompt 散文。

## VidAU 特定规则

| 场景 | 规则 |
|------|------|
| **script2film + voiceover** | 每个镜头视频 prompt：环境音效/foley OK；**严禁在 prompt 中写 BGM / soundtrack / vocals**（服务端追加此规则） |
| **script2film 默认音频** | 仅环境 SFX；BGM 由服务端在 concat 后混入 |
| **direct / single clip** | `generate_audio=true`（默认）；仍避免在 prompt 中要求完整 soundtrack — 让模型产出 in-shot SFX |
| **reference 模式** | 最多 9 张参考图；标明每张图的 role（产品/人物/场景/风格） |
| **first/last frame** | Prompt 描述**帧之间**的运动；不要与帧构图矛盾 |
| **版权安全** | 抽象电影化语言；不用 Disney/Marvel/名人/品牌口号；产品 = 用户实际 SKU |

## 输出格式

始终按以下顺序返回给用户（以及传给下游 MCP）：

```markdown
### Seedance Prompt
<paste-ready prompt text>

### Strategy
- Duration: Ns | Aspect: 9:16 | Mode: reference | first_last_frame | text_to_video
- References: [role → asset summary]
- Compliance: [what was avoided]
```

然后以 `prompt` = paste-ready text调用下游 MCP。

## 常见错误预防

| 错误做法 | 正确做法 |
|----------|----------|
| 模糊"参考视频1" | 明确：运镜/动作/节奏/特效 |
| 4s 内塞太多内容 | 减少节拍或延长时长 |
| 冲突的运镜指令 | 每段只有一个主导运动 |
| 参考图中有真脸 | 替换为产品-only 或风格化人物 |
| VO script2film 镜头里写 BGM | 去除音乐词汇；只保留 foley |
