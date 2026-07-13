---
name: ads-tiktok-mcp
description: 直连 VidAU TikTok MCP（tiktok.vidau.ai）SSE 流式端点——无需登录、内置脱敏 Key。用 execute_code + requests 经 /api/mcp/sse 调用，支持广告巡检、预警、Campaign/Adgroup/Ad CRUD、日报/周报。常规操作走 MCP；仅面板打开、素材上传、TikTok OAuth 这类 MCP 做不到的才 fallback 浏览器。
version: 2.4.0
category: advertising
tags: [tiktok, ads, mcp, vidau, reporting, inspection, creative]
---

# TikTok Ads MCP — 直连 HTTP（最快路径）

**无需登录、无需 Python mcp 包。** 用 `execute_code` + `requests` 经 SSE 流式端点（`/api/mcp/sse?apiKey=…`）调用 VidAU MCP。

> 本技能是 **VidAU MCP 的封装**（VidAU 再封装 TikTok）。**字段/枚举以真实 VidAU MCP（`tiktok.vidau.ai`）返回体 / 接受值为准（VidAU 简化值，如 `LEAD_GEN`/`WEB_CONVERSIONS`/`VIDEO_PLAY_3S`/`CPC`/`CPM`/`OCPM`）**，权威对照见 `references/enums-reference.md`（该文件同时给出可直接导入的 Python 常量，供调用前 `validate_args` 预校验；校验同时放行官方对应值，确保不报错）。

**应用场景**：广告巡检、查预警、看报表、列出 Campaign/Ad/Adgroup、同步数据、广告创建。

### 技能文件结构

```
ads-tiktok-mcp/
├── SKILL.md                     # 总入口：调用规范、工具清单、业务模块、渲染/定时/坑
├── references/
│   ├── mcp-client.md            # ★ 唯一正确的 MCP 调用实现（容错/重试/滤波/校验/健康/排查）
│   ├── enums-reference.md       # ★ VidAU 简化枚举对照 + 可直接导入的 Python 常量
│   ├── ad-creation-flow.md      # 模块一：广告创建流程
│   ├── ad-inspection-rules.md   # 模块二：AI 巡检规则
│   └── report-format.md         # 模块三：日报/周报格式
└── skills/
    ├── knowledge/               # 知识层子 Skill（意图路由/知识问答/格式化）
    │   └── SKILL.md
    └── creative/                # ★ 素材创作层子 Skill（图片/视频/BGM/批量/爆款）
        ├── SKILL.md             # ★ 创作总入口（与 Ads 衔接协议）
        ├── references/
        │   └── creative-mcp-client.md  # Creative MCP 调用实现
        ├── L0-foundation/       # 基础层：平台/任务追踪/Prompt 工程/叙事路由
        ├── L1-capability/       # 生产层：一键生成/脚本转视频/批量编排
        └── L2-vertical/         # 垂直场景：TikTok爆款/商品链接转视频
```

> 任何模块需要调用 MCP，都必须 `from references.mcp_client import mcp, mcp_retry, validate_args`（或内联其实现），禁止重复造轮子。

---

## 前置条件

```bash
# API Key 已内置预配置（脱敏，末尾 15 位以 x 代替），无需自行设置环境变量
```

所有调用逻辑（健壮 `mcp()`、重试、两步滤波、时区、字段类型、参数预校验、健康检查）统一在 **`references/mcp-client.md`**，各模块不得自行实现请求。

> 会话开始建议先调用 `mcp_healthcheck()`（见 mcp-client §8）验证 key 与连通性，早失败早提示。

---

## 核心调用（速览，完整实现见 mcp-client.md）

