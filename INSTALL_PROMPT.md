# TikTok Ads MCP — 安装提示词

将此提示词**完整复制粘贴**到 WorkBuddy / Hermes / Cursor 等 AI Agent 对话中，即可自动完成 skills 安装：

```
请帮我安装 TikTok Ads MCP 工具集，包含以下步骤：

## Step 1: 安装主 Skill（ads-tiktok-mcp）

从 GitHub 仓库读取主 skill 文件：
https://github.com/vidaudeveloper/tiktok-agent/blob/main/SKILL.md

这是一个基于 MCP (Model Context Protocol) 的 TikTok 广告管理 skill，支持：
- 广告创建（create_campaign / create_adgroup / create_ad）
- AI 广告巡检（花费、转化、CPA、ROAS、预算、素材疲劳六大异常检测）
- 日报/周报生成

该 skill 使用 HTTP JSON-RPC 直连 tiktok.vidau.ai 的 MCP 接口，无需浏览器登录。

## Step 2: 安装知识库子 Skill（tiktok-hermes-agent-knowledge）

从 GitHub 仓库读取知识库子 skill：
https://github.com/vidaudeveloper/tiktok-agent/tree/main/skills/knowledge

这个 skill 负责：
- TikTok 广告平台的自然语言意图识别与业务路由
- 10 大业务模块覆盖（开户、授权、同步、报表、素材、搭建、预警等）
- Hermes AI 助手面板的 JSON 响应格式
- 缺失槽位检测与用户引导
- ⚠️ 已内置 MCP 执行层配置（`related_skills: [ads-tiktok-mcp]`），自动关联两步滤波法等优化策略

## Step 3: 配置 MCP 服务

如需在 WorkBuddy 中配置 MCP 服务器（可选，skill 内置了直连 HTTP 方式）：

在 `~/.workbuddy/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "tiktok-ads-agent": {
      "url": "https://tiktok.vidau.ai/api/mcp/sse?apiKey=tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
    }
  }
}
```

## Step 4: 验证安装

安装完成后，依次执行以下验证：

1. 输入 "TikTok 广告层级是什么？" → 应返回知识库模块的正确层级结构说明
2. 输入 "查看投放数据" → 应返回当前账户的巡检摘要
3. 输入 "帮我创建一条 TikTok 广告" → 应进入参数收集流程，提示缺失必填项
4. 输入 "生成 TikTok 日报" → 应生成完整的日报卡片

## 注意事项

- 直连 MCP 方式不需要登录 VidAU，但需要有效的 API Key
- 如果对平台返回的数据有疑问，可以回复"打开广告面板"启动本地 Chrome 查看
- 巡检和日报不会自动同步数据（sync_advertiser_data 太慢），请在平台手动刷新
- 所有高风险操作（创建广告、修改预算等）均需你明确确认后才会执行
```

---

## 快速验证命令

安装后，在对话中输入以下任一命令测试：

| 测试命令 | 预期响应 |
|----------|----------|
| `查看投放数据` | 列出已授权账户，过滤有 Campaign 的账户，展示巡检摘要 |
| `生成今日日报` | 生成包含指标、环比变化、问题/优秀广告、优化建议的日报 |
| `帮我创建 TikTok 广告` | 进入参数收集流程，提示缺失的必填字段 |
| `TikTok 广告系列和广告组有什么区别` | 返回三级广告结构的清晰解释 |
| `打开广告面板` | 自动在本地 Chrome 中打开 tiktok.vidau.ai |

---

## 常见问题

### Q: 提示 "API Key 无效" 怎么办？
A: 请联系管理员获取最新的 API Key，并更新 Skill 文件中的 `API_KEY` 变量。

### Q: 巡检显示"数据不足"？
A: 先在平台手动执行一次数据同步（点击"同步实体数据"），然后重新巡检。

### Q: 创建广告时一直提示缺少字段？
A: 这是正常的安全机制。请按照提示逐步补充账户、推广目标、优化目标、素材、定向、预算、出价等信息。

### Q: 如何切换到浏览器模式？
A: 直连 MCP 是最快路径。如需文件上传/OAuth 等操作，回复"打开广告面板"即可使用浏览器。

---

*最后更新: 2026-07-09 · v2.1.0*
