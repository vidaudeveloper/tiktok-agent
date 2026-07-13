# Source and validation notes

This skill is normalized from the user-provided TikTok Agent knowledge base, AI助手 page usage scenario, and Vidau TikTok ad monitoring/report/ad-creation requirements in this conversation.

Primary runtime scenario:
- User uploads this skill in the platform's management page.
- User opens `https://tiktok.vidau.ai/`.
- The site defaults to the AI助手 page.
- User enters questions in the AI助手 panel.
- The platform invokes this skill and displays the returned user-facing content in the AI助手 panel.

Strict validation principles:
- Return valid JSON only for AI助手 panel interactions.
- Include both routing fields and `assistant_display` so the platform can parse the response while the user sees clean content.
- Do not fabricate business results or performance metrics.
- Preserve unknown/blocked states when tool permissions, account source, authorization, sync state, backend configuration, BC qualification, or scope are not known.
- Treat high-risk actions as confirmation-required.
- Use `https://tiktok.vidau.ai/` as the AI助手 entry page.
- Use `https://tiktok.vidau.ai/campaigns/ads` as the TikTok ad creation/ad list page entry point.
- Do not claim page navigation, data sync, ad creation, ad publishing, or backend action succeeded without a tool/backend result.

Recommended platform rendering behavior:
1. Parse the full JSON.
2. Render `assistant_display.markdown` in the AI助手 panel.
3. Render `assistant_display.cards`, `charts`, and `action_buttons` if supported.
4. Use `ui_actions` only for frontend actions such as opening a page or panel.
5. Use `tool_requests` only to trigger backend tools after permission and confirmation checks.
6. Never display raw JSON to ordinary users unless debug mode is enabled.


## MCP 执行层（Vidau Agent 对话模式）

本 skill 在 Vidau Agent 对话中使用时（非 AI助手面板），必须配合 `ads-tiktok-mcp` skill 一起加载。

- `ads-tiktok-mcp` 提供 `list_advertisers`、`get_daily_metrics`、`create_campaign` 等 MCP 工具（具体数量与名称以该 skill 当前版本为准；v2.3.0 已知含上述工具，未暴露 `get_alerts` 时异常预警改用 `get_daily_metrics` + 本 skill 阈值规则）。
- 详细映射与枚举（VidAU 简化值）见 SKILL.md 的「执行上下文说明」章节。
- 关键规则（对齐 v2.3.0）：两步滤波法、`sync_advertiser_data` 仅指定账户且数据陈旧时谨慎调用（15s 超时、失败不阻塞）、报表先下钻 `level=ADGROUP/AD` 空则降级 `level=ADVERTISER` 并注明、日期范围以账户时区为准。