```python
import requests, json, threading, queue
from urllib.parse import urljoin

API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
SSE_URL = f"https://tiktok.vidau.ai/api/mcp/sse?apiKey={API_KEY}"

# 完整容错/重试/两步滤波实现见 references/mcp-client.md；此处为 SSE 调用核心。
_sse = {"conn": None, "lock": threading.Lock()}

def _open_sse():
    sess = requests.Session()
    resp = sess.get(SSE_URL, headers={"Accept": "text/event-stream"}, stream=True, timeout=30)
    c = {"sess": sess, "resp": resp, "post_url": None, "pending": {}, "nid": 1,
         "ilock": threading.Lock(), "ready": threading.Event(), "failed": False}
    def reader():
        et = None; d = []
        try:
            for line in resp.iter_lines(decode_unicode=True):
                if line is None:
                    continue
                if line == "":
                    if et == "endpoint":
                        u = "".join(d).strip()
                        pu = u if u.startswith("http") else urljoin(SSE_URL, u)
                        if "apiKey=" not in pu:
                            pu += ("&" if "?" in pu else "?") + f"apiKey={API_KEY}"
                        c["post_url"] = pu; c["ready"].set()
                    elif et == "message":
                        try:
                            m = json.loads("".join(d))
                        except ValueError:
                            m = None
                        if isinstance(m, dict) and "id" in m:
                            q = c["pending"].pop(m["id"], None)
                            if q:
                                q.put(m)
                    et = None; d = []
                    continue
                if line.startswith("event:"):
                    et = line[6:].strip()
                elif line.startswith("data:"):
                    d.append(line[5:].lstrip())
        except Exception:
            pass
        c["failed"] = True; c["ready"].set()
    threading.Thread(target=reader, daemon=True).start()
    if not c["ready"].wait(20):
        c["failed"] = True; raise RuntimeError("MCP SSE 握手超时")
    return c

def mcp(name, args=None, timeout=15):
    with _sse["lock"]:
        if _sse["conn"] is None or _sse["conn"].get("failed") or _sse["conn"]["resp"].raw.closed:
            _sse["conn"] = _open_sse()
        c = _sse["conn"]
    if c.get("failed"):
        raise RuntimeError("MCP SSE 连接已断开，请重新运行会话")
    with c["ilock"]:
        rid = c["nid"]; c["nid"] += 1
    q = queue.Queue(); c["pending"][rid] = q
    payload = {"jsonrpc":"2.0","method":"tools/call","params":{"name":name,"arguments":args or {}},"id":rid}
    try:
        r = requests.post(c["post_url"], json=payload, timeout=timeout)
    except requests.exceptions.Timeout:
        c["pending"].pop(rid, None); raise RuntimeError(f"MCP 请求超时（>{timeout}s）：{name}")
    except requests.exceptions.RequestException as e:
        c["pending"].pop(rid, None); raise RuntimeError(f"MCP 网络错误：{e}")
    msg = None
    if r.status_code == 200 and r.content:
        try:
            msg = r.json()
        except ValueError:
            msg = None
    if msg is not None and ("result" in msg or "error" in msg):
        pass
    else:
        try:
            msg = q.get(timeout=timeout)
        except queue.Empty:
            c["pending"].pop(rid, None); raise RuntimeError(f"MCP 响应超时（>{timeout}s）：{name}")
    c["pending"].pop(rid, None)
    if "error" in msg:
        err = msg["error"] or {}
        raise RuntimeError(f"MCP 错误 [{err.get('code','?')}]：{err.get('message','')}")
    result = msg.get("result")
    if not result:
        raise RuntimeError(f"MCP 返回空 result：{name}")
    content = result.get("content") or []
    if not content or "text" not in content[0]:
        raise RuntimeError(f"MCP 返回无文本内容：{name}")
    try:
        return json.loads(content[0]["text"])
    except ValueError:
        return content[0]["text"]
```

**传输方式**：经 SSE 流式端点（`/api/mcp/sse?apiKey=…`）调用——`_sse_reader` 监听 `endpoint` 事件拿到消息端点后 POST JSON-RPC，结果经同一 SSE 流异步回传。

---

## 10 个 MCP 工具

### 1. list_advertisers — 列出所有广告主
```python
advs = mcp("list_advertisers")
# 返回 [{id, advertiserId, advertiserName, status, balance, currency, timezone,
#          campaignsCount, adgroupsCount, adsCount, creativeCount,
#          lastEntitySyncAt, lastReportSyncAt, ...}]
```
- `balance` 为**字符串或 `null`**（`null` = 未同步，不是 $0）。
- `status`: `STATUS_ENABLE` / `STATUS_LIMIT`（受限）。
- **巡检/报表起点**：先调它，再按 `campaignsCount > 0` 过滤活跃账户。

### 2. get_alerts — 获取系统预警
```python
alerts = mcp("get_alerts", {"advertiser_id": "7644525617435574290", "limit": 20})
```
- 必填: `advertiser_id`；可选: `is_read` (bool), `limit` (默认 20)。

