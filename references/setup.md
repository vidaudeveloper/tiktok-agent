# TikTok Ads MCP — 安装与配置指南 v2.0.0

## 架构对比：v1.0.0 vs v2.0.0

| 维度 | v1.0.0 | v2.0.0 |
|------|--------|--------|
| 运行方式 | Hermes Agent + 浏览器模拟 | **MCP HTTP JSON-RPC 直连** |
| 需要登录 | 是（VidAU SSO） | **否（API Key 认证）** |
| 子模块数 | 4 个独立 Skill | **1 个主 Skill + 1 个知识库 Skill** |
| 响应速度 | 慢 | **极快（~1s/调用）** |

---

## 安装前准备

### 环境要求

1. AI Agent 运行环境（WorkBuddy / Hermes / Cursor / Claude Desktop 等）
2. 支持 Skill 上传和管理的平台
3. 有效的 API Key（已内置在 Skill 文件中）
4. TikTok 广告账户（已在 VidAU 平台授权）

### 权限检查清单

- [ ] VidAU 平台已授权的 TikTok 广告账户
- [ ] 账户余额充足（或至少有一个余额 > 0 的账户）
- [ ] AI Agent 环境支持 skill 导入

---

## 安装步骤

### 方式一：一键安装（推荐）

1. 打开 [INSTALL_PROMPT.md](../INSTALL_PROMPT.md)
2. 将完整提示词复制粘贴到 AI Agent 对话中
3. AI Agent 将自动完成 skill 安装、MCP 配置和功能验证

### 方式二：手动安装

#### Step 1: 安装主 Skill

1. 打开 AI Agent 的 Skill 管理面板
2. 导入仓库根目录的 `SKILL.md` 文件
3. 确认 `ads-tiktok-mcp` skill 已被识别并激活

#### Step 2: 安装知识库子 Skill

1. 进入 `skills/knowledge/` 目录
2. 将该目录（或目录中的 `SKILL.md`）导入到 Skill 管理中
3. 确认 `tiktok-hermes-agent-knowledge` skill 已被识别

#### Step 3: 验证安装

在对话中依次测试以下命令：

```
# 1. 知识问答
TikTok 广告层级是什么？

# 2. 数据巡检
查看投放数据

# 3. 广告创建（不会真的创建，会提示缺失字段）
帮我创建一条 TikTok 广告

# 4. 日报生成
生成 TikTok 日报
```

---

## MCP 配置说明

### Skill 内置直连（默认，无需额外配置）

主 Skill 内置了 HTTP 直连方式，使用以下配置：

```python
API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
TOOLS_URL = "https://tiktok.vidau.ai/api/mcp/tools"
```

无需额外配置即可使用。

### 标准 MCP 端点配置（可选）

如需在 WorkBuddy 的 MCP 配置中使用，在 `~/.workbuddy/mcp.json` 中添加：

```json
{
  "mcpServers": {
    "tiktok-ads-agent": {
      "url": "https://tiktok.vidau.ai/api/mcp/sse?apiKey=tk_1bfa861961110ed257b517680da9efexxxxxxxxxxxxxxx"
    }
  }
}
```

---

## 默认配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| 平台 URL | `https://tiktok.vidau.ai/` | VidAU 主平台 |
| MCP 端点 | `/api/mcp/sse` | SSE 端点（带 apiKey 参数） |
| 请求超时 | 15 秒 | 单次 MCP 请求 |
| 并行超时 | 30 秒 | as_completed timeout |
| 并行线程数 | 5 | ThreadPoolExecutor |
| 巡检方法 | 两步滤波法 | 先过滤空账户再查数据 |

---

## 可选配置项

### 自定义巡检阈值

在 `references/ad-inspection-rules.md` 中可调整各类异常的触发阈值：

- CPA 异常：目标 CPA × 1.3 → 可调整为 × 1.2 或 × 1.5
- ROAS 异常：目标 ROAS × 0.7 → 可调整为 × 0.6 或 × 0.8
- 预算耗尽：花费/预算 ≥ 70% → 可调整为 60% 或 80%
- 素材疲劳：CTR < 历史 × 0.7 → 可调整为 × 0.6 或 × 0.8

### 自定义报告格式

在 `references/report-format.md` 中可调整日报/周报的输出格式和内容。

---

## 常见问题

### Q: 提示 API Key 无效？
A: 联系管理员更新 SKILL.md 中的 `API_KEY` 变量值。

### Q: 巡检结果为空？
A:
1. 确认至少有一个 `campaignsCount > 0` 的已授权账户
2. 在平台手动执行一次数据同步
3. 检查账户余额是否 > 0

### Q: 创建广告一直提示缺少字段？
A: 这是安全机制。请按提示依次补充：账户信息、推广目标、优化目标、素材、定向、预算、出价。

### Q: `sync_advertiser_data` 超时怎么办？
A: 这是已知问题（大账户同步 60s+）。建议在平台手动刷新，巡检和日报场景不依赖此接口。

### Q: 如何切换回浏览器模式？
A: 回复"打开广告面板"，AI 会在本机启动 Chrome 打开 VidAU 平台。

---

## 升级指南

从 v1.0.0 升级到 v2.0.0：

1. 备份旧的 skill 配置
2. 删除旧的 4 个子 skill（creation / inspection / knowledge / report）
3. 安装新的 `ads-tiktok-mcp` 主 Skill 和 `tiktok-hermes-agent-knowledge` 知识库 Skill
4. 验证功能正常

---

*最后更新: 2026-07-08 · v2.0.0*
