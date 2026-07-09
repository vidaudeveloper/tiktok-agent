---
name: ads-tiktok-mcp
description: 直连 tiktok.vidau.ai MCP 接口 — 无需登录、最快路径。用 execute_code + requests 直接 POST JSON-RPC 到 /api/mcp/tools。支持广告巡检、预警、Campaign/Adgroup/Ad CRUD、日报指标。禁止浏览器/computer_use，这一层更底层更快。
version: 2.0.0
category: productivity
tags: [tiktok, ads, mcp, sse, fast]
---

# TikTok Ads MCP — 直连 HTTP（最快路径）

**无需登录、无需浏览器、无需 Python mcp 包。** 直接用 `execute_code` + `requests` POST JSON-RPC 到 MCP tools endpoint。

**应用场景**：广告巡检、查预警、看报表、列出 Campaign/Ad/Adgroup、同步数据、广告创建。

---

## 核心定义

```python
# 永远用这段代码，放入 execute_code
import requests, json

API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
TOOLS_URL = "https://tiktok.vidau.ai/api/mcp/tools"

def mcp(name, args=None):
    """调用 MCP 工具，返回解析后的 data"""
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {"name": name, "arguments": args or {}},
        "id": 1
    }
    r = requests.post(f"{TOOLS_URL}?apiKey={API_KEY}", json=payload,
                      headers={"Content-Type": "application/json"}, timeout=60)
    result = r.json()["result"]["content"][0]["text"]
    return json.loads(result)
```

**速度优势**：直接 POST 到 `/api/mcp/tools`，跳过 SSE 握手（SSE 有 Vercel cold start 延迟 ~30-40s）。

---

## ⚡ 速度优化实战经验（重要！）

**MCP 的最大坑不是慢，而是不会过滤空账户。**

### 核心原则：两步滤波法

```python
# ❌ 错误做法 — 对所有账户做深度查询，白等
for aid in all_ids:
    mcp("get_daily_metrics", {"advertiser_id": aid, "level": "ADGROUP"})  # 空！浪费时间

# ✅ 正确做法 — 先筛选出真正有数据的账户
advs = mcp("list_advertisers")
active_with_camps = [a for a in advs if a.get("campaignsCount", 0) > 0]

# 只对有 Campaign 的账户查今日数据（ADVERTISER 级别最快）
def has_today_data(adv):
    try:
        m = mcp("get_daily_metrics", {
            "advertiser_id": adv["advertiserId"],
            "start_date": "2026-07-08",
            "end_date": "2026-07-08",
            "level": "ADVERTISER"
        })
        return adv, len(m) > 0
    except:
        return adv, False

with ThreadPoolExecutor(max_workers=5) as ex:
    # ⚠️ 必须设 timeout！默认 as_completed 会无限等超时请求
    futures = {ex.submit(has_today_data, a): a for a in active_with_camps}
    for f in as_completed(futures, timeout=30):
        adv, ok = f.result()
        if ok:
            # 只有这个账户才做深度分析
            do_deep_inspection(adv)
```

### 关键教训（血泪史）

1. **`get_daily_metrics` 的 `level=ADGROUP` / `level=CAMPAIGN` 基本都返回空** — 平台只同步了 ADVERTISER 级别的报告数据。**永远先查 `level=ADVERTISER`**，只有它有数据。
2. **`sync_advertiser_data` 巨慢且容易超时（60s+）** — 大账户（Ufun 50+ Campaign）同步请求大概率卡死。**不要在交互式查询中调 sync！** 让用户在平台手动刷新。
3. **大多数账户里常常只有少量真正有今日数据** — 不要对所有账户发 `get_daily_metrics`，先用 `list_advertisers` 的 `campaignsCount > 0` 过滤，再对子集查 ADVERTISER 级别。
4. **`ThreadPoolExecutor.as_completed` 必须设 `timeout`** — 不加 timeout 的话，一个请求卡 60 秒你就得等 60 秒。
5. **Total time: 滤波法耗时显著低于全量扫描** — 速度大幅提升。

### MCP 请求超时设置

```python
# 单次请求：不要再设 60s，15s 足够
def mcp(name, args=None, timeout=15):
    ...
    r = requests.post(..., timeout=timeout)

# 并行：as_completed 加 timeout
for f in as_completed(futures, timeout=30):
    ...
```

