# Source and validation notes

This skill is normalized from the user-provided TikTok Agent knowledge base, Hermes page usage scenario, and Vidau TikTok ad monitoring/report/ad-creation requirements in this conversation.

Primary runtime scenario:
- User uploads this skill in Hermes.
- User opens `https://tiktok.vidau.ai/`.
- The site defaults to the AI助手 page.
- User enters questions in the AI助手 panel.
- Hermes invokes this skill and displays the returned user-facing content in the AI助手 panel.

Strict validation principles:
- Return valid JSON only for Hermes AI助手 panel interactions.
- Include both routing fields and `assistant_display` so Hermes can parse the response while the user sees clean content.
- Do not fabricate business results or performance metrics.
- Preserve unknown/blocked states when tool permissions, account source, authorization, sync state, backend configuration, BC qualification, or scope are not known.
- Treat high-risk actions as confirmation-required.
- Use `https://tiktok.vidau.ai/` as the AI助手 entry page.
- Use `https://tiktok.vidau.ai/campaigns/ads` as the TikTok ad creation/ad list page entry point.
- Do not claim page navigation, data sync, ad creation, ad publishing, or backend action succeeded without a tool/backend result.

Recommended Hermes rendering behavior:
1. Parse the full JSON.
2. Render `assistant_display.markdown` in the AI助手 panel.
3. Render `assistant_display.cards`, `charts`, and `action_buttons` if supported.
4. Use `ui_actions` only for frontend actions such as opening a page or panel.
5. Use `tool_requests` only to trigger backend tools after permission and confirmation checks.
6. Never display raw JSON to ordinary users unless debug mode is enabled.