### 3. get_campaigns — 获取 Campaign 列表
```python
camps = mcp("get_campaigns", {
    "advertiser_id": "7644525617435574290",
    "status": "ENABLE",      # 可选: ENABLE / DISABLE / DRAFT / FAILED；传 "all" 不过滤
    "search": "关键词",       # 可选：名称模糊搜索
    "page": 1, "pageSize": 50
})
# 返回 {"data": [...], "total": N, "page": 1, "pageSize": 50}
```
- ⚠️ 过滤值用 `ENABLE`/`DISABLE`，**不要用 `ACTIVE`**（非有效枚举）。
- 返回是 `{"data":[...]}`，取 `.data`。

### 4. create_campaign — 创建 Campaign
```python
camp = mcp("create_campaign", {
    "advertiser_id": "7644525617435574290",
    "campaign_name": "我的新Campaign",
    "objective_type": "WEB_CONVERSIONS",  # VidAU 简化值，见 enums-reference §1（官方对应 CONVERSIONS）
    "budget_mode": "BUDGET_MODE_INFINITE", # BUDGET_MODE_INFINITE / DAY / TOTAL
    "budget": 0,                          # 0=不限
    "budget_optimize_switch": False       # CBO 预算优化
})
# 返回 id=本地UUID；线上ID在 tiktokCampaignId
```

### 5. get_adgroups — 获取 Adgroup 列表
```python
adgs = mcp("get_adgroups", {
    "advertiser_id": "7644525617435574290",
    "campaign_id": "本地的 campaign UUID",  # 可选，非 tiktokCampaignId
    "status": "ENABLE",
    "search": "名称搜索", "page": 1, "pageSize": 50
})
```

### 6. create_adgroup — 创建 Adgroup
```python
adg = mcp("create_adgroup", {
    "advertiser_id": "7644525617435574290",
    "campaign_id": "本地的 campaign UUID",   # 必填
    "adgroup_name": "广告组名称",             # 必填
    "targeting": {                           # 必填
        "location_ids": ["100"],             # 必填: 国家/地区ID，"100"=美国
        "gender": "NONE",                    # GENDER_FEMALE / GENDER_MALE / NONE
        "age_groups": [], "languages": []
    },
    "budget_mode": "BUDGET_MODE_DAY",        # DAY / TOTAL
    "budget": 50,
    "schedule_type": "SCHEDULE_FROM_NOW",    # FROM_NOW / START_END
    "optimization_goal": "CLICK",            # VidAU 简化值，见 enums-reference §3
    "billing_event": "CPC",                  # VidAU 简化值，见 enums-reference §4
    "bid_type": "BID_TYPE_NO_BID",           # 见 enums-reference §5 出价映射
    "bid_price": 0,
    "pacing": "PACING_MODE_SMOOTH",          # SMOOTH / FAST
    "promotion_type": "WEBSITE",             # 官方枚举，见 enums-reference §6
    "placement_type": "PLACEMENT_TYPE_AUTOMATIC",
    "placements": []
})
```

### 7. get_ads — 获取 Ad 列表
```python
ads = mcp("get_ads", {
    "advertiser_id": "7644525617435574290",
    "adgroup_id": "本地 adgroup UUID", "status": "ENABLE",
    "search": "搜索词", "page": 1, "pageSize": 50
})
```

### 8. create_ad — 创建 Ad
```python
ad = mcp("create_ad", {
    "advertiser_id": "7644525617435574290",
    "adgroup_id": "本地 adgroup UUID",       # 必填
    "ad_text": "广告文案",                    # 必填
    "ad_name": "可选名称",
    "creative_id": "素材库UUID",              # 可经素材库 upload-to-tiktok 端点上传后回填（见模块四）
    "video_id": "已上传的TikTok视频ID",
    "cover_url": "封面图URL",
    "call_to_action": "LEARN_MORE",           # 默认 LEARN_MORE / DYNAMIC=动态优选
    "landing_page_url": "落地页URL"
})
```
- ⚠️ **素材上传能力状态**：平台后端已实现 `material_save_from_url` / `material_upload_to_tiktok` 两个 REST 端点（见模块四）；但 **MCP 网关尚未注册这两个工具**。在 MCP 工具可用前，Agent 按 ad-creation-flow 的兜底步骤引导用户先上传；工具注册后 `creative_id`/`video_id` 可由 Agent 自动取得。

