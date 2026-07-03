---
name: tiktok-hermes-report
description: Use when generating TikTok daily or weekly advertising reports in Hermes for Vidau. Applies to client-readable TikTok performance reports, daily reports, weekly reports, account/campaign/ad/adgroup/creative summaries, period-over-period changes, problem ads, excellent ads, creative highlights, risk reminders, optimization recommendations, and next-day or next-week actions. Must use only authorized TikTok ad accounts on https://tiktok.vidau.ai/accounts with positive balance and display report output primarily in the AI Assistant page on https://tiktok.vidau.ai/; if AI Assistant rendering fails after 10 verified attempts, return the readable report content in the current conversation instead of raw JSON.
license: MIT
metadata:
  hermes:
    version: 2.2.0
    tags: [tiktok, ads, report, daily-report, weekly-report, vidau, hermes]
    related_skills: [tiktok-hermes-ad-inspection, tiktok-hermes-ad-creation, tiktok-hermes-agent-knowledge]
---

# TikTok Hermes Daily / Weekly Report Skill

## Reference files

- `references/report_contract.md` — JSON output contract, ui_payload schemas, metric honesty rules
- `references/test_scenarios.md` — Verified test scenarios, edge cases, page structure, browser considerations
- `references/report_workflow_cn.md` — 中文投放报告流程补充规范：使用场景、输入要求、数据依赖、执行步骤、判断规则、风险边界、输出格式、异常处理


## Report workflow supplement

When a user asks to verify or supplement the report workflow, consult `references/report_workflow_cn.md`. That file is the normative Chinese workflow supplement for:
- 使用场景
- 输入要求
- 数据依赖
- 执行步骤
- 判断规则
- 风险边界
- 输出格式
- 异常处理

Always treat this skill as a TikTok 投放报告 skill, not an ad creation skill. The purpose is to automatically generate user-readable TikTok advertising performance reports inside the `AI助手` menu.

## 1. Scope

Generate customer-readable TikTok daily and weekly advertising reports inside Hermes for the Vidau self-developed platform.

Use this skill for:
- TikTok 日报、周报、客户投放报告。
- 今日 / 当前日期核心数据、近 7 天周报数据。
- 花费、CPA、点击率（CTR）、CVR、ROAS、点击、展示、转化、余额。
- 环比变化、问题广告、优秀广告、素材亮点、优化建议、风险提醒、明日广告动作 / 下周动作。

Do not use this skill for:
- Creating, publishing, pausing, deleting, or editing ads.
- Adjusting budget or bid.
- Sending external notifications without user confirmation.
- Non-TikTok media channels.

## 2. Platform boundary

All report workflows must be scoped to Vidau TikTok platform pages:

- Authorized account source page: `https://tiktok.vidau.ai/accounts`
- Report display page: `https://tiktok.vidau.ai/`
- Report display area: `AI助手` page or panel by default
- Fallback display area: current conversation only after 10 verified AI助手 display attempts fail

Do not route the report result to other Vidau pages unless the user explicitly asks for export or external notification and confirms the action. The only automatic fallback is current-conversation display after 10 failed AI助手 rendering attempts.

## 3. Account selection and authorization rules

### 3.1 Default account scope

Daily and weekly reports must be generated only for accounts shown as authorized on:

`https://tiktok.vidau.ai/accounts`

If the user does not specify an ad account:
1. Open or use the authorized accounts page.
2. Read all authorized TikTok ad accounts on the page.
3. Read the `余额` field for each account.
4. Keep only accounts with `余额 > 0`.
5. Generate the report for all eligible authorized accounts.
6. Record this default in `slot_updates`.

If no authorized account with `余额 > 0` exists, return `blocked` or `clarification_required`; do not generate a report.

### 3.2 User-specified account

If the user specifies an account name or account ID:
1. Go to `https://tiktok.vidau.ai/accounts`.
2. Use the page search box labeled `搜索账户名或 ID`.
3. Search the specified account name or ID.
4. If the account is found, verify that it is authorized and its `余额 > 0`.
5. Automatically trigger or request latest data sync for the specified account before generating the report. Prefer `同步实体数据`; after sync, use the latest available account, campaign, ad group, ad, creative, performance, and balance data.
6. If the account is unauthorized, missing, balance is not greater than 0, or latest data sync fails, do not generate a normal report.