---

## 10 个 MCP 工具

### 1. list_advertisers — 列出所有广告主
```python
advs = mcp("list_advertisers")
# 返回 [{id, advertiserId, advertiserName, status, balance, currency, timezone,
#          campaignsCount, adgroupsCount, adsCount, creativeCount,
#          lastEntitySyncAt, lastReportSyncAt, ...}]
```
- `balance` 可能为 `null` = 未同步（不是 $0）
- `status`: `STATUS_ENABLE` / `STATUS_LIMIT`（受限）
- **巡检起点**：先调这个，再按 `campaignsCount > 0` 筛选活跃账户

### 2. get_alerts — 获取系统预警
```python
alerts = mcp("get_alerts", {"advertiser_id": "7644525617435574290", "limit": 20})
# 返回 [{type, title, message, is_read, created_at, ...}]
```
- 必填: `advertiser_id`
- 可选: `is_read` (bool), `limit` (默认 20)

### 3. get_campaigns — 获取 Campaign 列表
```python
camps = mcp("get_campaigns", {
    "advertiser_id": "7644525617435574290",
    "status": "ACTIVE",     # 可选过滤: ACTIVE/DISABLE/DRAFT/FAILED/all
    "search": "关键词",      # 可选：名称模糊搜索
    "page": 1,              # 默认 1
    "pageSize": 50          # 默认 20
})
# 返回 {"data": [{id, tiktokCampaignId, campaignName, objectiveType,
#                  budgetMode, budget, status, isDraft, ...}],
#        "total": N, "page": 1, "pageSize": 50}
```
- `status` 值来自 TikTok API: `ENABLE`（运行中） / `DISABLE`（暂停） / `ACTIVE` / `DRAFT` / `FAILED`
- 传 `"all"` 不过滤

### 4. create_campaign — 创建 Campaign
```python
camp = mcp("create_campaign", {
    "advertiser_id": "7644525617435574290",
    "campaign_name": "我的新Campaign",
    "objective_type": "WEB_CONVERSIONS",  # 必填: REACH/VIDEO_VIEWS/LEAD_GEN/TRAFFIC/WEB_CONVERSIONS/APP_PROMOTION/PRODUCT_SALES
    "budget_mode": "BUDGET_MODE_INFINITE", # 可选: BUDGET_MODE_INFINITE/DAY/TOTAL
    "budget": 0,                           # 0=不限
    "budget_optimize_switch": False        # CBO 预算优化
})
```

### 5. get_adgroups — 获取 Adgroup 列表
```python
adgs = mcp("get_adgroups", {
    "advertiser_id": "7644525617435574290",
    "campaign_id": "本地的campaign UUID",   # 可选，不是 tiktokCampaignId
    "status": "ENABLE",
    "search": "名称搜索",
    "page": 1, "pageSize": 50
})
```

### 6. create_adgroup — 创建 Adgroup
```python
adg = mcp("create_adgroup", {
    "advertiser_id": "7644525617435574290",
    "campaign_id": "本地的campaign UUID",   # 必填
    "adgroup_name": "广告组名称",            # 必填
    "targeting": {                          # 必填
        "location_ids": ["100"],            # 必填: 国家/地区ID
        "gender": "NONE",                   # GENDER_FEMALE / GENDER_MALE / NONE
        "age_groups": [], "languages": []
    },
    "budget_mode": "BUDGET_MODE_DAY",       # DAY / TOTAL
    "budget": 50,
    "schedule_type": "SCHEDULE_FROM_NOW",   # FROM_NOW / START_END
    "optimization_goal": "CLICK",           # CLICK/CONVERT/REACH/VIDEO_PLAY_3S 等
    "billing_event": "CPC",                 # CPC/CPM/OCPM
    "bid_type": "BID_TYPE_NO_BID",
    "bid_price": 0,
    "pacing": "PACING_MODE_SMOOTH",         # SMOOTH / FAST
    "promotion_type": "WEBSITE",            # WEBSITE/APP_ANDROID/APP_IOS/LEAD_GEN_*
    "placement_type": "PLACEMENT_TYPE_AUTOMATIC",
    "placements": []
})
```
- **targeting.location_ids 必填**，如 `["100"]` = 美国

