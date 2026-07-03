---
name: tiktok-hermes-ad-creation
description: Use when creating TikTok ads on https://tiktok.vidau.ai, opening the Vidau ad creation page, explaining the delivery hierarchy, inspecting account/ad/creative counts, monitoring creation status, retrying retryable failures, or opening the warning center after success. Closed-loop workflow for Vidau self-built TikTok ad platform.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [tiktok, ads, vidau, ad-creation, closed-loop]
    related_skills: []
---

# TikTok Hermes Ad Creation

## Non-negotiable scope
- Only handle TikTok ad creation on the Vidau self-built platform.
- Use the page URL `https://tiktok.vidau.ai/campaigns/ads` as the canonical ad creation page.
- Do not claim an ad, campaign, ad group, or creative was created unless Hermes/backend tool results explicitly say so.
- Do not directly call TikTok Ads API unless the host runtime exposes an approved creation tool and the user has confirmed the action.
- The default behavior is: validate fields -> generate task/pre-fill payload -> instruct Hermes to open the Vidau ad creation page -> let the self-built platform submit.
- For interface submission failures, retry at most 3 times. Do not retry business-rule failures.

## Hermes execution model and digital-worker boundary
- Hermes is responsible for understanding user intent, validating ad-creation fields, generating a structured creation task, opening the Vidau page or panel, pre-filling ad information, waiting for explicit user confirmation, and monitoring creation status after submission.
- The actual ad creation must be executed by the Vidau self-built platform or backend task system. Hermes must not treat page opening, field prefill, button click, or task payload generation as proof that TikTok ads were created.
- The ad creation Agent must support digital-worker-style operation, but must not use blind page clicking as the primary execution method. Prefer structured JSON instructions that the Vidau platform can consume to perform page opening, field prefill, task submission, and status querying.
- Use UI operation only as a controlled execution layer when the host runtime supports it. UI actions must be state-aware, confirmation-aware, and subordinate to the structured JSON task flow.
- Keep the execution order as: understand intent -> validate fields -> generate structured JSON task/prefill payload -> open Vidau page or panel -> prefill available fields -> wait for user confirmation -> let Vidau backend submit -> monitor backend/page status.


## Closed-loop workflow goal
This skill must support a user-facing ad creation workflow closure for `https://tiktok.vidau.ai`:
1. Explain the TikTok delivery hierarchy before or during creation: account -> campaign -> ad group -> ad -> creative/material.
2. Inspect and show available account summary when backend/page state provides it: advertiser/account name, advertiser/account ID, intended ad count, creative/material count, and current task status.
3. Validate required fields and platform prerequisites before creation.
4. Open the Vidau TikTok ad creation page or relevant panel, prefill available fields, and require explicit user confirmation before final submit/publish.
5. After creation is submitted, monitor status until it reaches a terminal or user-actionable state.
6. If status is `failed`, report the backend/page failure reason and retry only if the failure is retryable, up to 3 total attempts.
7. If all 3 attempts fail, stop and tell the user the final failure reason plus which attempt failed.
8. If status is `success`, notify the user that delivery creation is complete and include account name, account ID, and successful ad count.
9. After success, ask whether the user wants to enable monitoring. If the user says yes, open the `预警中心` panel in Vidau.

## Vidau page targets
Canonical host: `https://tiktok.vidau.ai`.

Ad creation entry:
```json
{
  "action": "open_url",
  "url": "https://tiktok.vidau.ai/campaigns/ads",
  "page_key": "vidau_tiktok_ads",
  "target": "right_panel_or_current_tab",
  "reason": "打开 Vidau TikTok 广告创建页面"
}
```

Warning center entry after successful creation and user consent:
```json
{
  "action": "open_panel_or_route",
  "host": "https://tiktok.vidau.ai",
  "panel_label": "预警中心",
  "page_key": "vidau_warning_center",
  "fallback_route_candidates": ["/warning-center", "/alerts", "/monitoring/alerts"],
  "requires_user_confirmation": false,
  "reason": "用户确认需要开启监测后，打开预警中心面板"
}
```
If Hermes cannot find a reliable route or panel button for `预警中心`, stop and report `warning_center_entry_unknown` instead of guessing or clicking unrelated UI.

