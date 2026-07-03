---
name: tiktok-hermes-ad-inspection
description: strict tiktok ad inspection and anomaly diagnosis for hermes on the vidau platform. use when the user asks to inspect tiktok advertising data or enters trigger terms including 查看数据, 投放数据, 看一下数据, 数据分析, 数据建议, 数据监控, 排查数据, 观察数据, 素材分析, 数据情况, 监控数据, 数据巡检, or after successful ad creation. supports spend, conversions, cpa, roas, budget exhaustion, creative fatigue, daily inspection inquiry, hermes json, and safe ai assistant ui payloads. outputs recommendations only and never executes budget, bid, targeting, status, creative, notification, or schedule changes.
---

# TikTok Hermes Ad Inspection

## Scope

Use this skill only for TikTok ad inspection workflows on the Vidau Hermes self-developed platform:

```text
https://tiktok.vidau.ai/
```

Default display target:

```text
https://tiktok.vidau.ai/Ai
AI助手页面
```

Default behavior:
- Inspect TikTok only. Do not inspect Facebook, Google, Meta, or other channels.
- Analyze synced ad data and generate anomaly diagnosis, risk level, evidence, and optimization recommendations.
- Do not execute any spend-affecting or delivery-affecting change.
- Even after user confirmation, keep this skill as recommendation-only. If execution is needed, hand off to a separate Hermes operation module.
- Daily automatic inspection is inquiry-only: ask whether the user wants to enable it. Do not create a scheduled job from this skill.

## Trigger conditions

Trigger inspection when the user asks about TikTok ad data, monitoring, warnings, optimization suggestions, creative fatigue, or ad performance diagnosis, including these exact Chinese trigger terms:

```text
查看数据
投放数据
看一下数据
数据分析
数据建议
数据监控
排查数据
观察数据
素材分析
数据情况
监控数据
数据巡检
```

Also trigger after a TikTok ad creation task succeeds, then ask whether the user wants daily automatic inspection alerts.

## Required reference

Before giving any inspection conclusion, load:

```text
references/inspection_workflow.md
```

Use it as the source of truth for the 8 inspection modules, anomaly rules, data dependency, risk boundaries, output schema, priority handling, and exception handling.

## Workflow

1. Confirm the request is TikTok ad inspection on Vidau Hermes.
2. Resolve inspection scope from user input, current Hermes context, or recently created TikTok ad object.
3. If no advertiser or account context is available, return `clarification_required` and ask the user to select or provide the TikTok advertiser account.
4. Pull available real-time and historical data from Hermes synced data. Do not invent missing data.
5. Normalize time zone, currency, and formulas before judging anomalies.
6. Evaluate spend, conversion, CPA, ROAS, budget, and creative fatigue rules.
7. Merge related anomalies into a clear root issue when possible.
8. Sort issues by the multi-anomaly priority rules.
9. Return Hermes-compatible JSON plus customer-readable Chinese summary.
10. Ask whether the user wants to enable daily automatic inspection alerts when appropriate, but do not create a schedule.

## Hermes output contract

Prefer structured JSON for Hermes. Keep customer-facing Chinese concise.

```json
{
  "reply_type": "inspection_result | alert_required | clarification_required | sync_required | insufficient_data | blocked",
  "natural_language_summary": "不超过100个中文字符的摘要",
  "slot_updates": {},
  "ui_actions": [],
  "ui_payload": {
    "inspection_summary": {},
    "metric_cards": [],
    "anomaly_cards": [],
    "priority_order": [],
    "chart_specs": [],
    "recommended_actions": [],
    "daily_monitoring_inquiry": {}
  },
  "tool_followups": [],
  "risk_warnings": []
}
```

Each anomaly card must include:

```json
{
  "anomaly_type": "spend_fast | spend_slow | no_spend | spend_spike | conversion_drop | zero_conversion | clicks_normal_no_conversion | cpa_high | cpa_history_high | cpa_no_conversion_risk | roas_low | roas_history_drop | spend_no_revenue | budget_near_exhausted | budget_exhausted | total_budget_near_exhausted | budget_stop_delivery | creative_ctr_drop | creative_cvr_drop | creative_lifecycle_fatigue | insufficient_data",
  "object_level": "account | campaign | adgroup | ad | creative",
  "object_id": "string or unknown",
  "object_name": "string or unknown",
  "severity": "info | low | medium | high | critical",
  "evidence": [],
  "rule_matched": "exact rule name from references/inspection_workflow.md",
  "possible_causes": [],
  "recommended_checks": [],
  "suggested_actions": [],
  "execution_mode": "recommendation_only",
  "execution_allowed_in_this_skill": false
}
```

## Safety and honesty rules

- Do not claim that an action was completed unless a separate authorized Hermes execution module returns a confirmed result.
- Do not pause ads, change budgets, change bids, edit targeting, edit creatives, send external notifications, or create schedules.
- Do not fabricate spend, conversions, CPA, ROAS, CTR, CVR, budgets, balances, status, or historical averages.
- If a required metric is missing, mark that rule as `insufficient_data`.
- If sample size is small or the object is in learning phase, avoid strong conclusions and provide observation guidance.
- Phrase all optimization items as suggestions, not completed actions.
