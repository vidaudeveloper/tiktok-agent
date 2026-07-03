# Source and validation notes

This skill is normalized from the user-provided TikTok Agent knowledge base and TikTok ad monitoring/report/ad-creation requirements in this conversation.

Strict validation principles:
- Do not fabricate business results or performance metrics.
- Preserve unknown/blocked states when tool permissions, account source, authorization, sync state, backend configuration, BC qualification, or scope are not known.
- Treat high-risk actions as confirmation-required.
- Use the Vidau self-built TikTok ad page URL `https://tiktok.vidau.ai/campaigns/ads` only as the UI entry point; do not claim the page or backend action succeeded without a tool result.


## Added closed-loop workflow requirements
- Show delivery hierarchy, account name, account ID, ad count, creative/material count, and status whenever backend/page state provides them.
- Creation status must be monitored after submission and must end in user-facing success/failure/partial-success messaging.
- Failed creation may be automatically retried up to 3 total attempts only when the backend/page failure is retryable.
- Do not retry business validation failures, missing fields, unauthorized accounts, non-Vidau opened accounts, policy/audit rejection, invalid material, insufficient balance, or permission failures.
- Successful creation must notify account name, account ID, successful ad count, and creative count, then ask whether to enable monitoring.
- Only after the user confirms monitoring should Hermes open the Vidau `预警中心` panel.
- UI clicks must be state-aware, max 2 per same button/semantic target, and must stop on unexpected page changes.