## UI operation safety rules
When Hermes controls the web UI on `https://tiktok.vidau.ai`, obey these rules strictly:
1. The same button or same semantic UI target may be clicked at most 2 times within one workflow.
2. After every click, read the actual page state before taking another action.
3. If the page state did not change after a click, do not repeat the click unless there is a clear user-approved reason and the total click count remains within the 2-click limit.
4. If a click causes a non-target field change, such as mistakenly selecting an extra country/region, stop immediately and report the unexpected change.
5. Submit, Next, Publish, budget changes, bid changes, delete, pause/resume, and external notification actions require explicit user confirmation.
6. Do not use a “click repeatedly until it works” strategy.
7. Page state must be judged by the visible/actual page result or backend status, not merely by a tool return such as `Clicked`.

## Closed-loop status monitoring protocol
After a creation task is submitted by Vidau or backend tools, do not assume success. Poll/read status through the available backend/page state tool and use the backend/page status as source of truth.

Required monitoring output fields:
```json
{
  "task_id": "{{task_id}}",
  "advertiser_name": "{{advertiser_name}}",
  "advertiser_id": "{{advertiser_id}}",
  "requested_ad_count": "{{requested_ad_count}}",
  "creative_count": "{{creative_count}}",
  "status": "processing|success|failed|partial_success|unknown",
  "successful_ad_count": "{{successful_ad_count}}",
  "failed_ad_count": "{{failed_ad_count}}",
  "failure_reason": "{{failure_reason}}",
  "retry_attempt": "{{retry_attempt}}",
  "max_retry_attempts": 3
}
```

Terminal status rules:
- `success`: notify completion and include account name, account ID, and successful ad count.
- `failed`: report reason. If retryable and retry_attempt < 3, retry. If retry_attempt = 3 or failure is not retryable, stop.
- `partial_success`: report successful and failed counts, give failed reasons, and ask the user whether to retry failed items only if backend says those failures are retryable.
- `processing` / `pending` / `review`: report current state and next check action. Do not claim completed delivery.
- `unknown`: report that Vidau/Hermes cannot determine status and request a backend/page status check.

## Retry contract for failed creation
Retry up to 3 total attempts only for retryable creation failures. Each retry must use the same idempotency key/task id when backend supports it, or a backend-approved retry endpoint. Never re-click submit repeatedly to simulate retry.

For each retry, output:
```json
{
  "reply_type": "retry_result",
  "retry_attempt": 1,
  "max_retry_attempts": 3,
  "retry_reason": "{{backend_failure_reason}}",
  "retry_method": "backend_retry_endpoint|task_retry|not_ui_repeat_click",
  "status_after_retry": "{{status}}"
}
```

Do not retry when failure is caused by missing fields, unauthorized account, non-Vidau account, invalid objective/budget/bid/event/material, insufficient balance, policy/audit rejection, unavailable creative, permission/scope missing, or any business validation error.

## Success notification template
When creation succeeds, tell the user in Chinese:
- `已完成投放创建`
- `投放账号：{{advertiser_name}}`
- `账户 ID：{{advertiser_id}}`
- `已成功投放 {{successful_ad_count}} 条广告`
- `素材数量：{{creative_count}}`
- Ask: `是否需要开启监测？如果需要，我将为你打开预警中心。`

Do not open `预警中心` until the user answers yes/需要/开启/打开.

## Monitoring consent behavior
If the user confirms monitoring after a successful creation, return an action to open the warning center panel:
```json
{
  "reply_type": "monitoring_open_requested",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "已为你打开预警中心，用于后续监测广告投放状态。",
  "ui_actions": [
    {
      "action": "open_panel_or_route",
      "host": "https://tiktok.vidau.ai",
      "panel_label": "预警中心",
      "page_key": "vidau_warning_center",
      "fallback_route_candidates": ["/warning-center", "/alerts", "/monitoring/alerts"],
      "reason": "用户确认开启监测"
    }
  ],
  "blocked_reason": null
}
```

