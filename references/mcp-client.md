# MCP 调用规范（mcp-client）

本文件定义 ads-tiktok-mcp 经 **SSE 流式端点** `https://tiktok.vidau.ai/api/mcp/sse?apiKey=…`（API Key 已内置脱敏）调用 VidAU MCP 的**唯一正确方式**。所有业务模块都必须复用这里的 `mcp()`，禁止各模块自行实现请求逻辑。

> 本技能**以真实 VidAU MCP（`tiktok.vidau.ai`）返回体 / 接受值为准**（见 `references/enums-reference.md`）。VidAU 对 TikTok 字段做了简化封装，接受**简化枚举值**（如 `LEAD_GEN`/`WEB_CONVERSIONS`/`VIDEO_PLAY_3S`/`CPC`/`CPM`/`OCPM`）。枚举权威来源是 VidAU MCP 实际接受值；enums-reference 同时给出「VidAU 值 ↔ 官方 TikTok 值」对照。

---

## 1. 鉴权：API Key 已内置预配置（脱敏）

API Key 已内置预配置（脱敏，末尾 15 位以 x 代替）；请勿将其替换为明文真实 Key。

```python
API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
# Ads MCP 走 SSE 流式端点（API Key 已内置脱敏，末尾 15 位以 x 代替）
SSE_URL = f"https://tiktok.vidau.ai/api/mcp/sse?apiKey={API_KEY}"
if not API_KEY:
    raise RuntimeError("未内置 API Key，无法调用 MCP（请检查 SKILL.md 的 API_KEY 变量）")
```

API Key 已内置预配置（脱敏），调用方无需自行 export；如需更换见根 SKILL.md 的 API_KEY 变量。

---

## 2. 健壮的 mcp() 实现（符合 JSON-RPC 2.0 规则）

关键点：
- 响应可能含 `"error"`（不是 `"result"`）→ 必须优先判断，否则 `result.content[0].text` 会 `KeyError` 崩溃。
- `result` / `content` / `text` 任意一层缺失都要兜底，不能假设结构一定存在。
- 超时、网络异常、非 JSON 响应都要转成可读中文错误，不得裸透传原始错误码。
- 单次请求 `timeout=15s`；并行 `as_completed(timeout=30)`。
- 连接复用（每线程一个 `Session`）+ 全局并发信号量，避免 429。