### 7. get_ads — 获取 Ad 列表
```python
ads = mcp("get_ads", {
    "advertiser_id": "7644525617435574290",
    "adgroup_id": "本地adgroup UUID",
    "status": "ENABLE",
    "search": "搜索词",
    "page": 1, "pageSize": 50
})
```

### 8. create_ad — 创建 Ad
```python
ad = mcp("create_ad", {
    "advertiser_id": "7644525617435574290",
    "adgroup_id": "本地adgroup UUID",       # 必填
    "ad_text": "广告文案",                   # 必填
    "ad_name": "可选名称",
    "creative_id": "素材库UUID",             # 自动上传到线上
    "video_id": "已上传的TikTok视频ID",
    "cover_url": "封面图URL",               # 自动以文件形式上传
    "call_to_action": "LEARN_MORE",          # 默认 LEARN_MORE / 传 DYNAMIC=动态优选
    "landing_page_url": "落地页URL"
})
```

### 9. get_daily_metrics — 日报指标
```python
metrics = mcp("get_daily_metrics", {
    "advertiser_id": "7644525617435574290",
    "start_date": "2026-07-01",
    "end_date": "2026-07-07",
    "level": "ADVERTISER"                   # ADVERTISER/CAMPAIGN/ADGROUP/AD
})
```
- 必填: `advertiser_id`, `start_date`, `end_date`
- `level` 可选，默认按广告主聚合

### 10. sync_advertiser_data — 同步实体数据
```python
mcp("sync_advertiser_data", {
    "advertiser_id": "7644525617435574290",
    "levels": ["ADVERTISER", "CAMPAIGN", "ADGROUP", "AD"]
})
```
- `levels`: 要同步的层级，全部同步用 `["ADVERTISER", "CAMPAIGN", "ADGROUP", "AD"]`

---

---

## 业务模块总览

基于 MCP 工具，实现三大业务模块。**触发关键词决定走哪个模块**；多个关键词同时命中时按"创建 → 巡检 → 报表"顺序处理。

### 触发关键词

| 模块 | 关键词 |
|------|--------|
| 广告创建 | 创建广告、建广告、投广告、投放广告、我要投放、帮我投放、创建TikTok广告、搭建广告、批量创建广告、批量投放、帮我把广告发布出去、推广 |
| AI 巡检 | 广告巡检、巡检、开启巡检、监控广告、AI巡检、开始巡检、确认、好的、需要、开始 |
| 日报/周报 | 日报、给我生成日报、给我生成周报、日报数据、昨天日报、今日数据报告、报告、周报数据、周报、本周周报、根据我的格式生成日报、根据我的格式生成周报 |

指定日期/账户格式：`x天日报`、`x日-x日的日报`、`广告账户名称 xxx 的日报/周报`、`广告系列名称 xxx 日报/周报`等。

---

## 巡检自动触发条件

除用户手动触发巡检关键词外，以下场景**自动发起巡检询问**（展示巡检价值和覆盖范围，引导用户确认开启）：

### 场景 1：广告创建完成后
广告创建成功后，自动询问：
> "广告已创建完成，是否需要帮您创建广告巡检呢？覆盖花费、转化、CPA、ROAS、预算、素材等方面进行AI监测，及时掌握广告数据趋势，可以回复「确认」，进行开启"

### 场景 2：预算花费超过 60%
检测到广告预算已花费超 60% 时，自动询问：
> "当前检测到您的广告预算已花费超 60%，是否需要开启 AI 广告巡检呢？覆盖花费、转化、CPA、ROAS、预算、素材等方面进行AI监测，及时掌握广告数据趋势，可以回复「确认」，进行开启"

### 场景 3：检测到有在投放的广告
检测到有在投放的广告账户/广告系列/广告组/广告时，自动询问：
> "检测到您有正在投放的广告，是否需要开启 AI 广告巡检呢？可以回复「确认」，进行开启"

### 场景 4：每日定时主动询问
- 每天早上 **9:45** 主动询问一次
- 每天下午 **15:00** 主动询问一次

用户回复"确认"、"好的"、"需要"、"开始"、"巡检"等关键词后，立即触发广告巡检。

---

## 模块一：广告创建

> 详细流程见 `references/ad-creation-flow.md`

### 投放前

触发创建关键词后，在 VIDAU AGENT 端口进入参数收集流程。投放前须完成检查：