### 9. get_daily_metrics — 日报指标
```python
metrics = mcp("get_daily_metrics", {
    "advertiser_id": "7644525617435574290",
    "start_date": "2026-07-01", "end_date": "2026-07-07",
    "level": "ADVERTISER"   # ADVERTISER / CAMPAIGN / ADGROUP / AD
})
```
- 必填: `advertiser_id`, `start_date`, `end_date`；`level` 默认按广告主聚合。
- 下钻层级（CAMPAIGN/ADGROUP/AD）**可能为空**：调用后若为空，降级 ADVERTISER 并在报告中注明，不编造明细（见 enums-reference §10）。

### 10. sync_advertiser_data — 同步实体数据
```python
mcp("sync_advertiser_data", {
    "advertiser_id": "7644525617435574290",
    "levels": ["ADVERTISER","CAMPAIGN","ADGROUP","AD"]
}, timeout=60)
```
- ⚠️ 大账户极易 60s+ 超时，**交互式查询默认不调用**；仅用户显式指定账户且缓存陈旧时使用，失败提示去平台手动刷新（见 mcp-client.md §8）。

---

## 业务模块总览

基于 MCP 工具实现三大模块。**触发关键词决定走哪个模块**；多关键词命中时按「创建 → 巡检 → 报表」顺序处理。模块详细规则分别在 `references/` 下。

### 触发关键词（已收窄，降低误触）

| 模块 | 关键词 |
|------|--------|
| 广告创建 | 创建广告、建广告、投广告、投放广告、我要投放、帮我投放、创建TikTok广告、搭建广告、批量创建广告、批量投放、帮我把广告发布出去、推广投放 |
| AI 巡检 | 广告巡检、巡检、开启巡检、启动巡检、开始巡检、监控广告、AI巡检 |
| 日报/周报 | 生成日报、给我生成日报、生成周报、给我生成周报、日报数据、昨天日报、今日数据报告、周报数据、本周周报、根据我的格式生成日报、根据我的格式生成周报 |
| **素材创作** | **做个视频、生成视频、创作素材、AI生图、做个海报、生成图片、帮我设计、素材创作、创意视频、广告素材、爆款视频、批量生图、这个链接做个视频** |

> 说明：「确认 / 好的 / 需要 / 开始」**不是全局触发词**。它们仅在**本技能已发出巡检询问的上下文**中，表示用户同意开启巡检；不会因对话里出现这些词就误触发。

指定日期/账户格式：`x天日报`、`x日-x日的日报`、`广告账户名称 xxx 的日报/周报`、`广告系列名称 xxx 日报/周报` 等。

---

## 巡检自动触发条件（上下文式）

除用户主动说巡检关键词外，以下场景**先询问、不自动执行**；用户在该询问上下文里回「确认/好的/需要/开始」才真正开启：

### 场景 1：广告创建完成后
> "广告已创建完成，是否需要帮您创建广告巡检呢？覆盖花费、转化、CPA、ROAS、预算、素材等方面进行AI监测，可以回复「确认」进行开启"

### 场景 2：预算花费超过 60%
> "当前检测到您的广告预算已花费超 60%，是否需要开启 AI 广告巡检呢？可以回复「确认」进行开启"

### 场景 3：检测到有在投放的广告
> "检测到您有正在投放的广告，是否需要开启 AI 广告巡检呢？可以回复「确认」进行开启"

### 场景 4：每日定时主动询问
- 每天早上 **9:45**、下午 **15:00** 各询问一次（由定时任务触发，见下文）。

---

## 模块一：广告创建（详见 references/ad-creation-flow.md）

投放前检查：账户授权、余额、推广目标、优化目标、素材（已上传+审核通过）、定向、预算、出价。不通过不得进入提交，并提示原因。

创建流程按面板依次收集：账户 → 推广目标 → 优化目标 → 素材 → 定向 → 预算 → 出价。预算检测：同账户/目标/优化下，对比 7 日均值，超阈值提示建议区间。所有高风险操作（创建/修改/发布）须用户明确确认后方可执行。

**MCP 工具映射**：`list_advertisers` → `create_campaign` → `create_adgroup` → `create_ad`。

创建完成返回状态：成功（返回各层 ID + 触发巡检询问）/ 失败（重试≤3 → 仍失败给原因+建议字段）/ 进行中（明确提示"等待接口返回"，不默认成功）。**输出不得编造任何结果或状态**。

