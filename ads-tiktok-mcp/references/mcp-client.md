# MCP 调用规范（mcp-client）

本文件定义 ads-tiktok-mcp 与 `https://tiktok.vidau.ai/api/mcp/tools` 交互的**唯一正确调用方式**。所有业务模块都必须复用这里的 `mcp()`，禁止各模块自行实现请求逻辑。

> 本技能**以真实 VidAU MCP（`tiktok.vidau.ai`）返回体 / 接受值为准**（见 `references/enums-reference.md`）。VidAU 对 TikTok 字段做了简化封装，接受**简化枚举值**（如 `LEAD_GEN`/`WEB_CONVERSIONS`/`VIDEO_PLAY_3S`/`CPC`/`CPM`/`OCPM`）。枚举权威来源是 VidAU MCP 实际接受值；enums-reference 同时给出「VidAU 值 ↔ 官方 TikTok 值」对照。

---

## 1. 鉴权：API Key 必须来自环境变量

API Key 已内置预配置（脱敏，末尾 15 位以 x 代替），无需自行设置环境变量。

```python
import os
API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
if not API_KEY:
    raise RuntimeError("缺少环境变量 TIKTOK_MCP_API_KEY，请先配置后再调用")
TOOLS_URL = "https://tiktok.vidau.ai/api/mcp/tools"
```

API Key 已内置预配置（脱敏），调用方无需自行 export；如需更换见 SKILL.md 的 API_KEY 变量。

---

## 2. 健壮的 mcp() 实现（符合 JSON-RPC 2.0 规则）

关键点：
- 响应可能含 `"error"`（不是 `"result"`）→ 必须优先判断，否则 `result.content[0].text` 会 `KeyError` 崩溃。
- `result` / `content` / `text` 任意一层缺失都要兜底，不能假设结构一定存在。
- 超时、网络异常、非 JSON 响应都要转成可读中文错误，不得裸透传原始错误码。
- 单次请求 `timeout=15s`；并行 `as_completed(timeout=30)`。
- 连接复用（每线程一个 `Session`）+ 全局并发信号量，避免 429。

```python
import requests, json, threading, time, os

_api_key = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
TOOLS_URL = "https://tiktok.vidau.ai/api/mcp/tools"

# 每线程一个 Session，连接复用且线程安全
_thread_local = threading.local()
def _session():
    s = getattr(_thread_local, "session", None)
    if s is None:
        s = requests.Session()
        s.headers.update({"Content-Type": "application/json"})
        _thread_local.session = s
    return s

# 全局并发信号量：限制同时在途请求数，降低被限流概率
_CONCURRENCY = threading.Semaphore(5)

def _mask_key(k):
    return (k[:6] + "…" + k[-4:]) if k and len(k) > 10 else "***"

def mcp(name, args=None, timeout=15):
    """调用 VidAU MCP 工具，返回解析后的 data（dict / list / str）。"""
    if not _api_key:
        raise RuntimeError("缺少 TIKTOK_MCP_API_KEY，无法调用 MCP")
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {"name": name, "arguments": args or {}},
        "id": 1,
    }
    # 1) 传输层（限制并发，避免 429）
    with _CONCURRENCY:
        try:
            r = _session().post(
                f"{TOOLS_URL}?apiKey={_api_key}",
                json=payload,
                timeout=timeout,
            )
        except requests.exceptions.Timeout:
            raise RuntimeError(f"MCP 请求超时（>{timeout}s）：{name}")
        except requests.exceptions.RequestException as e:
            raise RuntimeError(f"MCP 网络错误：{e}")

    # 2) 解析层
    try:
        resp = r.json()
    except ValueError:
        raise RuntimeError(f"MCP 返回非 JSON（HTTP {r.status_code}），可能不是有效接口")

    # 3) 错误层（JSON-RPC error 优先于 result）
    if "error" in resp:
        err = resp["error"] or {}
        code = err.get("code", "unknown")
        msg = err.get("message", "无错误信息")
        raise RuntimeError(f"MCP 错误 [{code}]：{msg}")

    # 4) 结果层（逐层兜底，避免 KeyError/IndexError）
    result = resp.get("result")
    if not result:
        raise RuntimeError(f"MCP 返回空 result：{name}")
    content = result.get("content") or []
    if not content or not isinstance(content[0], dict) or "text" not in content[0]:
        raise RuntimeError(f"MCP 返回缺少文本内容：{name}")
    text = content[0]["text"]

    # 5) 文本可能是 JSON 字符串，也可能是纯文本
    try:
        return json.loads(text)
    except ValueError:
        return text
```

---

## 3. 统一重试（仅对可重试错误，带指数退避 + 抖动）

只对**网络超时 / 5xx / 限流 429** 重试；**参数错误 / 4xx 业务错误**不重试（重试无意义且放大错误）。