1. 账户授权状态 — 是否已授权、有效
2. 账户余额 — 是否充足
3. 推广目标 — 是否已选择
4. 优化目标 — 是否已选择
5. 素材 — 是否已上传、审核通过
6. 定向 — 年龄、性别、地区、兴趣、行为、设备等
7. 预算 — 日预算/总预算
8. 出价 — 出价策略与出价金额

**不通过**：不允许进入创建提交，必须提示用户原因（如"账户未授权"、"余额不足"、"推广目标未选"等）。

### 创建流程

按面板依次收集：广告账户 → 推广目标 → 优化目标 → 素材 → 定向 → 预算 → 出价。

**预算检测**：当用户选择同一个广告账户、同一个推广目标、同一个优化目标时，填写预算须检测 7 日内预算的平均值。当前输入的预算超出或低于平均值的阈值范围时，提示建议预算范围。

所有高风险操作（创建、修改、发布）均须等待用户明确确认后方可执行。

### MCP 工具映射

| 步骤 | 工具 |
|------|------|
| 检查账户 | `list_advertisers` |
| 创建 Campaign | `create_campaign` |
| 创建 Adgroup | `create_adgroup` |
| 创建 Ad | `create_ad` |

### 创建完成

创建完成后，必须返回创建状态：

| 状态 | 处理 |
|------|------|
| 成功 | 返回广告 ID、广告组 ID、Campaign ID 等创建结果；触发 AI 巡检询问 |
| 失败 | 触发失败重试规则（最多 3 次）→ 仍失败返回失败原因 + 建议修改字段 |
| 进行中 | 明确提示"创建进行中，等待接口返回"，不默认认为成功 |

**成功后自动触发巡检询问**：广告巡检询问内容见上方"巡检自动触发条件 - 场景 1"。

### 输出格式要求

广告创建在不同执行阶段，应根据当前执行状态返回对应的用户可见信息：

- 输出内容不得编造任何广告创建结果、TikTok 返回数据或广告状态
- 若接口未返回结果，应明确提示当前状态，而非默认认为创建成功
- 所有涉及广告创建、修改、发布等高风险操作，均需等待用户明确确认后方可执行

### 异常处理

| 异常类型 | 处理规则 |
|----------|----------|
| 通用异常 | 优先返回明确原因，结合接口返回信息进行转换，不直接透传原始错误码 |
| MCP / TikTok 接口异常 | 自动重试最多 3 次 |
| 网络异常 | 超时重新请求一次；连续失败提示网络异常 |
| 用户取消 | "取消""不用了""退出" → 立即停止创建，返回"已取消广告创建"；若已成功提交 TikTok，返回"本次创建已成功提交，无法终止" |

---

## 模块二：AI 广告巡检

> 详细规则见 `references/ad-inspection-rules.md`

### 功能
提醒通知广告数据变化趋势，及时调整广告数据。

### 职责边界
巡检目标、异常指标、阈值规则、建议动作和多异常处理优先级。

包含：花费异常、转化异常、CPA 异常、ROAS 异常、预算耗尽、素材疲劳。

### 六类异常（按优先级排序）

| 优先级 | 异常类型 | 核心判断 |
|--------|----------|----------|
| 1 | **花费异常** | 花费过快/过慢/突然不花/暴涨 |
| 2 | **转化异常** | 转化下降/归零/有点击无转化 |
| 3 | **CPA 异常** | CPA > 目标×1.3、CPA > 历史×1.5、有花费无转化 |
| 4 | **ROAS 异常** | ROAS < 目标×0.7、ROAS < 历史×0.6、有消耗无收入 |
| 5 | **预算耗尽** | 日预算≥70%、日预算≥95%、总预算≥90%、归零停投 |
| 6 | **素材疲劳** | CTR < 历史×0.7、CVR < 历史×0.7、连续≥7天且CTR降≥30% |

### 巡检流程

发现异常 → 检测阈值规则 → 根因分析（排查 + 止损建议 + 优化建议）

当多个异常同时出现时，按优先级顺序展示对应的"排查"、"止损建议"、"优化建议"。

### 默认巡检内容

开启巡检后，默认展示：
- 数据异常指标
- 阈值规则
- 建议动作
- 多异常处理优先级
- 覆盖：花费异常、转化异常、CPA 异常、ROAS 异常、预算耗尽、素材疲劳