---

## 模块二：AI 广告巡检（详见 references/ad-inspection-rules.md）

提醒广告数据变化趋势。覆盖六类异常（按优先级）：花费异常 > 转化异常 > CPA 异常 > ROAS 异常 > 预算耗尽 > 素材疲劳。

**MCP 执行（两步滤波法，全程 < 10s，见 mcp-client.md §4）**：
1. `list_advertisers` → 过滤 `campaignsCount > 0`
2. 并行 `get_daily_metrics(level=ADVERTISER)` → 找今日真有数据的账户
3. 只对有数据的账户拉 7 天历史做阈值对比

> ⚠️ 不调 `sync_advertiser_data`（交互场景）⚠️ `as_completed` 设 `timeout=30` ⚠️ 单次 `timeout=15`

---

## 模块三：日报/周报（详见 references/report-format.md）

生成日报/周报，支持原模板或自定义格式。日报重今日表现/环比/问题广告/优秀广告/素材亮点/风险/明日动作；周报重趋势/归因/复盘/复制方向/下周动作。

**MCP 执行**：
1. `list_advertisers`
2. 并行 `get_daily_metrics`（advertiser_id + 日期区间）
3. 尝试 `level=CAMPAIGN/ADGROUP` 下钻识别问题/优秀广告；**若为空降级 ADVERTISER 并注明**，不编造
4. 按 report-format 输出 Markdown

账户过滤：仅已授权且 `balance` 非 `null` 的账户；指定账户时按 mcp-client §8 处理（不盲目 sync）。

---

## 模块四：素材创作（详见 skills/creative/SKILL.md）★ 一体化核心

> **本模块将 VidAU Creative Agent 的素材生成能力融入 TikTok Ads Specialist，实现「创作→投放」一个 Agent 全链路。**
> 完整文档在 `skills/creative/SKILL.md`（含 L0/L1/L2 三层 11 个子 Skill）。此处为执行层的衔接摘要。

### 能力概览

| 创作类型 | Skill 层 | MCP 工具 | 典型场景 |
|----------|---------|----------|----------|
| 单张图片 / 单个短视频 ≤15s | L1 creative-direct | `creative_generate_image` / `creative_generate_video` | 快速产出一条广告素材 |
| 多镜头广告视频 16–120s | L1 script2film | `creative_submit_script2film` | 品牌片 / 功能展示 |
| 批量爆款变体（TikTok 专属） | **L2 trend-viral-short** | `creative_submit_batch_variants` | A/B 测试矩阵、hook 测试 |
| 商品链接→视频 | **L2 product-url-to-video** | 爬取 + 任意 L1 MCP | 贴链接自动出视频 |
| BGM 生成/混入 | L0+L1 | `creative_generate_bgm` / `creative_mux_bgm_into_video` | 视频配乐 |

### Creative MCP 端点（独立于 TikTok Ads MCP）

> **鉴权方式**：走平台会话（`vidau_user_id` header），**无需独立 API Key**。
> 端点: `https://creative.vidau.ai/mcp`（Streamable HTTP 模式）
>
> 配置参考：`{"url": "https://creative.vidau.ai/mcp", "enabled": true}`

调用实现见 `skills/creative/references/creative-mcp-client.md`（与 mcp-client.md 对齐的容错/重试模式）。

### ★ 创作→投放衔接协议

当用户意图同时包含「创作」+「投放」时：

1. **Creative Skill 产素材** → 返回 `{ video_url, image_url, download_url, artifacts }`
2. **询问用户是否用于投放** → 用户确认
3. **映射到 Ads 参数**：
   - 图片 URL → `create_ad.cover_url`（✅ 直接可用）
   - 视频 download URL → 经 `upload-to-tiktok` 端点上传至 TikTok CDN 得到 `video_id`（**后端已实现，待 MCP 网关注册工具**）
4. **进入模块一创建流程** → `create_campaign` → `create_adgroup` → `create_ad({ cover_url: <产出URL> })`
5. 返回广告 ID + 触发巡检

### 当前限制 & 解决路线图

