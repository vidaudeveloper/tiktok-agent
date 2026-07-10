# 字段与枚举对照（enums-reference）— VidAU MCP 简化值

> **校准基线**：本技能已按真实 **VidAU MCP（`tiktok.vidau.ai`）返回体/接受值** 校准。
> VidAU 对 TikTok 字段做了简化封装，MCP **实际接受的是「简化枚举值」**（如 `LEAD_GEN`、`WEB_CONVERSIONS`、`VIDEO_PLAY_3S`、`CPC`、`CPM`、`OCPM`）。
> 下列 **「本技能使用值（VidAU）」即为 MCP 实际接受的值，可直接传入**，这样调用不会因枚举不匹配而报错。
>
> ⚠️ 注意：这些是 **VidAU 封装层的简化值**，**不是** 官方 TikTok Marketing API 原始枚举（`LEAD_GENERATION` / `CONVERSIONS` / `VIDEO_VIEW` / `BILLING_EVENT_CLICK` 等）。本技能以 VidAU 实际接受值为准。
> 对照表给出「VidAU 值 ↔ 官方 TikTok 值」便于理解，但**传给 MCP 的必须是 VidAU 值**。
> `validate_args`（mcp-client §7）**同时接受 VidAU 简化值与其官方对应值**，确保无论如何都不会因枚举校验报错。

---

## 0. Python 常量（供 `validate_args` 导入 / 内联）

```python
# 本技能使用值 = VidAU MCP 简化值（MCP 实际接受）
OBJECTIVE_TYPES = {
    "REACH", "TRAFFIC", "VIDEO_VIEWS", "LEAD_GEN", "WEB_CONVERSIONS",
    "APP_PROMOTION", "PRODUCT_SALES",
}
# 同时接受官方对应值，避免误报（两种都能过校验、都不会让 MCP 报错）
OBJECTIVE_TYPES |= {"LEAD_GENERATION", "CONVERSIONS", "WEBSITE_CONVERSIONS"}

OPTIMIZATION_GOALS = {
    "CLICK", "CONVERT", "REACH", "VIDEO_PLAY_3S",
}
OPTIMIZATION_GOALS |= {"VIDEO_VIEW", "COMPLETE_VIDEO_VIEW"}

BILLING_EVENTS = {"CPC", "CPM", "OCPM"}
BILLING_EVENTS |= {"BILLING_EVENT_CLICK", "BILLING_EVENT_IMPRESSION", "BILLING_EVENT_CONVERT"}

BID_TYPES = {"BID_TYPE_NO_BID", "BID_TYPE_CUSTOM"}

# promotion_type：VidAU 接受 WEBSITE/APP_ANDROID/APP_IOS，线索表单以 LEAD_GEN 开头
PROMOTION_TYPES = {"WEBSITE", "APP_ANDROID", "APP_IOS"}
PROMOTION_TYPES |= {"LEAD_GENERATION"}
PROMOTION_LEAD_PREFIX = "LEAD_GEN"  # 线索表单变体（如 LEAD_GEN_FORM 等）统一认 LEAD_GEN 前缀
```

---

## 1. Campaign 推广目标（objective_type）— VidAU 简化值

| 本技能使用值（VidAU） | 官方 TikTok 值 | 含义 |
|------------------------|----------------|------|
| `REACH` | `REACH` | 覆盖量 |
| `TRAFFIC` | `TRAFFIC` | 引流 |
| `VIDEO_VIEWS` | `VIDEO_VIEWS` | 视频播放 |
| `LEAD_GEN` | `LEAD_GENERATION` | 线索收集 |
| `WEB_CONVERSIONS` | `CONVERSIONS` / `WEBSITE_CONVERSIONS` | 网站转化 |
| `APP_PROMOTION` | `APP_PROMOTION` | 应用推广 |
| `PRODUCT_SALES` | `PRODUCT_SALES` | 商品销售 |

> 传入 MCP 用 `LEAD_GEN` / `WEB_CONVERSIONS`（VidAU 简化值）；`LEAD_GENERATION` / `CONVERSIONS` 也会通过校验，但优先用 VidAU 值。

---

## 2. 预算模式（budget_mode）— 与官方一致

| 值 | 含义 |
|----|------|
| `BUDGET_MODE_INFINITE` | 不限预算（budget=0） |
| `BUDGET_MODE_DAY` | 日预算 |
| `BUDGET_MODE_TOTAL` | 总预算 |