```python
import random

_RETRYABLE = ("超时", "网络错误", "429", "500", "502", "503", "504")

def mcp_retry(name, args=None, tries=3, base_delay=1.0):
    last = None
    for i in range(tries):
        try:
            return mcp(name, args)
        except RuntimeError as e:
            msg = str(e)
            last = e
            # 参数类 / 业务错误（含 "MCP 错误 [" 且非 429）不重试
            if "MCP 错误 [" in msg and "429" not in msg:
                raise
            if i < tries - 1 and any(k in msg for k in _RETRYABLE):
                time.sleep(base_delay * (2 ** i) + random.uniform(0, 0.5))  # 指数退避 + 抖动
                continue
            raise
    raise last
```

---

## 4. 两步滤波法（速度优化，全局统一一份）

所有「批量巡检 / 批量报表」场景都复用此模式，禁止各模块另写一套。

```python
from datetime import datetime, timedelta
import concurrent.futures

def list_active_advertisers_with_today_data(today=None, max_workers=5):
    """返回 (有今日数据的账户列表, 跳过数)。"""
    today = today or datetime.now().strftime("%Y-%m-%d")
    advs = mcp("list_advertisers")
    # 第一步：先过滤掉没有 Campaign 的空账户
    active = [a for a in advs if int(a.get("campaignsCount", 0) or 0) > 0]

    def has_today(adv):
        try:
            m = mcp("get_daily_metrics", {
                "advertiser_id": adv["advertiserId"],
                "start_date": today, "end_date": today,
                "level": "ADVERTISER",
            })
            # 兼容 list / dict / None：只要存在有效指标即视为有数据
            if isinstance(m, list):
                return bool(m) and bool(m[0])
            if isinstance(m, dict):
                return any(m.get(k) not in (None, 0, "0", "") for k in ("spend", "impressions", "clicks"))
            return False
        except RuntimeError:
            return False

    accounts_with_data = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as ex:
        futures = {ex.submit(has_today, a): a for a in active}
        for f in concurrent.futures.as_completed(futures, timeout=30):
            try:
                if f.result():
                    accounts_with_data.append(futures[f])
            except concurrent.futures.TimeoutError:
                continue
    return accounts_with_data, len(active) - len(accounts_with_data)
```

**规则**：
- 永远先 `list_advertisers` → 按 `campaignsCount > 0` 过滤 → 再对子集查 `level=ADVERTISER` 找今日有数据的账户。
- 只对有数据的账户做深度分析，其余跳过。
- 不同 `advertiser_id` 的调用互不依赖，用线程池并行；`as_completed` 必须带 `timeout`。

---

## 5. 报表日期用广告主时区

`get_daily_metrics` 的日期按**广告主时区**结算。`list_advertisers` 返回含 `timezone` 字段时优先用它换算；缺失时显式提示「按服务器时区，可能与后台存在偏差」。

```python
from zoneinfo import ZoneInfo
import datetime as _dt

def local_date_for(adv, server_now=None):
    server_now = server_now or _dt.datetime.now()
    tz = adv.get("timezone")
    if tz:
        try:
            return server_now.astimezone(ZoneInfo(tz)).strftime("%Y-%m-%d")
        except Exception:
            pass
    return server_now.strftime("%Y-%m-%d")  # 降级：服务器时区
```

---

## 6. 数值字段类型约定（统一，避免 fmt 报错）

- `balance`、`spend`、`cpc`、`cpm`、`ctr` 等**数值字段均为字符串或 `null`**，使用前必须 `float()`。
- `fmt_bal` 统一处理：

```python
def fmt_bal(bal):
    if bal is None:
        return "N/A（未同步）"
    try:
        return f"${float(bal):,.2f}"
    except (ValueError, TypeError):
        return str(bal)
```

### 字段类型速查表

| 字段 | 类型 | 取值说明 |
|------|------|----------|
| `advertiserId` | str | 广告主 ID（过滤用） |
| `campaignsCount` / `adgroupsCount` / `adsCount` | int/str | 可能为字符串，比较前 `int()` |
| `balance` | str / null | 字符串金额；`null`=未同步 |
| `status`（list_advertisers） | str | `STATUS_ENABLE` / `STATUS_LIMIT` |
| `status`（campaign/adgroup/ad） | str | `ENABLE` / `DISABLE` / `DRAFT` / `FAILED` |
| `spend` / `impressions` / `clicks` / `ctr` / `cpc` / `cpm` | str / null | 字符串数值，需 `float()` |
| `id`（create_* 返回） | str | **本地 UUID**（非线上 ID） |
| `tiktokCampaignId` 等 | str | **线上 TikTok ID** |

---

## 7. 参数预校验（提前暴露错误，给出友好提示）

在调用前校验必填项与枚举合法性，避免把非法请求发到 MCP 再拿晦涩错误。