### MCP 执行（优化版）

**两步滤波法，全程 < 10s：**

1. `list_advertisers` → 过滤 `campaignsCount > 0` 的账户
2. 并行 `get_daily_metrics(level=ADVERTISER)` → 找出今日真有数据的账户（通常 27 个里只有 1-2 个）
3. **只对有数据的账户**拉 7 天历史做阈值对比分析

> ⚠️ 不要调 `sync_advertiser_data`（巨慢易超时）  
> ⚠️ 不要查 `level=ADGROUP/CAMPAIGN/AD`（基本都空）  
> ⚠️ `as_completed` 必须设 `timeout=30`  
> ⚠️ 单次 MCP timeout 设 15s 足够

---

## 模块三：日报/周报

> 详细格式见 `references/report-format.md`

### 功能
- 生成日报/周报，支持使用原有模版生成或自定义日报/周报格式
- 日报重点：今日表现、环比变化、问题广告、优秀广告、素材亮点、风险提醒、明日动作
- 周报重点：本周趋势、问题归因、优秀广告复盘、素材可复制方向、下周动作

### 职责边界
- 自动检测是否有正在投放的广告
- 自动检测是否有 7 天内花费大于 0 的广告账户/广告系列/广告组/广告
- 仅生成日报和周报

### 触发与自动询问

#### 未指定日期和广告账户时
- 触发日报关键词 → 自动回复："若没有指定日期和广告账户，将会为您生成今日日报"
- 触发周报关键词 → 自动回复："若没有指定日期和广告账户，将会为您生成本周周报（最多支持一个月内的周报数据）"

#### 指定日期和广告账户时
支持关键词格式：
- "x天日报"、"x日-x日的日报"
- "广告账户名称 xxx / 广告ID xxxx 的日报/周报"
- "广告系列名称 xxx / 广告系列 IDxxx 日报/周报"
- "广告名称 xxx / 广告IDxxx 的日报/周报"

根据指定的日期、对应的广告层级名称或输出的层级 ID 生成日报或周报。

### 账户过滤
仅针对余额 > 0 的已授权账户；`balance: null` 视为未同步，跳过。

### MCP 执行
1. `list_advertisers` 获取账户列表
2. 并行 `get_daily_metrics`（advertiser_id + start_date + end_date）
3. 按 Campaign/Adgroup 级别细分识别问题广告和优秀广告
4. 按 report-format.md 格式输出 Markdown

### 输出格式要求

#### 数据指标展示
展示数据指标、环比变化、数据亮点。
- 环比变化：🔴 红色箭头向上 = 上涨，🟢 绿色箭头向下 = 下降（广告数据惯例）

#### 问题广告列表
以列表形式展示：【日期、广告名称、广告ID、花费、点击率、转化率、曝光数、存在问题、优化建议】

#### 优秀广告列表
以列表形式展示：【广告名称、广告ID、花费、点击率、转化率、曝光数、建议】

#### 优化建议
分点列出

#### 明日关注/下周关注重点
列出需要关注的广告、指标、风险点

#### 末尾询问
询问需要帮用户执行哪些操作：
- 创建广告巡检
- 生成新的广告文案
- 创建新的素材
- 打开广告面板进行调整
- 再次同步数据

### 格式要求
- 必须直接整理好的日报或周报形式，不要前言后语
- 分段清晰，标题明确，格式干净
- 字段缺失时，不要编造，留空或标注"暂无数据"

### 默认值与数据范围
- 日报：默认当前日期
- 周报：默认 7 天内
- 展示数据日期范围在报表头部明确标注

---

## 输出渲染策略

所有模块输出遵循以下降级策略：

1. **支持 ui_payload 渲染** → 输出结构化 JSON + ui_payload
   - 巡检：`inspection_summary`（概览卡片）、`anomaly_overview`（异常统计）、`anomaly_cards`（异常详情）、`recommended_actions`（建议列表）、`daily_inspection_setup`（每日自动巡检确认卡片）
   - 日报/周报：数据指标卡片、环比变化、问题广告列表、优秀广告列表
2. **不支持 ui_payload 渲染** → 自动降级为 **Markdown 卡片格式**
3. **Markdown 也无法展示** → 仅输出简洁摘要，禁止在用户界面直接展示完整 raw JSON

---

## 定时任务支持

