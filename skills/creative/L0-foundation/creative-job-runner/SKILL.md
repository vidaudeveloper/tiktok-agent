---
name: creative-job-runner
description: 创作任务提交后在对话中追踪状态（不在对话中 sleep/自动轮询）
metadata:
  layer: L0-foundation
  requires: []
  tags: [foundation, async, tracking]
---

# Creative Job Runner — 提交后即时返回 + 对话追踪

**所有** VidAU 图片/视频 Skill 都必须遵循此追踪协议。

**不要**在对话中 `sleep` 或循环调用 `creative_get_job` 直到完成（script2film 可能需要 10–30 分钟）。

## 自动启用时机

| MCP 工具 | 追踪模式 |
|----------|----------|
| `creative_submit_*` | **对话追踪** — 提交后立即回复；用户可在本线程随时询问进度 |
| `creative_generate_image` / `_video` / `image_to_video` | **同步** — 调用前告知预计时间；完成后读取 `tracking.user_message` |
| 任何返回 `job_id` 的工具 | 必须使用对话追踪 — 不自动轮询 |

## 异步任务标准流程（必须遵循）

1. **提交时** — 从响应中读取 `tracking.user_message`，**立即发送给用户**（job_id、预计积分/时间）；告知用户可在本线程随时询问进度。
2. **结束本轮** — 当 `tracking.should_continue_polling` 为 `false` 时，**不要**调用 `creative_get_job`，**不要** `sleep`。
3. **进度查询** — 用户在本线程询问时，调用一次 `creative_get_job` 或 `creative_list_jobs` 并回答；**不要**自动 sleep/轮询循环。
4. **用户追问** — 每次问题只调用一次 `creative_get_job` 或 `creative_list_jobs`；仍然**不要**轮询循环。
5. **取消** — 用户说"取消"→ `creative_cancel_job`。

## 旧行为 vs 现行行为

| ❌ 旧行为（禁止） | ✅ 新行为（必须）|
|-------------------|---------------|
| 每 10s/30s/60s 轮询 `creative_get_job` | 提交后立即返回 |
| 在对话中 `sleep` | 仅在用户询问时查询 |
| 等最终视频出来后才回复 | 提交即确认 + job_id；用户可随时跟进 |

## 同步生成（creative-direct）

**调用前**：告知用户"正在生成，约 1–3 分钟，请稍候…"。

工具返回后，交付 `tracking.user_message` + artifact URLs。VIP 或积分错误 → 遵循 **creative-platform** 计费规则。

## Agent 行为规范

- 严格遵循 `tracking.agent_action`；如果说"no polling"，不得忽略。
- **不要**发送任务面板链接；进度保留在本对话中。
- 用户说"我的任务"→ `creative_list_jobs` 并在对话中展示列表。

## 交付

- 任务 `completed` 且用户仍在当前线程时，交付 `artifacts[0].urls.download` + 本地保存提示。
- `delivery.mode=both`（默认）：URL + `local.suggested_filename` / `suggested_subpath`