## When to use
Use this skill for requests such as:
- 创建 TikTok 广告 / 批量创建广告 / 新建广告计划
- 打开广告创建页面 / 调起自研平台创建窗口
- 检查广告创建字段是否完整
- 展示账号名称、账号 ID、广告数量、素材数量
- 创建后查询状态：成功、失败、审核中、创建中、部分成功
- 接口提交失败后自动重试

Do not use this skill for Facebook, Google, non-TikTok channels, pure data report generation, anomaly inspection, or generic TikTok knowledge Q&A.

## Hermes output protocol
When Hermes needs to act, output valid JSON only. Do not wrap it in Markdown.

Required top-level fields:
```json
{
  "reply_type": "workflow_summary|ad_creation_check|ad_creation_prefill|confirmation_required|creation_submitted|status_check|retry_result|creation_success|creation_failed|monitoring_consent_request|monitoring_open_requested|blocked|clarification_required",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "短结论，不能编造结果",
  "slot_updates": {},
  "missing_slots": [],
  "ui_actions": [],
  "task_payload": {},
  "tool_requests": [],
  "risk_warnings": [],
  "blocked_reason": null,
  "unknowns": [],
  "next_step": {}
}
```

Canonical page action:
```json
{
  "action": "open_url",
  "url": "https://tiktok.vidau.ai/campaigns/ads",
  "page_key": "vidau_tiktok_ads",
  "target": "right_panel_or_current_tab",
  "reason": "打开 Vidau TikTok 广告创建页面"
}
```

If Hermes supports route navigation instead of raw URL, also include:
```json
{
  "action": "navigate",
  "page_key": "vidau_tiktok_ads",
  "route": "/campaigns/ads"
}
```

## Required slots before task generation
Collect or obtain these fields. If any required field is unknown, return `clarification_required` or mark it in `missing_slots`; do not silently assume.

### Account prerequisites
- `advertiser_id`: TikTok 广告账户 ID
- `advertiser_name`: TikTok 广告账户名称
- `account_source`: must indicate this account is opened by the Vidau/self-built platform
- `authorization_status`: must be authorized
- `account_status`: must be normal/active if provided by backend
- `creation_permission`: must be true or tool-confirmed
- `balance_or_credit_available`: available if backend provides this check

Blocking rule: if the account is not opened by the Vidau/self-built platform, block ad creation even if it is authorized. It may support data viewing/reporting/monitoring only.

### Campaign-level slots
- `objective`: promotion objective
- `campaign_name`
- `budget_mode`: campaign budget optimization or ad group budget
- `campaign_budget`: required when campaign budget optimization is enabled
- `delivery_country_or_region`

### Ad-group-level slots
- `adgroup_name`
- `schedule`: start/end time or continuous delivery setting
- `targeting`: region/country at minimum; age/gender/interests/custom audience if required by UI
- `placement`
- `optimization_goal`
- `billing_event` if required by platform UI
- `bid_strategy`
- `bid_value`: required only for bid strategies that need a bid/cost cap/ROAS target
- `adgroup_budget`: required when budget is controlled at ad group level
- `promotion_object`: landing page, app, TikTok Shop/store/product/live room, or other object depending on objective
- `pixel_or_event`: required for conversion objectives when the UI requires tracking/event selection

### Ad-level slots
- `identity`: TikTok identity / display identity / ad posting identity when required
- `creative_assets`: at least one usable creative asset
- `creative_count`
- `ad_count`
- `ad_text_or_caption`
- `call_to_action` if required by objective/UI
- `landing_page_url` or app/product binding when required by objective

## Field validation behavior
1. Show account/ad/creative summary when available.
2. Check account prerequisites first. If blocked, stop.
3. Check required fields by hierarchy: campaign -> ad group -> ad.
4. Return all missing fields at once when possible.
5. If all required fields pass, create a `task_payload` for Hermes/UI prefill.
6. Do not submit or publish unless the user explicitly confirms and the approved tool exists.

