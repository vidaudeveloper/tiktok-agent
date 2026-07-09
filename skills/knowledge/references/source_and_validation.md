# Source and validation notes

This skill is normalized from the user-provided TikTok Agent knowledge base, AI Agent page usage scenario, and Vidau TikTok ad monitoring/report/ad-creation requirements in this conversation.

Primary runtime scenario:
- User uploads this skill in AI Agent.
- User opens `https://tiktok.vidau.ai/`.
- The site defaults to the AI助手 page.
- User enters questions in the AI助手 panel.
- AI Agent invokes this skill and displays the returned user-facing content in the AI助手 panel.

Strict validation principles:
- Return valid JSON only for AI Agent AI助手 panel interactions.
- Include both routing fields and `assistant_display` so AI Agent can parse the response while the user sees clean content.
- Do not fabricate business results or performance metrics.
- Preserve unknown/blocked states when tool permissions, account source, authorization, sync state, backend configuration, BC qualification, or scope are not known.
- Treat high-risk actions as confirmation-required.
- Use `https://tiktok.vidau.ai/` as the AI助手 entry page.
- Use `https://tiktok.vidau.ai/campaigns/ads` as the TikTok ad creation/ad list page entry point.
- Do not claim page navigation, data sync, ad creation, ad publishing, or backend action succeeded without a tool/backend result.

Recommended AI Agent rendering behavior:
1. Parse the full JSON.
2. Render `assistant_display.markdown` in the AI助手 panel.
3. Render `assistant_display.cards`, `charts`, and `action_buttons` if supported.
4. Use `ui_actions` only for frontend actions such as opening a page or panel.
5. Use `tool_requests` only to trigger backend tools after permission and confirmation checks.
6. Never display raw JSON to ordinary users unless debug mode is enabled.


## MCP 执行层（Vidau Agent 对话模式）

本 skill 在 Vidau Agent 对话中使用时（非 AI Agent AI助手面板），必须配合 `ads-tiktok-mcp` skill 一起加载。

- `ads-tiktok-mcp` 提供 `list_advertisers`、`get_daily_metrics`、`create_campaign` 等 10 个 MCP 工具
- 详细映射见 SKILL.md 的「执行上下文说明」章节
- 关键规则：两步滤波法、不调 sync、超时 15s、不查 ADGROUP/AD 级别报告
