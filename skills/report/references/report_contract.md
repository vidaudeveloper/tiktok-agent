# TikTok Report — JSON Output Contract

## Top-level schema

Used by `tiktok_hermes_report` module for all daily/weekly report responses.

```json
{
  "reply_type": "daily_report|weekly_report|sync_required|clarification_required|confirmation_required|blocked|insufficient_data|ui_input_unavailable|report_render_failed|report_generation_not_completed|login_required|current_chat_fallback",
  "module": "tiktok_hermes_report",
  "render_mode": "ui_payload|markdown_card|summary_only",
  "display_target": "ai_assistant|current_conversation",
  "display_attempts": 0,
  "max_display_attempts": 10,
  "natural_language_summary": "结论前置，不超过 80 个中文字符",
  "slot_updates": {
    "date_range": "{{date_range}}",
    "eligible_accounts": ["{{account1}}", "{{account2}}"],
    "report_type": "daily_report|weekly_report"
  },
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

## reply_type semantics

| reply_type | Meaning |
|------------|---------|
| `daily_report` | Full daily report generated |
| `weekly_report` | Full weekly report generated |
| `sync_required` | Data not synced; user must trigger sync |
| `clarification_required` | Missing required input (account, date range, etc.) |
| `confirmation_required` | High-risk action needs user approval |
| `blocked` | Cannot generate report (no eligible accounts, account not found, etc.) |
| `insufficient_data` | Not enough data to produce meaningful metrics |
| `ui_input_unavailable` | AI助手 input box not found or not interactable |
| `report_render_failed` | Request submitted but AI助手 returned no content |
| `report_generation_not_completed` | AI助手 page opened but no generation attempted |
| `login_required` | Page redirected to login; must authenticate first |
| `current_chat_fallback` | AI助手 display/rendering failed after 10 verified attempts; return readable report content in the current conversation |

## render_mode fallback chain

1. `ui_payload` — preferred, rich card/KPI rendering on AI助手 page
2. `markdown_card` — downgrade when ui_payload unsupported
3. `summary_only` — last resort, plain text only

## display fallback rule

Default `display_target` is `ai_assistant`. Switch to `current_conversation` only after 10 verified AI助手 display attempts fail. This fallback must never bypass login, authorization, positive-balance eligibility, sync, or data sufficiency checks, and must never expose complete raw JSON to the user.

## Daily report ui_payload structure

```json
{
  "ui_payload": {
    "type": "report",
    "subtype": "daily",
    "date": "2026-07-01",
    "eligible_accounts": [
      {"name": "FLY-Vidau-菲律宾GMAX", "id": "7644865578137600008", "balance": "USD 571.42"}
    ],
    "core_metrics": {
      "spend": "{{value}}",
      "cpa": "{{value}}",
      "ctr": "{{value}}",
      "cvr": "{{value}}",
      "roas": "{{value}}",
      "clicks": "{{value}}",
      "impressions": "{{value}}",
      "conversions": "{{value}}",
      "balance": "{{value}}"
    },
    "period_comparison": {
      "type": "previous_day",
      "summary": "{{text}}"
    },
    "problem_ads": [],
    "excellent_ads": [],
    "creative_highlights": [],
    "optimization_suggestions": [],
    "risk_reminders": [],
    "tomorrow_actions": []
  }
}
```

## Weekly report ui_payload structure

```json
{
  "ui_payload": {
    "type": "report",
    "subtype": "weekly",
    "date_range": "2026-06-25 ~ 2026-07-01",
    "eligible_accounts": [],
    "weekly_core_metrics": {
      "spend": "{{value}}",
      "cpa": "{{value}}",
      "ctr": "{{value}}",
      "cvr": "{{value}}",
      "roas": "{{value}}",
      "clicks": "{{value}}",
      "impressions": "{{value}}",
      "conversions": "{{value}}",
      "balance": "{{value}}"
    },
    "week_over_week_change": {
      "type": "previous_7_days",
      "summary": "{{text}}"
    },
    "problem_ads_review": [],
    "excellent_ads_review": [],
    "creative_learnings": [],
    "risk_review": [],
    "next_week_plan": []
  }
}
```

## Metric honesty rules

| Condition | Output |
|-----------|--------|
| Denominator is zero (CTR, CVR, CPA, CPC, CPM, ROAS) | `N/A` |
| No conversion value / GMV | `ROAS: N/A` |
| No spend or conversions | Omit CPA entirely |
| No ad-level data | `problem_ads: []`, `excellent_ads: []` |
| No creative data | `creative_highlights: []` (or "暂无素材维度数据") |
| Sample too small | Append "样本量不足，不能做强结论" |
| Balance unavailable | Show as `"-"` rather than $0 |