---

## 3. Adgroup 优化目标（optimization_goal）— VidAU 简化值

| 本技能使用值（VidAU） | 官方 TikTok 值 | 含义 |
|------------------------|----------------|------|
| `CLICK` | `CLICK` | 点击 |
| `CONVERT` | `CONVERT` | 转化 |
| `REACH` | `REACH` | 覆盖 |
| `VIDEO_PLAY_3S` | `VIDEO_VIEW` | 视频观看（3s 播放） |

> 传入 MCP 用 `VIDEO_PLAY_3S`（VidAU 简化值）；`VIDEO_VIEW` 会通过校验但不作为首选。
> 其余官方值（`IMPRESSION`/`INSTALL`/`REGISTRATION`/`SUBSCRIBE`/`FOLLOW`/`LEAD`/`VALUE` 等）如需使用，按 VidAU 实际返回体为准补充。

---

## 4. 计费事件（billing_event）— VidAU 简化值

| 本技能使用值（VidAU） | 官方 TikTok 值 | 含义 |
|------------------------|----------------|------|
| `CPC` | `BILLING_EVENT_CLICK` | 点击计费 |
| `CPM` | `BILLING_EVENT_IMPRESSION` | 展示计费 |
| `OCPM` | `BILLING_EVENT_CONVERT` | 转化计费（oCPM） |

> 传入 MCP 用 `CPC` / `CPM` / `OCPM`（VidAU 简化值）。

---

## 5. 出价类型（bid_type）与出价策略映射

| 出价策略 | bid_type | bid_price | billing_event |
|----------|----------|-----------|---------------|
| 最大投放（Max Delivery / No Bid） | `BID_TYPE_NO_BID` | 0 | 任意 |
| 自定义出价（Custom） | `BID_TYPE_CUSTOM` | > 0 | `CPC` / `CPM` / `OCPM` |
| 成本上限（Cost Cap） | `BID_TYPE_CUSTOM` | 上限价 | `OCPM` |

创建流程「出价」检查项须与上面映射对齐。

---

## 6. 投放类型（promotion_type）— VidAU 简化值

| 值 | 含义 |
|----|------|
| `WEBSITE` | 网站 |
| `APP_ANDROID` | Android 应用 |
| `APP_IOS` | iOS 应用 |
| `LEAD_GEN*`（以 `LEAD_GEN` 开头的线索表单变体） | 线索表单 |

> 线索表单的具体形式由 lead_gen 相关字段配置，VidAU 接受 `LEAD_GEN` 开头的任意值。

---

## 7. 版位（placement_type / placements）

| placement_type | 含义 |
|----------------|------|
| `PLACEMENT_TYPE_AUTOMATIC` | 自动版位 |
| `PLACEMENT_TYPE_NORMAL` | 编辑版位（手动） |

手动版位 `placements` 用 TikTok 版位枚举（如 `PLACEMENT_TIKTOK`、`PLACEMENT_PANGLE` 等）。

---

## 8. 状态枚举（status）— 统一口径，禁止混用

| 实体 | 有效过滤值 | 说明 |
|------|------------|------|
| Campaign | `ENABLE` / `DISABLE` | 运行中 / 暂停。**不要用 `ACTIVE`**（非有效值） |
| Campaign（创建态） | `DRAFT` / `FAILED` / `DELETE` | 草稿 / 失败 / 已删除 |
| Adgroup / Ad | `ENABLE` / `DISABLE` | 同上 |

- `list_advertisers` 的 `status`：`STATUS_ENABLE` / `STATUS_LIMIT`（受限）。

---

## 9. 地区 / 语言等定向

- `targeting.location_ids`：国家/地区 ID（`"100"` = 美国，来自 TikTok region 编码）。
- `gender`：`GENDER_FEMALE` / `GENDER_MALE` / `NONE`。
- `age_groups` / `languages`：数组，可为空。

---

## 10. 报表层级（level）与下钻降级

`get_daily_metrics` 的 `level`：`ADVERTISER` / `CAMPAIGN` / `ADGROUP` / `AD`。

- 经验上 VidAU 对多数账户只可靠返回 `ADVERTISER` 级别；`CAMPAIGN/ADGROUP/AD` 可能为空。
- 调用策略：优先尝试下钻层级；**若返回空，降级为 `ADVERTISER` 聚合并在报告中注明「该账户暂不支持广告级下钻」**，不得编造明细。