### 3.3 Search miss recovery

If the specified account cannot be found in the `搜索账户名或 ID` box:
1. Click `拉取最新授权`.
2. Wait for the refresh result.
3. Trigger or request `同步实体数据` after authorization refresh completes.
4. Search the account again.
5. If still not found, return `clarification_required` or `blocked` with a clear reason.

Never claim the account exists, is authorized, or has positive balance unless the page, context, or tool result confirms it.

## 4. Data date defaults

### 4.1 Daily report

If the user asks for 日报 and does not specify a date:
- Use the current date as the report date.
- Display the report data date clearly as `当前日期`.
- Record the default in `slot_updates.date_range`.

### 4.2 Weekly report

If the user asks for 周报 and does not specify a date range:
- Use the most recent 7 days as the report period.
- Display the report date range clearly.
- Record the default in `slot_updates.date_range`.

### 4.3 Comparison period

For daily reports:
- Prefer comparison with previous day.
- If previous day data is unavailable, compare with recent 7-day daily average and label it clearly.

For weekly reports:
- Prefer comparison with the previous 7 days.
- If unavailable, mark comparison data as `unknown` rather than fabricating it.

## 5. Data requirements

Before generating a full report, validate:

- `advertiser_id`
- `advertiser_name`
- account authorization status from `https://tiktok.vidau.ai/accounts`
- `余额` and whether `余额 > 0`
- report type: `daily_report` or `weekly_report`
- date range
- data sync status
- performance data for the requested report period from `https://tiktok.vidau.ai/`
- comparison data from `https://tiktok.vidau.ai/` if period-over-period changes are required
- ad/adgroup/campaign data if problem or excellent ads are required
- creative data if creative highlights are required

If required data is absent, return `sync_required`, `clarification_required`, `blocked`, or `insufficient_data`. Do not fabricate missing metrics.

## 6. Accuracy rules

- Do not fabricate spend, CPA, CTR, CVR, ROAS, clicks, impressions, conversions, conversion value, budget, balance, account status, ad status, or creative performance.
- `CTR` and `点击率` are the same metric. In Chinese output, use `点击率（CTR）`.
- If conversion value / GMV is missing, output ROAS as `unknown` or `N/A`; do not calculate it.
- If ad-level data is missing, do not identify problem ads or excellent ads as factual conclusions.
- If creative-level data is missing, do not generate factual creative highlights. You may output `素材亮点：暂无素材维度数据，不能做强结论`.
- If sample size is too small, state `样本量不足，不能做强结论`.
- Optimization suggestions must be grounded in actual data, page results, tool results, or known safe report rules.
- Any external notification or export/send action requires user confirmation.

## 7. Metric formulas

Only calculate when numerator and denominator are available and denominator is nonzero:

- CTR = clicks / impressions
- CVR = conversions / clicks
- CPA = spend / conversions
- ROAS = conversion_value / spend
- CPC = spend / clicks
- CPM = spend / impressions * 1000

If denominator is zero, output `N/A` and explain briefly only when relevant.

## 8. AI assistant report generation protocol

Report generation is NOT complete when the AI助手 page is merely opened. The following hard rules govern the end-to-end report generation flow.

### 8.1 Report generation is an interactive flow, not a page navigation

Opening `https://tiktok.vidau.ai/` is only the first step. The report is generated only when Hermes:

1. Navigates to the AI助手 page.
2. Locates the chat input box (`"输入你的问题或指令..."`).
3. Types the report generation request into the input box (e.g., `"生成 TikTok 日报"` or `"生成 TikTok 周报"`).
4. Submits the request by pressing Enter or clicking the send button.
5. Waits for the AI助手 conversation area to display the report response content.
6. Reads and verifies that the conversation area contains actual report data (metrics, tables, cards).
7. Only then concludes the report was successfully generated.
8. The report must also be visible under the `AI助手` menu by default; do not display it in another Vidau menu.
9. If AI助手 rendering/display fails, retry the submit → wait → read → verify flow up to 10 total attempts before using current-conversation fallback.