```python
import requests, json, threading, time, queue, os
from urllib.parse import urljoin

API_KEY = "tk_1bfa861961110ed257b517680da9efeb5xxxxxxxxxxxxxxx"
SSE_URL = f"https://tiktok.vidau.ai/api/mcp/sse?apiKey={API_KEY}"

# 全局并发信号量：限制同时在途请求数，降低被限流概率（POST 用临时 Session，天然线程安全）
_CONCURRENCY = threading.Semaphore(5)

# --- SSE 会话（进程级单例，懒初始化、失败自动重连）---
_sse_lock = threading.Lock()
_sse_conn = None

def _sse_reader(conn):
    event_type = None
    data_parts = []
    try:
        for raw in conn["resp"].iter_lines(decode_unicode=True):
            if conn.get("stop"):
                break
            if raw is None:
                continue
            if raw == "":
                if event_type == "endpoint":
                    url = "".join(data_parts).strip()
                    post_url = url if url.startswith("http") else urljoin(SSE_URL, url)
                    if "apiKey=" not in post_url:
                        post_url += ("&" if "?" in post_url else "?") + f"apiKey={API_KEY}"
                    conn["post_url"] = post_url
                    conn["ready"].set()
                elif event_type == "message":
                    try:
                        msg = json.loads("".join(data_parts))
                    except ValueError:
                        msg = None
                    if isinstance(msg, dict) and "id" in msg:
                        q = conn["pending"].pop(msg["id"], None)
                        if q:
                            q.put(msg)
                event_type = None
                data_parts = []
                continue
            if raw.startswith("event:"):
                event_type = raw[6:].strip()
            elif raw.startswith("data:"):
                data_parts.append(raw[5:].lstrip())
    except Exception:
        pass
    conn["failed"] = True
    conn["ready"].set()

def _new_sse():
    sess = requests.Session()
    resp = sess.get(SSE_URL, headers={"Accept": "text/event-stream"},
                    stream=True, timeout=30)
    if resp.status_code != 200:
        raise RuntimeError(f"MCP SSE 连接失败（HTTP {resp.status_code}）")
    conn = {"session": sess, "resp": resp, "post_url": None,
            "pending": {}, "next_id": 1, "id_lock": threading.Lock(),
            "ready": threading.Event(), "failed": False, "stop": False}
    threading.Thread(target=_sse_reader, args=(conn,), daemon=True).start()
    if not conn["ready"].wait(timeout=20):
        conn["failed"] = True
        raise RuntimeError("MCP SSE 握手超时（未收到 endpoint 事件）")
    return conn

def _ensure_sse():
    global _sse_conn
    with _sse_lock:
        if _sse_conn is None or _sse_conn.get("failed") or getattr(_sse_conn["resp"].raw, "closed", True):
            _sse_conn = _new_sse()
        return _sse_conn

def _mask_key(k):
    return (k[:6] + "…" + k[-4:]) if k and len(k) > 10 else "***"

def mcp(name, args=None, timeout=15):
    """经 SSE 流式端点调用 VidAU MCP 工具，返回解析后的 data（dict / list / str）。"""
    if not API_KEY:
        raise RuntimeError("未内置 API Key，无法调用 MCP")
    conn = _ensure_sse()
    if conn.get("failed"):
        raise RuntimeError("MCP SSE 连接已断开，请重新运行会话")
    with conn["id_lock"]:
        rid = conn["next_id"]; conn["next_id"] += 1
    q = queue.Queue()
    conn["pending"][rid] = q
    payload = {"jsonrpc": "2.0", "method": "tools/call",
               "params": {"name": name, "arguments": args or {}}, "id": rid}
    with _CONCURRENCY:
        try:
            r = requests.post(conn["post_url"], json=payload, timeout=timeout)
        except requests.exceptions.Timeout:
            conn["pending"].pop(rid, None)
            raise RuntimeError(f"MCP 请求超时（>{timeout}s）：{name}")
        except requests.exceptions.RequestException as e:
            conn["pending"].pop(rid, None)
            raise RuntimeError(f"MCP 网络错误：{e}")
        # 部分实现会在 POST 响应体直接返回结果；否则结果经 SSE 流异步返回
        post_msg = None
        if r.status_code == 200 and r.content:
            try:
                post_msg = r.json()
            except ValueError:
                post_msg = None
        if post_msg is not None and ("result" in post_msg or "error" in post_msg):
            msg = post_msg
        else:
            try:
                msg = q.get(timeout=timeout)
            except queue.Empty:
                conn["pending"].pop(rid, None)
                raise RuntimeError(f"MCP 响应超时（>{timeout}s）：{name}")
        conn["pending"].pop(rid, None)
    return _parse_mcp(msg, name)

def _parse_mcp(msg, name):
    if "error" in msg:
        err = msg.get("error") or {}
        code = err.get("code", "unknown")
        msg_text = err.get("message", "无错误信息")
        raise RuntimeError(f"MCP 错误 [{code}]：{msg_text}")
    result = msg.get("result")
    if not result:
        raise RuntimeError(f"MCP 返回空 result：{name}")
    content = result.get("content") or []
    if not content or not isinstance(content[0], dict) or "text" not in content[0]:
        raise RuntimeError(f"MCP 返回缺少文本内容：{name}")
    text = content[0]["text"]
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
    if not API_KEY:
        return False, "未内置 API Key"
    try:
        mcp("list_advertisers")
        return True, f"已连接（key={_mask_key(API_KEY)}）"
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
| `未内置 API Key` | SKILL.md 的 API_KEY 缺失 | 检查根 SKILL.md 的 API_KEY 变量（已脱敏，末尾 15 位 x） |
| `MCP 返回非 JSON` | 端点/网关异常、key 无效被拦 | 检查 key 与 `SSE_URL`（/api/mcp/sse）；用 `mcp_healthcheck()` |
| `MCP 错误 [401]` | key 无效/过期 | 重新获取并配置 key |
| `MCP 错误 [429]` | 触发限流 | 已自动退避重试；仍失败则降低并发或稍后重试 |
| `MCP 请求超时` | 账户过大/sync | 避免 sync；减小 pageSize；对大账户分页 |
| `MCP 返回空 result` | 工具名/参数错 | 用 `validate_args` 预校验；核对工具名 |
| `ValueError: Unknown format code 'f'` | 用 `f"{bal:,}"` 格式化字符串 | 先 `float(bal)`（见 §6） |
| 下钻 level 返回空 | VidAU 仅同步 ADVERTISER 级 | 降级 ADVERTISER 并注明，不编造（见 enums-reference §10） |
| `status` 过滤无效 | 用了 `ACTIVE` | 改用 `ENABLE`/`DISABLE`（见 enums-reference §8） |

---

## 12. 定时自动同步（每 15 分钟）

独立于「交互式 / on-demand 同步」（§10）的**后台路径**，用于保持已授权账户数据常新；**不影响用户实时查询性能**。

**触发**：宿主平台 cron，cron 表达式 `*/15 * * * *`（按宿主时区），到点执行一次「自动同步」任务。

**执行流程**：
1. `list_advertisers` → 仅保留**已授权（authorized）**账户。
2. 对每个授权账户调用（素材随广告实体一并同步）：
```python
mcp("sync_advertiser_data", {
    "advertiser_id": "<id>",
    "levels": ["CAMPAIGN", "ADGROUP", "AD"]   # 广告系列 / 广告组 / 广告；素材(creative)随 AD 实体一并同步
}, timeout=60)
```
3. 用 `mcp_retry`（仅超时 / 429 / 5xx 退避重试；参数错误不重试）。
4. **账户级超时隔离**：单账户 60s+ 超时不中断其他账户——用线程 / 协程并发 + 每账户独立超时，失败仅记录并跳过，不阻塞整轮。
5. 同步成功后更新上下文 `lastEntitySyncAt`（见 SKILL 上下文模型）。
6. 结果处理：全部失败 → 发告警（不阻断用户正常使用）；部分失败 → 记录失败账户，下一轮重试。

**注意**：
- 只同步已授权账户；未授权 / 受限 / 封禁账户跳过。
- 并发受 §5 全局信号量限制，避免触发 429。
- 与 §10 的「交互式默认不 sync」互不冲突：本路径是定时后台任务，§10 守的是用户实时查询路径。
