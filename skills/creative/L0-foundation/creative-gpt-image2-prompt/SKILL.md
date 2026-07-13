---
name: creative-gpt-image2-prompt
description: GPT Image 2 / gpt-image-2 Prompt 工程 —— 任何图片 MCP 调用前的必经门槛。用于 `creative_generate_image`、`batch_variants`、`direct_image` workflows 等。
metadata:
  layer: L0-foundation
  requires: []
  tags: [foundation, prompt, gpt-image-2, image, openai]
---

# Creative GPT Image 2 Prompt

生产级 **GPT Image 2 (`gpt-image-2`)** Prompt 工程。

> **强制门槛**：在任何图片生成 MCP 调用前**必须加载本 Skill**（`creative_generate_image`、`creative_submit_batch_variants`、`creative_submit_workflow` 含 `direct_image`，或任何静态/关键帧生成的 `prompt`）。**绝不要**将用户原始文本直接作为 `prompt` 传入。

## 加载时机

| 触发动作 | 行为 |
|----------|------|
| `creative_generate_image` | 产 prompt → MCP `prompt` |
| `creative_submit_batch_variants` | 产基础 prompt + variant hooks |
| `direct_image` workflow | 产 prompt → `input.prompt` |
| 关键帧 / 产品静物 brief | 按用户区域输出英文或中文 prompt |
| 用户说 GPT Image / 生图 / 海报 / mockup | 起草或优化 |

## Agent 工作流（必须遵循）

1. **分类素材** — 产品主角 / 海报 / UI mockup / 信息图 / 摄影 / 人物设定 / 编辑
2. **按需 Read 参考资料**
3. **用六块协议构建 prompt**（见下方）
4. **文字审核** — 所有显示文字放在 `"quotes"` 中；中文需指定简体/繁体
5. **交付** copy-ready prompt → 然后调用下游 MCP

## 六块协议

顺序很重要——布局在主体细节之前：

```
1. Canvas & layout   — aspect ratio, grid, zones, device frame
2. Subject & task    — 要描绘什么，参考图的角色
3. Composition       — 前景/背景，层级关系，机位
4. Style & materials — 介质，配色，光照（分项控制）
5. Text & labels     — 精确引用字符串，易读性约束
6. Constraints       — 避免线，编辑不变量，无假 logo
```

### Block 1 示例

- `Vertical 9:16 product hero poster…`
- `Square 1:1 e-commerce white-background packshot…`
- `Landscape 16:9 UI mockup, 1290×2796 phone frame…`

### Block 5（排版）

弱：`poster with brand name and price`

强：`Exact readable text: "无线耳机 Pro" / "主动降噪" / "¥399"`

## VidAU 特定规则

| 场景 | 规则 |
|------|------|
| 产品广告 | 清晰层级：产品最大；无假竞品 logo |
| batch_variants | 一个强的 base prompt + hook 变化在 `count` 提交中 |
| script2film keyframes | 写实 or 品牌锁定风格；匹配 `brief.narrative` 语调 |
| 编辑 / inpaint | 声明变换 + 显式保留清单 |
| 安全 | 无真人肖像（除非用户拥有参考图）；时尚内容适度 |

## 输出格式

```markdown
### GPT Image 2 Prompt
<paste-ready prompt>

### Strategy
- Category: product_hero | poster | ui_mockup | …
- Aspect: 9:16 | 1:1 | 16:9
- References: [role summary]
- Text blocks: [quoted strings]
```

然后以 `prompt` = paste-ready text调用 MCP。

## 快速检查清单

- [ ] 比例/布局最先声明
- [ ] 引号中的字面文本
- [ ] 材质、光照、配色分开描述
- [ ] 5–12 个具体场景名词（不是空洞形容词）
- [ ] 目标 `Avoid:` 行（1–3 项）
- [ ] 多 URL 时标注 reference role 索引