### 8.2 Failure states

| Condition | Reply type | Meaning |
|-----------|------------|---------|
| AI助手 input box is not found or not interactable | `ui_input_unavailable` | Cannot submit report request — UI is broken or missing |
| Report request was submitted but no response content appears in the conversation area before 10 attempts are exhausted | `report_render_failed` | Backend processed the request but failed to render the report in the conversation area for the current attempt |
| AI助手 page was opened but no report generation was attempted | `report_generation_not_completed` | Only "opened page" — must not be equated to "report generated" |
| AI助手 cannot display the report after 10 verified attempts | `current_chat_fallback` | Return the readable report content in the current conversation, using Markdown card format first and concise summary only if Markdown is unavailable |

### 8.3 Prohibited behavior

- Navigating to `https://tiktok.vidau.ai/` and claiming the report is "ready" without submitting the request is **prohibited**.
- Reporting "successfully opened AI助手 page" as a proxy for "report generated" is **prohibited**.
- Claiming the report was generated without reading the conversation area content is **prohibited**.

### 8.4 End-to-end test protocol

Each report generation flow must include these verification steps:

```
1. Navigate to AI助手 page → verify page loads
2. Locate chat input box → verify it exists and is interactable
3. Type report request → verify text entered correctly
4. Submit request → verify submission was accepted
5. Wait for response → poll conversation area for report content
6. Read response → verify report structure and metrics
7. If display/response verification fails, repeat steps 2–6 until either the report displays or 10 total attempts have been made
8. After 10 failed AI助手 display attempts, return `current_chat_fallback` and show the readable report in the current conversation
```

If step 1 fails but the page redirects to login, return `login_required` — do not skip to fallback. Login, authorization, zero-balance, and missing-data blocks are not rendering failures and must not be bypassed by current-conversation fallback.

### 8.5 Action sequence for report generation

```json
{
  "reply_type": "daily_report|weekly_report",
  "module": "tiktok_hermes_report",
  "generation_phase": "navigate|submit|waiting|completed|failed",
  "natural_language_summary": "报告生成阶段状态",
  "slot_updates": {},
  "missing_slots": [],
  "ui_actions": [
    {
      "action": "navigate_and_type",
      "url": "https://tiktok.vidau.ai/",
      "input_ref": "输入你的问题或指令...",
      "submit_text": "{{report_request_text}}",
      "reason": "在 AI 助手输入框中提交报告生成请求"
    },
    {
      "action": "wait_for_response",
      "target_area": "conversation_area",
      "timeout_seconds": 60,
      "reason": "等待 AI 助手对话区返回报告内容"
    },
    {
      "action": "read_conversation_content",
      "target": "latest_assistant_message",
      "reason": "读取 AI 助手返回的报告内容以验证生成成功"
    }
  ],
  "display_attempts": 0,
  "max_display_attempts": 10,
  "fallback_display_target": "current_conversation_after_10_failed_ai_assistant_attempts",
  "ui_payload": {},
  "tool_requests": [],
  "risk_warnings": [],
  "blocked_reason": null,
  "generation_error": null
}
```

## 9. Report display rules

Reports must be displayed first in `https://tiktok.vidau.ai/` under the `AI助手` page / panel. If and only if AI助手 rendering/display fails after 10 verified attempts, display the readable report content in the current conversation.

### 9.1 Preferred rendering

If the AI Assistant page supports `ui_payload` rendering:
- Output structured JSON with `ui_payload`.
- Use cards, KPI grids, comparison blocks, ranked lists, insight cards, recommendation lists, and action lists.

### 9.2 Markdown fallback

If the AI Assistant page does not support `ui_payload` rendering:
- Automatically downgrade to Markdown card format.
- Keep the report customer-readable.
- Do not output raw JSON directly to the user interface.

