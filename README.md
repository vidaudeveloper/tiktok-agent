# Vidau TikTok Skills Bundle（配套 skill 包）

本包包含 **两个相互独立、配套使用** 的 TikTok 广告 skill。两者**不合并、各自触发**，
仅通过显式交叉引用保持口径一致。打包成 bundle 是为了**一次性部署交付**，不影响运行时行为。

---

## 包含的两个 skill

| 目录 | 名称 | 角色 | 版本 | 触发时机 |
|------|------|------|------|----------|
| `vidau-tiktok-agent-knowledge/` | Vidau TikTok AI 助手（知识/路由/响应格式化） | **大脑层**：意图识别、槽位提取、页面路由、响应 JSON 格式化、UI payload 渲染 | v1.1.0 | 用户在 AI 助手面板 / Agent 对话中询问 TikTok 相关（开户、授权、报表、巡检、知识、创编） |
| `ads-tiktok-mcp/` | Vidau TikTok Ads MCP 封装（执行层） | **执行层**：调用 VidAU MCP（`tiktok.vidau.ai`）真实接口，含健壮 `mcp()`、重试、枚举、报表、巡检、创编流程 | v2.3.0 | 路由判定需要真实数据 / 真实写操作时 |

---

## 分工与数据流

```
用户自然语言
     │
     ▼
[vidau-tiktok-agent-knowledge]  ← 大脑层（先触发）
  · 意图识别 / 槽位提取 / 缺失判断
  · 页面路由 / 预填 / 确认检测
  · 仅当确需真实数据或写操作时 ──► 调用执行层
     │ （按需 read ads-tiktok-mcp 的 reference，不预载）
     ▼
[ads-tiktok-mcp]  ← 执行层
  · 健壮 mcp() 调用 VidAU MCP
  · 枚举校验 / 报表 / 巡检 / 创编
     │
     ▼
返回结构化结果 ──► 大脑层格式化为 AI 助手面板 JSON（assistant_display / cards / charts / action_buttons）
```

- **大脑层不预载执行层重型文件**：仅在路由判定需要真实数据时，才读取 `ads-tiktok-mcp/references/mcp-client.md` 与 `enums-reference.md`，避免无谓上下文消耗。
- **执行层不感知前端**：只负责正确调用 MCP 并返回数据，不直接生成面向用户的展示文案。

---

## 对齐基线（两 skill 已对齐，请勿各自漂移）

- **VidAU 简化枚举**（真实 MCP 接受值）：`objective_type`=`LEAD_GEN`/`WEB_CONVERSIONS`、`optimization_goal`=`VIDEO_PLAY_3S`、`billing_event`=`CPC`/`CPM`/`OCPM`、`promotion_type`=`WEBSITE`/`APP_ANDROID`/`APP_IOS`/`LEAD_GEN*`。完整常量与官方对照见 `ads-tiktok-mcp/references/enums-reference.md`。
- **报表 level 规则**：先下钻 `ADGROUP`/`AD` 识别问题/优秀广告；为空则降级 `ADVERTISER` 并注明层级，**不编造**。两 skill 口径一致。
- **sync 策略**：交互场景默认不调 `sync_advertiser_data`；仅指定具体账户且数据陈旧时谨慎调用（15s 超时、失败不阻塞）。两 skill 口径一致。
- **时区**：报表默认日期与异常同环比统一按**广告主时区**，非服务器时区。
- **状态值**：`status` 统一用 `ENABLE`/`DISABLE`，无 `ACTIVE`。
- **部署平台**：VidAU（`tiktok.vidau.ai`）。全文不含任何历史平台专有字眼。

---

## 部署说明

1. 将两个子目录作为**两个独立 skill** 分别安装/加载到平台。
2. 大脑层 `vidau-tiktok-agent-knowledge` 需配合执行层 `ads-tiktok-mcp` 一起加载才能跑通"创建/巡检/报表"等需要真实数据的链路。
3. `ads-tiktok-mcp` 已内置预配置 API Key（脱敏，末尾 15 位以 x 代替），无需设置环境变量；如需更换见 `INSTALL_PROMPT.md`。
4. 大脑层 `agents/openai.yaml` 为 Agent 配置型文件，按平台约定放置（若平台期望的文件名不同，仅改名即可，内容无需改动）。

---

## 文件清单

```
vidau-tiktok-skills-bundle/
├── README.md                                  ← 本文件
├── vidau-tiktok-agent-knowledge/             ← 大脑层 v1.1.0
│   ├── SKILL.md
│   ├── references/
│   │   └── source_and_validation.md
│   └── agents/
│       └── openai.yaml
└── ads-tiktok-mcp/                           ← 执行层 v2.3.0
    ├── SKILL.md
    └── references/
        ├── mcp-client.md          ★ 唯一正确的 MCP 调用实现
        ├── enums-reference.md     ★ VidAU 简化枚举 + 官方对照 + Python 常量
        ├── ad-creation-flow.md
        ├── ad-inspection-rules.md
        └── report-format.md
```

---

## Changelog

- **2026-07-10**：初版 bundle。合并交付 `vidau-tiktok-agent-knowledge v1.1.0`（去历史平台字眼、对齐执行层）与 `ads-tiktok-mcp v2.3.0`（校准真实 VidAU MCP 返回体、枚举切回 VidAU 简化值、带 vidau 标识）。两 skill 内容相对各自的独立 zip 版本**未做改动**，仅新增本 README 说明配套关系。