### 定时巡检
用户可说："每天x点开启巡检"、"定时巡检"、"每天上午/下午巡检"
- 使用 cronjob 创建定时任务，到点自动执行巡检并推送结果
- 支持"定时关闭巡检"："关闭定时巡检" → cronjob action='remove'

### 定时日报/周报
用户可说："每天x点给我生成日报"、"每周星期x生成周报"、"每周星期x下午n"
- 使用 cronjob 创建定时任务，到点自动生成报表并推送
- 支持自定义格式："根据我的格式生成日报" → 识别格式后保存为 cron 模板

---

## 报告生成成功判定

只有满足以下全部条件，才算报告生成成功：
1. 已识别用户关键词进行操作
2. 已读取 accounts 页面已授权账户
3. 已过滤余额 > 0 账户
4. 已获取日报/周报投放指标
5. 必须生成本地渲染或 Markdown 格式
6. 回复内容必须优先展示在页面上

若缺少数据，不要编造。

---

## 巡检模板（优化版 — 两步滤波法，<10s）

```python
import requests, json, concurrent.futures
from datetime import datetime, timedelta

API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
TOOLS_URL = "https://tiktok.vidau.ai/api/mcp/tools"

def mcp(name, args=None, timeout=15):
    """MCP 调用，timeout 默认 15s（别设 60s！）"""
    payload = {"jsonrpc": "2.0", "method": "tools/call",
               "params": {"name": name, "arguments": args or {}}, "id": 1}
    r = requests.post(f"{TOOLS_URL}?apiKey={API_KEY}", json=payload,
                      headers={"Content-Type": "application/json"}, timeout=timeout)
    return json.loads(r.json()["result"]["content"][0]["text"])

def fmt_bal(bal):
    if bal is None: return "N/A"
    try: return f"${float(bal):,.2f}"
    except: return str(bal)

# === 第一步：快速扫描 ===
today = datetime.now().strftime("%Y-%m-%d")
advs = mcp("list_advertisers")
active = [a for a in advs if a.get("campaignsCount", 0) > 0]
print(f"总账户: {len(advs)} | 有Campaign: {len(active)}")

# 并行查今日数据（只查 ADVERTISER 级别！）
def check_today(adv):
    try:
        m = mcp("get_daily_metrics", {
            "advertiser_id": adv["advertiserId"],
            "start_date": today, "end_date": today, "level": "ADVERTISER"
        })
        return adv, (len(m) > 0 and len(m[0]) > 0) if isinstance(m, list) else False
    except:
        return adv, False

with concurrent.futures.ThreadPoolExecutor(max_workers=5) as ex:
    futures = {ex.submit(check_today, a): a for a in active}
    accounts_with_data = []
    for f in concurrent.futures.as_completed(futures, timeout=30):
        adv, ok = f.result()
        if ok:
            accounts_with_data.append(adv)

print(f"有今日数据: {len(accounts_with_data)} 个账户")
print(f"跳跃账户（无今日数据）: {len(active) - len(accounts_with_data)} 个 — 跳过，不浪费请求")

# === 第二步：只对有今日数据的账户做深度巡检 ===
for adv in accounts_with_data:
    aid = adv["advertiserId"]
    name = adv["advertiserName"]
    # 拉今日 + 近7天对比数据
    m_today = mcp("get_daily_metrics", {"advertiser_id": aid, "start_date": today, "end_date": today, "level": "ADVERTISER"})
    m_7d = mcp("get_daily_metrics", {"advertiser_id": aid,
        "start_date": (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d"),
        "end_date": (datetime.now() - timedelta(days=1)).strftime("%Y-%m-%d"),
        "level": "ADVERTISER"})
    # 按阈值规则分析...（此处省略具体分析代码）
    print(f"\n📊 {name}: 已获取 {len(m_today)} 条今日数据 + {len(m_7d)} 条历史数据")
```

---

## ⚠️ 常见坑

### 速度类（重要！不遵守会慢 30 倍）

