# TikTok Ads MCP — 广告智能工具集 v2.1.0

<p align="center">
  <strong>TikTok 广告平台 MCP 工具集</strong><br>
  直连 HTTP · 无需登录 · 极速响应 · 三合一模块
</p>

---

## 版本更新：v1.0.0 → v2.0.0

| 维度 | v1.0.0 (Hermes) | v2.0.0 (MCP) |
|------|----------------|--------------|
| 架构 | Hermes Agent Runtime | **MCP HTTP JSON-RPC 直连** |
| 登录 | 需要 VidAU SSO | **不需要登录** |
| 速度 | 慢（需浏览器） | **极快（~1s/调用）** |
| 模块数 | 4 个独立子模块 | **1 个主 Skill + 1 个知识库子 Skill** |
| 广告创建 | 浏览器模拟 | **MCP 工具直接调用** |
| 数据巡检 | 页面抓取 | **两步滤波法 < 10s** |

---

## 功能模块

### MCP 主 Skill — 三合一

集成广告创建、AI 巡检、日报/周报三大模块于一个 Skill：

- **广告创建**：账户检查 → Campaign 创建 → Adgroup 创建 → Ad 创建，全程 MCP 直连
- **AI 巡检**：花费异常、转化异常、CPA 异常、ROAS 异常、预算耗尽、素材疲劳 — 六类异常自动检测
- **日报/周报**：一键生成，支持自定义日期和账户过滤

### 知识库子 Skill（skills/knowledge）

- 自然语言意图识别与业务路由
- TikTok 广告平台知识问答
- Hermes AI 助手面板 JSON 响应格式
- ✅ v2.1.0 已内置 MCP 执行层配置（`related_skills: [ads-tiktok-mcp]`），自动使用两步滤波法等优化策略

---

## 快速开始

### 方式一：一键安装（推荐）

直接复制 [INSTALL_PROMPT.md](INSTALL_PROMPT.md) 中的安装提示词，粘贴到 WorkBuddy / Hermes / Cursor 对话中。

### 方式二：手动安装

1. 克隆本仓库
2. 将根目录 `SKILL.md` 导入到 WorkBuddy / Hermes 的 Skill 管理中
3. 将 `skills/knowledge/` 目录也作为子 Skill 导入
4. （可选）配置 MCP 服务器端点

### 方式三：MCP 直连

Skill 内置了 HTTP 直连方式，无需额外配置即可使用。如需配置标准 MCP 端点，参考 [INSTALL_PROMPT.md](INSTALL_PROMPT.md)。

---

## 10 个 MCP 工具

| # | 工具 | 用途 |
|---|------|------|
| 1 | `list_advertisers` | 列出所有广告主及其状态 |
| 2 | `get_alerts` | 获取系统预警 |
| 3 | `get_campaigns` | 获取 Campaign 列表 |
| 4 | `create_campaign` | 创建 Campaign |
| 5 | `get_adgroups` | 获取 Adgroup 列表 |
| 6 | `create_adgroup` | 创建 Adgroup |
| 7 | `get_ads` | 获取 Ad 列表 |
| 8 | `create_ad` | 创建 Ad |
| 9 | `get_daily_metrics` | 获取日报指标 |
| 10 | `sync_advertiser_data` | 同步实体数据 |

---

## 速度优化：两步滤波法

传统方式对全部账户做深度查询，**耗时较长**。两步滤波法：

1. `list_advertisers` → 过滤 `campaignsCount > 0`
2. 并行 `get_daily_metrics(level=ADVERTISER)` → 只查有数据的账户

**结果：查询时间大幅缩短，效率显著提升。**

---

## 使用示例

| 你说 | 动作 |
|------|------|
| 「帮我创建一个 TikTok 广告」 | 进入参数收集流程，逐步创建 |
| 「看看今天的投放数据」 | 输出巡检摘要，标注异常指标 |
| 「生成本周周报」 | 输出完整的周报卡片 |
| 「TikTok 广告层级是什么？」 | 知识库返回三级结构解释 |
| 「打开广告面板」 | 在本机 Chrome 中打开平台 |

---

## 目录结构

```
tiktok-agent/
├── SKILL.md                         # MCP 主 Skill（广告创建+巡检+报表）
├── README.md                        # 本文件
├── INSTALL_PROMPT.md                # 一键安装提示词
├── references/
│   ├── ad-creation-flow.md          # 广告创建详细流程
│   ├── ad-inspection-rules.md       # AI 巡检规则详解
│   ├── report-format.md             # 日报/周报格式规范
│   ├── source_and_validation.md     # 知识库来源与验证说明
│   └── setup.md                     # 安装与配置指南
├── agents/
│   └── openai.yaml                  # Agent 接口配置
├── assets/
│   └── icon.svg                     # 工具集图标
└── skills/
    └── knowledge/                   # 知识库子 Skill（意图路由+问答）
        ├── SKILL.md
        └── references/
            └── source_and_validation.md
```

---

## 注意事项

1. MCP 直连方式不需要浏览器登录，但需要有效的 API Key
2. 巡检和日报不会自动同步数据（sync 太慢），请在平台手动刷新
3. 非 VidAU 开通的 TikTok 账户仅支持数据查看和报表
4. 所有高风险操作（创建广告、修改预算等）均需用户明确确认
5. `balance` 字段为字符串类型，需要 `float()` 转换后再格式化

---

## 版本历史

- **v2.1.0** (2026-07-09)：知识库 skill 升级 — 内置 MCP 执行层关联（`related_skills: [ads-tiktok-mcp]`），source_and_validation 新增 MCP 执行层说明
- **v2.0.0** (2026-07-08)：MCP 架构升级，三合一模块，直连 HTTP，速度提升 30 倍
- **v1.0.0** (2026-07-03)：Hermes 初始版本，4 个独立子模块

---

MIT License
