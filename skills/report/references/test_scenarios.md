# TikTok Report Skill — Verified Test Scenarios

*Verified: 2026-07-01 | Platform: tiktok.vidau.ai | Skill version: 1.1.0*

## Account page edge cases (https://tiktok.vidau.ai/accounts)

Tested with real page data. Table columns: 账户名称, 状态, 实体数量 (C/G/A), 来源, 余额, 时区, 最后同步时间, 操作.

### Balance edge cases

| Scenario | Example Account | Balance Display | Correct Behavior |
|----------|----------------|-----------------|------------------|
| Positive balance | FLY-Vidau-evoloophome+001 | `USD 5,009.97` | Include in report |
| Zero/empty balance | FLY-Vidau-Chefwood-MY+1 | `"-"` | Exclude from report |
| Under-review account | 网络科技 | `"-"` (审核中 status) | Exclude — not enabled |
| Small positive | FLY-Vidau-宗帅1-宁跃 | `USD 143.27` | Include |
| Large balance | FLY-Vidau-RESSONIC DICRECT | `USD 4,306.01` | Include |

### Status filter options on page

- 全部状态 (default)
- 已启用
- 已暂停
- 审核中

### Source filter options

- 全部来源 (default)
- 123网络科技 (7625903947655479312)
- 元生-志炫 (7501630149742673921)

### Pagination

- Total: 27 accounts across 2 pages
- Page size options: 10, 20 (default), 50, 100
- Default page size shows 20 accounts per page

## Search flow validation

Search box label: `"搜索账户名或 ID"`
Buttons on page:
- `"拉取最新授权"` — refresh TikTok authorization
- `"同步实体数据"` — sync campaigns/adgroups/ads/creatives

### Search miss recovery sequence

1. Type account name/ID in search box
2. If not found → click `"拉取最新授权"`
3. Wait for refresh
4. Click `"同步实体数据"`
5. Re-search
6. If still not found → return `clarification_required` or `blocked` with `account_not_found` reason

## AI assistant page structure (https://tiktok.vidau.ai/)

- Heading: "TikTok 智能投放助手"
- Quick-action buttons: 查看投放数据, 查看实时数据, 创建广告, 生成广告文案, 查看预警信息, 了解开户流程, AI 创作视频
- Search box: "搜索广告户名称或ID..."
- Chat input: "输入你的问题或指令..."
- Sidebar navigation: AI 助手, 数据看板, 报表分析, 素材管理, 广告管理, 预警中心, 广告账户, 授权管理

## Browser tool considerations

- **Hermes headless browser** (browser_navigate): Not logged into Vidau → redirects to /login. Use BrowserMCP or computer_use instead for authenticated pages.
- **BrowserMCP** (mcp_browsermcp_*): Good for navigation and snapshot (read-only). Tab ID expires after certain operations. Re-navigate to recover. type() and click() on React SPA elements can be unreliable.
- **computer_use**: Drives real desktop Chrome. Best for visual confirmation. Requires Chrome window to be on-screen.

## Date default rules verified

| Report Type | Default Date Range | Comparison |
|-------------|--------------------|------------|
| 日报 (daily) | Current date (not yesterday) | Previous day preferred |
| 周报 (weekly) | Most recent 7 days | Previous 7 days preferred |

## End-to-end report generation test protocol (v2.1)

Each report generation flow must be verified with this exact sequence:

### E2E Step 1: Navigate to AI助手 page
- URL: `https://tiktok.vidau.ai/`
- Expected: Page loads with heading "TikTok 智能投放助手"
- If redirected to `/login` → return `login_required` and stop

### E2E Step 2: Locate chat input box
- Expected element: textbox "输入你的问题或指令..."
- If not found → return `ui_input_unavailable` and stop

### E2E Step 3: Type and submit report request
- For daily: type `"生成 TikTok 日报"` → submit
- For weekly: type `"生成 TikTok 周报"` → submit
- For account-specific: type `"生成账户 XXX 的 TikTok 日报"` → submit
- Verify submission was accepted (input clears or loading indicator appears)

### E2E Step 4: Wait for AI助手 response
- Poll the conversation area for the latest assistant message
- Maximum wait per attempt: 60 seconds
- If no response content appears → record this display attempt as failed
- Retry submit/wait/read/verify until a valid response appears or 10 total attempts have been made

### E2E Step 5: Read and verify report content
- Check that the conversation area contains expected report sections:
  - Daily: 数据范围, 核心数据 (spend, CPA, CTR, CVR, ROAS...), 环比变化, etc.
  - Weekly: 数据范围, 周核心数据, 周同比变化, etc.
- If content structure is missing or empty → record this display attempt as failed
- If content is present and valid → confirm report_generation = completed
- If 10 attempts all fail but the report content can be generated from verified data → return `current_chat_fallback` in the current conversation

### E2E Step 6: Failure escalation chain
```
page_not_loaded → login_required
page_loaded_but_no_input → ui_input_unavailable
submitted_but_no_response_before_10_attempts → report_render_failed
response_received_but_empty_before_10_attempts → report_render_failed
10_ai_assistant_display_attempts_failed_with_verified_report_data → current_chat_fallback
response_valid → report_generation_completed
```

### Test cases for E2E validation

| # | Test Case | Expected Result |
|---|-----------|----------------|
| 1 | Navigate to AI助手 while logged out | `login_required` |
| 2 | Navigate to AI助手 while logged in, submit "生成 TikTok 日报" | Daily report with current date |
| 3 | Navigate to AI助手, submit "生成 TikTok 周报" | Weekly report with last 7 days |
| 4 | Submit "生成账户 FLY-Vidau-菲律宾GMAX 的 TikTok 日报" | Daily report for GMAX only |
| 5 | Submit report request with no eligible accounts | `blocked` or `insufficient_data` |
| 6 | Open AI助手 page without submitting any request | `report_generation_not_completed` |
| 7 | Input box missing from page | `ui_input_unavailable` |
| 8 | Submit request but conversation area never updates before 10 attempts | `report_render_failed` |
| 9 | AI助手 display fails for 10 verified attempts while report data is otherwise available | `current_chat_fallback`, report content returned in current conversation |