### 9.3 Summary fallback

If Markdown also cannot be displayed:
- Output only a concise natural-language summary.
- Include no complete raw JSON in the user interface.
- Mention that a full structured report requires AI Assistant rendering or export support.

### 9.4 Current-conversation fallback after 10 AI助手 attempts

Use the current-conversation fallback only when all conditions are true:
- The user request is otherwise valid.
- Eligible authorized accounts with `余额 > 0` have been confirmed.
- Required data has been read or unavailable fields have been marked as unknown.
- Hermes has attempted to render/display the report in `AI助手` 10 times.
- Each attempt included submit → wait → read → verify, or failed at a clearly recorded AI助手 display step.

When using this fallback:
- Set `reply_type` to `current_chat_fallback`.
- Set `display_target` to `current_conversation`.
- Prefer Markdown card format.
- Use concise summary only if Markdown is unavailable.
- Do not output complete raw JSON.
- State that AI助手 display failed after 10 attempts, so the report is returned in the current conversation.

## 10. Hermes output protocol

For Hermes orchestration, use the following structure internally. When the UI cannot render JSON, apply the fallback rules in section 9 before displaying to the user.

```json
{
  "reply_type": "daily_report|weekly_report|sync_required|clarification_required|confirmation_required|blocked|insufficient_data|ui_input_unavailable|report_render_failed|report_generation_not_completed|login_required|current_chat_fallback",
  "module": "tiktok_hermes_report",
  "render_mode": "ui_payload|markdown_card|summary_only",
  "display_target": "ai_assistant|current_conversation",
  "display_attempts": 0,
  "max_display_attempts": 10,
  "natural_language_summary": "结论前置，不超过 80 个中文字符",
  "slot_updates": {},
  "missing_slots": [],
  "ui_actions": [],
  "ui_payload": {},
  "markdown_fallback": "",
  "summary_fallback": "",
  "tool_requests": [],
  "risk_warnings": [],
  "blocked_reason": null,
  "unknowns": []
}
```

Recommended page actions:

```json
[
  {
    "action": "open_url",
    "url": "https://tiktok.vidau.ai/accounts",
    "page_key": "vidau_tiktok_accounts",
    "reason": "读取已授权广告账户和余额"
  },
  {
    "action": "open_url",
    "url": "https://tiktok.vidau.ai/",
    "page_key": "vidau_ai_assistant",
    "reason": "在 AI 助手页面展示数据报告"
  }
]
```

If user-specified account is not found:

```json
{
  "action": "click",
  "target_text": "拉取最新授权",
  "page_key": "vidau_tiktok_accounts",
  "reason": "指定账户未搜索到，先拉取最新授权"
}
```

After authorization refresh:

```json
{
  "action": "run_or_request",
  "name": "同步实体数据",
  "reason": "拉取最新授权后需要同步账户、广告系列、广告组、广告、素材等实体数据"
}
```

## 11. Required report sections

### 11.1 Daily report

Include sections in this order when data supports them:
1. `report_date`: 当前日期 or specified date.
2. `eligible_accounts`: authorized accounts with `余额 > 0`.
3. `core_metrics`: 花费、CPA、点击率（CTR）、CVR、ROAS、点击、展示、转化数、余额。
4. `period_comparison`: default previous day comparison.
5. `problem_ads`: only if ad/adgroup data exists.
6. `excellent_ads`: only if ad/adgroup data exists.
7. `creative_highlights`: only if creative data exists.
8. `optimization_suggestions`: evidence-based actions.
9. `risk_reminders`: budget, spend, conversion, CPA, ROAS, status, balance, or data quality risks.
10. `tomorrow_actions`: concrete next-day actions.

### 11.2 Weekly report