## Confirmation and risk rules
The following are high risk and must require explicit confirmation:
- Final submit/publish
- Budget modification
- Bid modification
- Pause/resume/delete ad/campaign/ad group
- External notification to customer
- Upload material to TikTok if not already in the platform material library

Opening the Vidau ad creation page and pre-filling fields is not high risk.

## Failure retry policy
Retry only when failure type is one of:
- network timeout
- gateway timeout
- temporary backend error
- transient interface submission failure
- idempotent task submission failure where backend explicitly says retryable

Do not retry when failure type is:
- missing required field
- unauthorized account
- account not opened by Vidau/self-built platform
- invalid objective/budget/bid/event/material
- insufficient balance/credit
- TikTok policy/audit rejection
- creative unavailable or invalid
- permission/scope missing
- business rule validation failed

Retry limit: 3 total attempts. After 3 failures, return final failed status and the backend failure reason. If no reason is returned, say the platform did not return a clear failure reason.

## Status mapping
Use backend/tool status directly when available. Map only for user display:
- `pending` / `processing` -> 创建中
- `review` / `under_review` -> 审核中
- `success` -> 成功
- `failed` -> 失败
- `partial_success` -> 部分成功
- missing/unrecognized -> unknown


## Standard workflow summary JSON
Use this when the user asks to understand the ad creation workflow or when Hermes needs to summarize the current task before opening/prefilling the page:
```json
{
  "reply_type": "workflow_summary",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "TikTok 广告创建将按 account -> campaign -> ad group -> ad -> creative 层级执行，并在创建后监测状态直到成功或失败。",
  "delivery_hierarchy": ["account", "campaign", "adgroup", "ad", "creative"],
  "account_summary": {
    "advertiser_name": "{{advertiser_name}}",
    "advertiser_id": "{{advertiser_id}}",
    "ad_count": "{{ad_count}}",
    "creative_count": "{{creative_count}}",
    "status": "{{status}}"
  },
  "ui_actions": [],
  "task_payload": {},
  "tool_requests": [],
  "risk_warnings": ["提交、发布、预算、出价、删除类动作必须用户明确确认。"],
  "next_step": {
    "label": "检查字段并打开创建页面",
    "requires_confirmation": false
  }
}
```

## Standard creation success JSON
```json
{
  "reply_type": "creation_success",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "已完成投放创建。投放账号：{{advertiser_name}}，账户 ID：{{advertiser_id}}，已成功投放 {{successful_ad_count}} 条广告。",
  "slot_updates": {
    "advertiser_name": "{{advertiser_name}}",
    "advertiser_id": "{{advertiser_id}}",
    "successful_ad_count": "{{successful_ad_count}}",
    "creative_count": "{{creative_count}}",
    "status": "success"
  },
  "missing_slots": [],
  "ui_actions": [],
  "task_payload": {},
  "tool_requests": [],
  "risk_warnings": [],
  "blocked_reason": null,
  "unknowns": [],
  "next_step": {
    "label": "是否需要开启监测？如果需要，我将为你打开预警中心。",
    "requires_confirmation": true
  }
}
```

## Standard final failure JSON after 3 retries
```json
{
  "reply_type": "creation_failed",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "广告创建失败，已自动重试 3 次，仍未成功。失败原因：{{failure_reason}}。",
  "slot_updates": {
    "advertiser_name": "{{advertiser_name}}",
    "advertiser_id": "{{advertiser_id}}",
    "status": "failed",
    "retry_attempt": 3,
    "max_retry_attempts": 3
  },
  "missing_slots": [],
  "ui_actions": [],
  "task_payload": {},
  "tool_requests": [],
  "risk_warnings": ["已达到最大重试次数，停止自动重试。"],
  "blocked_reason": "creation_failed_after_3_retries",
  "unknowns": [],
  "next_step": {
    "label": "查看失败原因并修改配置后重新创建",
    "requires_confirmation": false
  }
}
```