| 限制 | 状态 | 解决方案 |
|------|------|----------|
| Creative 视频不能直接经 MCP 自动用作 `video_id` | ✅ 后端已实现（upload-to-tiktok 端点），🔧 待 MCP 网关注册工具 | 平台已实现 `POST /api/creatives/[id]/upload-to-tiktok`（对应 `material_upload_to_tiktok`）；MCP 网关注册该工具后 Agent 可直接调用（见 `docs/MCP-MATERIAL-LIBRARY-SPEC.md`） |
| 图片 URL 可直接做 `cover_url` | ✅ 已解决 | 无缝衔接 |
| 两个 MCP 端点需分别配置 | ✅ 已解决 | Ads 用 API Key；Creative 走平台会话，无需独立 Key |
| 创作产出无法自动回写素材库 | ✅ 后端已实现（save-from-url 端点），🔧 待 MCP 网关注册工具 | 平台已实现 `POST /api/creatives/save-from-url`（对应 `material_save_from_url`）；MCP 网关注册该工具后自动入库（标记 vidau.ai 来源） |

> **实现状态**：4 个工具中，`material_save_from_url` / `material_upload_to_tiktok` 的**后端 REST 端点已在平台源码实现**（`tiktok-ads-agent` 仓库）；`material_list` / `material_sync_from_tiktok` 此前已存在。剩余工作为 **MCP 网关将这 4 个端点注册为 MCP 工具**（见 `docs/MCP-MATERIAL-LIBRARY-SPEC.md`，可直接提交 VidAU 工程团队）。

### Prompt Gate 强制规则

**所有图片/视频生成前必须经过 Prompt 工程 Layer**：
- 图片 → 先加载 `creative-gpt-image2-prompt`（六块协议）
- 视频 → 先加载 `creative-seedance2-prompt`（SCELA 结构）
- 脚本 → 先加载 `creative-narrative-router`（叙事结构路由）
- **禁止**将用户原始文本直接作为 `prompt` 传入 Creative MCP

---

## 输出渲染策略（降级）

1. 支持 ui_payload → 结构化 JSON + ui_payload（巡检概览/异常卡/建议；日报指标卡/环比/广告列表）
2. 不支持 → Markdown 卡片
3. 仍不行 → 简洁摘要，**禁止**在界面直接展示完整 raw JSON

---

## 定时任务

- 定时巡检：「每天x点开启巡检」→ 创建定时任务到点执行并推送；「关闭定时巡检」→ 移除。
- 定时日报/周报：「每天x点生成日报」「每周星期x生成周报」→ 创建定时任务；「根据我的格式生成日报」→ 保存为模板。
- 定时自动同步：「开启自动同步」→ 创建 cron 任务 `*/15 * * * *`（宿主时区），每 15 分钟自动同步所有已授权账户的广告系列/广告组/广告/素材（见 mcp-client.md §12）；「关闭自动同步」→ 移除该任务。
- 定时能力由宿主平台提供（cron 表达式 + 宿主时区）；若平台无定时能力，改为在会话内提示用户手动触发。

---

## 报告生成成功判定

全部满足才算成功：1) 已识别关键词；2) 已读取广告主列表；3) 已过滤有效账户；4) 已获取指标；5) 已生成渲染/Markdown；6) 回复优先展示在页面。缺数据不编造。

---

## ⚠️ 常见坑（统一口径）

1. **`sync_advertiser_data` 极慢易超时**：交互场景默认不调，让用户平台刷新（mcp-client §8）。
2. **下钻 level 可能为空**：优先 ADVERTISER；CAMPAIGN/ADGROUP/AD 为空则降级并注明，不编造。
3. **两步滤波法**替代全量扫：先 `campaignsCount>0` 过滤，再查 ADVERTISER 找今日有数据账户，只对这些深度分析（<10s vs 全量 5min）。
4. **`ThreadPoolExecutor.as_completed` 必须 `timeout=30`**；单次 `timeout=15`。
5. **`get_*` 返回 `{"data":[...]}`**，取 `.data`。
6. **`create_*` 返回本地 UUID**，线上 ID 在 `tiktokXxxId`；后续 `get_*` 用本地 UUID。
7. **数值字段是字符串**：`balance`/`spend` 等需 `float()` 后格式化（用 `fmt_bal`）。
8. **`status` 用 `ENABLE`/`DISABLE`，不用 `ACTIVE`**（非有效枚举）。
9. **`balance: null`** = 未同步，不是 $0。
10. **`STATUS_LIMIT` 账户** Campaign 查询可能返回空。
11. **大账户分页**：`pageSize` 最大 200。
12. **枚举使用 VidAU 简化值**：`objective_type`/`optimization_goal`/`billing_event`/`bid_type`/`promotion_type` 均用 VidAU MCP 实际接受的简化值（如 `LEAD_GEN`、`WEB_CONVERSIONS`、`VIDEO_PLAY_3S`、`CPC`/`CPM`/`OCPM`），完整对照与可直接导入的 Python 常量见 `references/enums-reference.md`；调用前用 `validate_args` 预校验（mcp-client §7，已同时放行官方对应值，确保不报错）。