1. **`sync_advertiser_data` 极慢且易超时 🐌**：大账户（>10 Campaign）同步请求经常卡 60 秒超时。**巡检/日报场景绝对不要调它**，让用户在平台手动刷新即可。
2. **`get_daily_metrics(level=ADGROUP/CAMPAIGN/AD)` 基本都返回空**：平台只同步了 ADVERTISER 级别的报告。**永远先查 level=ADVERTISER**，只有它有数据。
3. **用两步滤波法替代全量扫**：先 `list_advertisers` 过滤 `campaignsCount > 0`，再单次 `get_daily_metrics(ADVERTISER)` 找出真正有今日数据的账户（通常 27 个里只有 1-2 个），只对这些做深度分析。滤波后 <10s vs 全量扫 5min。
4. **`ThreadPoolExecutor.as_completed` 必须设 `timeout=30`**：不加 timeout 的话一个卡死请求就让整个巡检卡住。
5. **单次 MCP 请求 timeout 设 15s 足够**，别设 60s。

### 格式/数据类

6. **`get_campaigns` / `get_adgroups` / `get_ads` 返回格式**：是 `{"data": [...], "total": N}`，不是直接数组。要取 `.data`。
7. **`create_*` 返回的 id 是本地的 UUID**，不是 TikTok 线上的 ID（线上 ID 是 `tiktokCampaignId` / `tiktokAdgroupId` 等字段）。
8. **并行调 MCP**：不同 `advertiser_id` 的调用互不依赖，用 `ThreadPoolExecutor` 并行大幅提速。
9. **`balance: null`** 不是 $0 — 是没同步报告数据。
10. **`STATUS_LIMIT` 账户** 的 Campaign 查询可能返回空。
11. **大账户（>200条 Campaign）** 分页拉，`pageSize` 最大设 200。
12. **`balance` 字段是字符串，不是数字**：MCP 返回的 `balance` 值是字符串（如 `"4003.86"`），不能直接用 `f"{bal:,.2f}"` 格式化，会报 `ValueError: Unknown format code 'f' for object of type 'str'`。**必须先用 `float(bal)` 转换**：
   ```python
   def fmt_bal(bal):
       if bal is None:
           return "N/A"
       try:
           return f"${float(bal):,.2f}"
       except (ValueError, TypeError):
           return str(bal)
   ```
13. **f-string 表达式内不能有反斜杠**：Python 3.11 及以下，f-string 的 `{}` 表达式内部不能包含反斜杠（如 `\n`、`\"`）。以下写法会报 `SyntaxError: f-string expression part cannot include a backslash`：
   ```python
   # ❌ 错误 — 反斜杠在 {} 内
   print(f"余额: {'$'+f'{adv.get(\"balance\"):,.2f}' if adv.get('balance') else 'N/A'}")
   # ✅ 正确 — 提前算好再放入 f-string
   bal_str = fmt_bal(adv.get("balance"))
   print(f"余额: {bal_str}")
   ```
   同理，`get_daily_metrics` 等返回的 `spend` 等数值字段也都是字符串，需要 `float()` 转换后再格式化。

---

## 打开面板

当用户说"打开面板"、"打开广告面板"、"打开 TikTok 面板"、"打开后台"时，**直接用 execute_code 在用户本机启动 Chrome**（不走 browser MCP，Google SSO 会拦截自动化浏览器）：

```python
import subprocess, os
chrome_paths = [
    r"C:\Program Files\Google\Chrome\Application\chrome.exe",
    r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe",
    os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe"),
]
chrome = next((p for p in chrome_paths if os.path.exists(p)), None)
if chrome:
    subprocess.Popen([chrome, "--new-window", "https://tiktok.vidau.ai/"],
                     stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
```

用户本地 Chrome 已经登录，直接打开就能看到面板。不要尝试 browser MCP 登录。

可用脚本：`~/AppData/Roaming/vidau/scripts/open_tiktok_panel.py`

---

## 与 ads-tiktok 的区别

| | ads-tiktok-mcp | ads-tiktok |
|---|---|---|
| 方式 | HTTP JSON-RPC 直连 | Browser MCP / Computer Use |
| 需要登录 | ❌ 不需要 | ✅ 需要 VidAU SSO |
| 速度 | 极快（~1s/调用） | 慢（需浏览器启动） |
| 适用 | 巡检、查询、轻量写入、广告创建 | 文件上传、OAuth、复杂UI交互 |
| API key | MCP 内置 tk_ key | 浏览器 session-token cookie |

**规则**：能用 MCP 就用 MCP，只有它搞不定的（上传素材、TikTok OAuth）才 fallback 到浏览器。