```python
from references.enums_reference import (  # 若环境不支持包导入，改为从 enums-reference.md 内联常量
    OBJECTIVE_TYPES, OPTIMIZATION_GOALS, BILLING_EVENTS, BID_TYPES,
    PROMOTION_TYPES, PROMOTION_LEAD_PREFIX,
)

def validate_args(tool, args):
    """返回 None 表示通过；否则返回中文错误串。

    枚举校验同时接受 VidAU 简化值与其官方对应值（见 enums-reference §0），
    确保模型无论传哪种都不会因枚举校验报错。
    """
    req = {
        "get_campaigns": ["advertiser_id"],
        "create_campaign": ["advertiser_id", "campaign_name", "objective_type"],
        "create_adgroup": ["advertiser_id", "campaign_id", "adgroup_name", "targeting", "optimization_goal", "billing_event"],
        "create_ad": ["advertiser_id", "adgroup_id", "ad_text"],
        "get_daily_metrics": ["advertiser_id", "start_date", "end_date"],
    }.get(tool, [])
    for k in req:
        if not args or args.get(k) in (None, "", []):
            return f"缺少必填参数：{k}"
    # 枚举合法性（VidAU 简化值 ∪ 官方对应值，二者都放行）
    if args.get("objective_type") and args["objective_type"] not in OBJECTIVE_TYPES:
        return f"objective_type 非法：{args['objective_type']}，应为 {sorted(OBJECTIVE_TYPES)}"
    if args.get("optimization_goal") and args["optimization_goal"] not in OPTIMIZATION_GOALS:
        return f"optimization_goal 非法：{args['optimization_goal']}，应为 {sorted(OPTIMIZATION_GOALS)}"
    if args.get("billing_event") and args["billing_event"] not in BILLING_EVENTS:
        return f"billing_event 非法：{args['billing_event']}，应为 {sorted(BILLING_EVENTS)}"
    if args.get("bid_type") and args["bid_type"] not in BID_TYPES:
        return f"bid_type 非法：{args['bid_type']}，应为 {sorted(BID_TYPES)}"
    if args.get("promotion_type"):
        pt = args["promotion_type"]
        if pt not in PROMOTION_TYPES and not pt.startswith(PROMOTION_LEAD_PREFIX):
            return f"promotion_type 非法：{pt}，应为 {sorted(PROMOTION_TYPES)} 或以 {PROMOTION_LEAD_PREFIX} 开头"
    return None
```

> 若 `execute_code` 环境不支持 `from references...` 导入，请把上述枚举常量内联到调用脚本顶部（常量值见 `enums-reference.md`）。

---

## 8. 健康检查（会话开始时一次校验）

在开始批量操作前，先验证 key 有效性与连通性，早失败、早提示。

```python
def mcp_healthcheck():
    if not _api_key:
        return False, "未配置 TIKTOK_MCP_API_KEY"
    try:
        mcp("list_advertisers")
        return True, f"已连接（key={_mask_key(_api_key)}）"
    except RuntimeError as e:
        return False, f"连接失败：{e}"
```

---

## 9. 字段类型：create_* 返回的是本地 UUID

`create_campaign` / `create_adgroup` / `create_ad` 返回的 `id` 是**本地 UUID**，线上 TikTok ID 在 `tiktokCampaignId` / `tiktokAdgroupId` / `tiktokAdId` 字段中。后续 `get_adgroups(campaign_id=...)` / `get_ads(adgroup_id=...)` 用的也是**本地 UUID**，不是线上 ID——不要混用。

---

## 10. sync 策略（全局唯一口径）

- **交互式查询 / 巡检 / 日报默认不调用 `sync_advertiser_data`**（大账户极易 60s+ 超时）。
- 仅当用户**显式指定某个账户且缓存数据明显陈旧**时，才可调用，且必须：带 `timeout=60`、用 `mcp_retry`、失败则提示用户去平台手动刷新，绝不阻塞主流程。
- 禁止在并行滤波循环里调 sync。

---

## 11. 故障排查速查（Troubleshooting）

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| `缺少 TIKTOK_MCP_API_KEY` | 环境变量未设 | `export TIKTOK_MCP_API_KEY=...` |
| `MCP 返回非 JSON` | 端点/网关异常、key 无效被拦 | 检查 key 与 `TOOLS_URL`；用 `mcp_healthcheck()` |
| `MCP 错误 [401]` | key 无效/过期 | 重新获取并配置 key |
| `MCP 错误 [429]` | 触发限流 | 已自动退避重试；仍失败则降低并发或稍后重试 |
| `MCP 请求超时` | 账户过大/sync | 避免 sync；减小 pageSize；对大账户分页 |
| `MCP 返回空 result` | 工具名/参数错 | 用 `validate_args` 预校验；核对工具名 |
| `ValueError: Unknown format code 'f'` | 用 `f"{bal:,}"` 格式化字符串 | 先 `float(bal)`（见 §6） |
| 下钻 level 返回空 | VidAU 仅同步 ADVERTISER 级 | 降级 ADVERTISER 并注明，不编造（见 enums-reference §10） |
| `status` 过滤无效 | 用了 `ACTIVE` | 改用 `ENABLE`/`DISABLE`（见 enums-reference §8） |