Include sections in this order when data supports them:
1. `report_date_range`: recent 7 days or specified range.
2. `eligible_accounts`: authorized accounts with `余额 > 0`.
3. `weekly_core_metrics`: spend, CPA, CTR, CVR, ROAS, clicks, impressions, conversions, balance.
4. `week_over_week_change`: compare with previous 7 days by default.
5. `problem_ads_review`: recurring or high-impact problems.
6. `excellent_ads_review`: scalable winners.
7. `creative_learnings`: reusable creative patterns if supported by creative data.
8. `risk_review`: budget, data, conversion, ROAS, creative fatigue, account balance risk.
9. `next_week_plan`: budget allocation suggestions, creative test plan, tracking fixes, data-sync actions.

## 12. Problem ad logic

Only identify problem ads from actual ad, ad group, campaign, or creative-level data. Prefer evidence such as:

- high spend with zero or low conversions
- CPA above target or historical baseline with enough conversions
- ROAS below target or historical baseline with enough spend and conversion value
- CTR/CVR deterioration with sufficient impressions/clicks
- rejected, limited, paused, or abnormal delivery status returned by system

## 13. Excellent ad logic

Only identify excellent ads when supported by metrics such as:

- CPA below target or historical baseline
- ROAS above target or historical baseline
- stable conversions with controlled spend
- CTR/CVR strong relative to account average
- sufficient sample size

## 14. Creative highlights logic

Only output factual creative highlights when creative-level metrics or metadata exists. If only performance data exists, frame as `可复盘方向` rather than asserting exact creative reasons.

## 15. Markdown fallback template

Use this only when `ui_payload` cannot render.

```markdown
# TikTok 投放日报 / 周报

## 数据范围
- 报告类型：{{report_type}}
- 数据日期：{{date_range}}
- 账户范围：{{eligible_account_scope}}
- 账户筛选：仅包含 https://tiktok.vidau.ai/accounts 中已授权且余额大于 0 的广告账户

## 核心数据
- 花费：{{spend}}
- CPA：{{cpa}}
- 点击率（CTR）：{{ctr}}
- CVR：{{cvr}}
- ROAS：{{roas}}
- 点击：{{clicks}}
- 展示：{{impressions}}
- 转化：{{conversions}}
- 余额：{{balance}}

## 环比变化
{{comparison_summary}}

## 问题广告
{{problem_ads_or_data_gap}}

## 优秀广告
{{excellent_ads_or_data_gap}}

## 素材亮点
{{creative_highlights_or_data_gap}}

## 优化建议
{{optimization_suggestions}}

## 风险提醒
{{risk_reminders}}

## 明日广告动作 / 下周动作
{{next_actions}}
```

## 16. External send / export confirmation

If `notification_channel` is not `none`, return `confirmation_required` before sending.

If the user asks to export Excel/PDF, create an export draft or confirmation step first if the platform treats export as a controlled action.

```json
{
  "reply_type": "confirmation_required",
  "module": "tiktok_hermes_report",
  "natural_language_summary": "发送报告到外部渠道前需要确认。",
  "slot_updates": {"notification_channel": "{{channel}}"},
  "ui_actions": [
    {"action": "open_confirmation_modal", "title": "确认发送报告", "description": "是否将报告发送到 {{channel}}？"}
  ],
  "ui_payload": {"confirmation": {"requires_user_confirm": true}},
  "risk_warnings": ["外部通知属于高风险动作，必须确认后执行。"]
}
```

## 17. Do-not-do list

- Do not use unauthorized accounts.
- Do not include accounts with `余额 <= 0` in default reports.
- Do not fabricate balances.
- Do not generate reports for accounts not found after search + authorization refresh + entity sync.
- Do not display raw JSON in the user interface if UI rendering fails.
- Do not use current-conversation fallback before 10 verified AI助手 display attempts fail.
- Do not use current-conversation fallback to bypass login, authorization, balance, or missing-data blocks.
- Do not send reports externally without confirmation.
- Do not claim `拉取最新授权` or `同步实体数据` succeeded unless the page or tool result confirms success.
- Do not claim the report is "generated" when only the AI助手 page was opened but no request was submitted.
- Do not claim the report is "generated" without reading the AI助手 conversation area content.
- Do not treat "successfully opened AI助手 page" as equivalent to "report generated".
- Do not skip the submit → wait → verify flow and jump directly to a success conclusion.