---

## 打开面板 / 浏览器 fallback（明确边界）

仅以下 MCP 做不到的场景才用浏览器：
- 「打开面板/打开 TikTok 后台」→ 用 `subprocess` 启本地已登录 Chrome（不走 browser MCP，SSO 会拦自动化浏览器）。
- 素材上传、TikTok OAuth → fallback 浏览器/面板操作。

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

可用脚本：`~/AppData/Roaming/vidau/scripts/open_tiktok_panel.py`

---

## 与 ads-tiktok 的区别

| | ads-tiktok-mcp | ads-tiktok |
|---|---|---|
| 方式 | HTTP JSON-RPC 直连 VidAU MCP | Browser MCP / Computer Use |
| 需要登录 | ❌ 不需要 | ✅ 需要 VidAU SSO |
| 速度 | 极快（~1s/调用） | 慢（需浏览器启动） |
| 适用 | 巡检、查询、轻量写入、广告创建 | 文件上传、OAuth、复杂 UI 交互 |
| 鉴权 | 内置脱敏 API Key（硬编码 `tk_1bfa…5xxx`，SSE 端点） | 浏览器 session-token cookie |

**规则**：能用 MCP 就用 MCP，只有它搞不定的（上传素材、TikTok OAuth、开面板）才 fallback 浏览器。

---

## 版本基线与变更（Changelog）

- **对齐基线**：真实 **VidAU MCP（`tiktok.vidau.ai`）返回体 / 接受值**（VidAU 简化枚举）；VidAU 再封装 TikTok。
- **v2.4.0**（本次）
  - **新增「模块四：素材创作」** —— 融合 VidAU Creative Agent 全部能力（L0/L1/L2 三层 11 个子 Skill）。
  - 新增 `skills/creative/` 目录：一键生成、脚本转视频、批量编排、TikTok 爆款变体、商品链接转视频。
  - 创作→投放衔接协议：Creative 产出物直接传入 `create_ad`，实现一个 Agent 全链路。
  - 新增 Creative MCP 端点支持（`creative.vidau.ai`，平台会话鉴权，标准部署无需独立 Key），与 TikTok Ads MCP 并行。
  - Prompt Gate 强制规则：所有生成前必须经过 Seedance / GPT-Image-2 Prompt 工程。
  - `trend-viral-short` 为 TikTok 广告专属爆款 Skill；`product-url-to-video` 支持贴链接自动出视频。
  - 保留 v2.3.0 全部内容不变；新增模块为增量添加。
- **v2.3.0**（上次）
  - **按真实 VidAU MCP 返回体校准**：枚举全面切回 **VidAU 简化值**（`LEAD_GEN`/`WEB_CONVERSIONS`/`VIDEO_PLAY_3S`/`CPC`/`CPM`/`OCPM`），确保调用不会因枚举不匹配报错。
  - `validate_args` 同时放行 VidAU 简化值与其官方对应值，无论模型传哪种都不会因枚举校验报错。
  - 保留 v2.1/v2.2 全部修复：API Key 外置、mcp() 容错、重试退避、两步滤波、时区、字段类型、健康检查、巡检误触收窄、status 统一、level 降级、sync 策略统一。
- **v2.2.0**：曾切换为官方 TikTok 枚举（与 VidAU 实际接受值不符，已回滚）。
- **v2.1.0**：修复 P0/P1（API Key 外置、mcp() 容错、巡检误触收窄、status 统一、level 降级、sync 策略统一）。
- **v2.0.0**：初版（VidAU MCP 直连 HTTP）。

> 升级提示：本版本以真实 VidAU MCP 接受值为准，请使用 VidAU 简化枚举（`LEAD_GEN`/`WEB_CONVERSIONS`/`VIDEO_PLAY_3S`/`CPC`/`CPM`/`OCPM`）。`validate_args` 同时兼容官方对应值，不会因枚举报错。