## Standard successful prefill JSON
```json
{
  "reply_type": "ad_creation_prefill",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "字段检查已通过，可打开 Vidau TikTok 广告创建页进行预填和确认。",
  "slot_updates": {
    "advertiser_id": "{{advertiser_id}}",
    "advertiser_name": "{{advertiser_name}}",
    "ad_count": "{{ad_count}}",
    "creative_count": "{{creative_count}}"
  },
  "missing_slots": [],
  "ui_actions": [
    {
      "action": "open_url",
      "url": "https://tiktok.vidau.ai/campaigns/ads",
      "page_key": "vidau_tiktok_ads",
      "target": "right_panel_or_current_tab",
      "reason": "打开 Vidau TikTok 广告创建页面"
    }
  ],
  "task_payload": {
    "platform": "tiktok",
    "source": "hermes",
    "mode": "prefill_only",
    "account": {},
    "campaign": {},
    "adgroup": {},
    "ads": [],
    "submit_requires_user_confirm": true
  },
  "tool_requests": [],
  "risk_warnings": ["最终提交或发布广告前必须由用户确认。"],
  "blocked_reason": null,
  "unknowns": [],
  "next_step": {
    "label": "打开创建页并预填",
    "requires_confirmation": false
  }
}
```

## Blocked example for external account
```json
{
  "reply_type": "blocked",
  "module": "tiktok_hermes_ad_creation",
  "natural_language_summary": "该账户不是 Vidau 平台开通账户，不能在本平台创建或发布 TikTok 广告。",
  "slot_updates": {},
  "missing_slots": [],
  "ui_actions": [],
  "task_payload": {},
  "tool_requests": [],
  "risk_warnings": [],
  "blocked_reason": "non_vidau_opened_account",
  "unknowns": [],
  "next_step": {
    "label": "可切换到数据查看、报表或异常监控",
    "requires_confirmation": false
  }
}
```

## Common Pitfalls

1. **Premature completion** — do not claim an ad was created or delivered unless a backend tool result or explicit page state confirms it. The JSON output templates with `{{placeholders}}` are instructions, not confirmation of success.

2. **UI click loops** — never click the same button or semantic UI target more than 2 times in one workflow. If the page state does not change after a click, read state again and adapt; do not repeat the click.

3. **Skipping account source check** — always check that the account is opened by the Vidau/self-built platform before allowing ad creation. Non-Vidau accounts may only support data viewing and reporting, not creation.

4. **Retrying non-retryable failures** — do not retry business-rule failures (missing fields, unauthorized account, insufficient balance, policy/audit rejection, invalid material, permission errors). Only retry network timeouts, gateway timeouts, transient backend errors, or backend-explicitly-retryable failures.

5. **Opening warning center without consent** — do not open the `预警中心` panel until the user has explicitly confirmed they want monitoring. The success notification must ask first.

6. **Silently assuming slot values** — if any required field (advertiser ID, campaign name, budget, objective, etc.) is unknown, mark it in `missing_slots` and return `clarification_required`. Never silently assume a value.

7. **Claiming status without backend confirmation** — after a creation task is submitted, poll the backend/page status until it reaches a terminal state. Do not assume success based on a single tool return like `Clicked` or `Submitted`.

## Verification Checklist

- [ ] Frontmatter: `name` ≤ 64 chars, lowercase + hyphens; `description` ≤ 1024 chars
- [ ] Account prerequisites checked first (source, authorization, status, balance)
- [ ] Campaign → ad group → ad field hierarchy validated before prefill
- [ ] All missing fields reported at once (not one at a time)
- [ ] Final submit/publish requires explicit user confirmation
- [ ] Budget changes, bid changes, pause/resume/delete require explicit confirmation
- [ ] After submission: status polled until terminal (success/failed/partial_success/unknown)
- [ ] Retries limited to 3 total attempts, retryable failures only
- [ ] Success notification includes: advertiser name, advertiser ID, successful ad count, creative count
- [ ] Warning center opened only after user consent
- [ ] UI clicks ≤ 2 per semantic target, state read between clicks
- [ ] `agents/openai.yaml` removed (not a Hermes artifact)
