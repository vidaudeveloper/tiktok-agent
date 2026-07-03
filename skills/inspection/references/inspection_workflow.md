# TikTok 广告巡检流程规范

本文件是 Hermes TikTok 广告巡检 Skill 的核心规则文件。执行巡检前必须加载本文件，并以真实数据为依据输出结论。

## 1. 使用场景

本 Skill 用于 TikTok 广告账户、广告系列、广告组、广告、素材维度的日常投放巡检，把巡检结果展示在 `https://tiktok.vidau.ai/Ai` 的 AI助手页面内。

适用场景：

1. 日常投放巡检：自动检查账户、广告系列、广告组、广告、素材的状态、花费、转化、CPA、ROAS、预算和素材表现。
2. 用户主动触发巡检：当用户输入指定触发词时，立即进入巡检流程。
3. 广告创建成功后触发：广告创建成功后，询问用户是否需要开启每日自动巡检数据预警。
4. 异常排查：花费过快、花费过慢、突然不花费、转化下降、转化归零、CPA 升高、ROAS 下降、预算耗尽、素材疲劳。
5. 优化建议：输出排查建议、止损建议、优化建议和后续观察建议。

触发关键词：

```text
查看数据
投放数据
看一下数据
数据分析
数据建议
数据监控
排查数据
观察数据
素材分析
数据情况
监控数据
数据巡检
```

默认范围：只巡检 TikTok，不巡检其他媒体渠道。

默认展示位置：`https://tiktok.vidau.ai/Ai` 的 AI助手页面。

每日自动巡检：本 Skill 只询问是否需要开启，不自动创建定时任务。

执行边界：本 Skill 只输出建议；即使用户确认，也不直接执行预算、出价、暂停、定向、素材、通知或排期修改。

## 2. 输入要求

### 2.1 用户输入

支持自然语言触发，例如：

- 查看数据
- 看一下今天投放数据
- 帮我做数据分析
- 帮我排查数据
- 看看素材有没有问题
- 帮我做 TikTok 数据巡检
- 广告创建成功后帮我监控数据

### 2.2 巡检对象输入

至少需要明确一个巡检范围。若用户未指定，优先使用 Hermes 当前上下文或最近创建成功的广告对象。

| 输入项 | 是否必需 | 说明 |
|---|---:|---|
| advertiser_id / 广告账户 ID | 必需，除非 Hermes 当前页面已有账户上下文 | 定位 TikTok 广告账户 |
| advertiser_name | 可选 | 展示账户名称 |
| campaign_id / campaign_name | 可选 | 指定广告系列时使用 |
| adgroup_id / adgroup_name | 可选 | 指定广告组时使用 |
| ad_id / ad_name | 可选 | 指定广告时使用 |
| creative_id / creative_name | 可选 | 指定素材时使用 |
| date_range | 可选 | 默认今日数据，对比过去 7 日 |
| target_cpa | 条件必需 | 判断 CPA 高于目标时需要 |
| target_roas | 条件必需 | 判断 ROAS 低于目标时需要 |
| target_aov | 条件可选 | 判断有消耗无收入时使用 |
| daily_budget / total_budget | 条件必需 | 判断花费进度和预算耗尽时需要 |

### 2.3 默认时间口径

- 当前日数据：today
- 历史对比：过去 7 日平均
- 小时级对比：过去 7 日同期小时平均
- 突然不花费判断：过去 3.5 小时
- 过去是否有花费：过去 24 小时
- 花费过慢保护：广告已投放超过 3 小时
- 素材生命周期：连续投放大于等于 7 天

### 2.4 缺失输入处理

若无法确认广告账户或对象，返回：

```json
{
  "reply_type": "clarification_required",
  "natural_language_summary": "未识别到需要巡检的 TikTok 广告账户，请先选择广告账户或指定广告对象。",
  "ui_payload": {
    "inspection_summary": {
      "overall_status": "clarification_required"
    }
  }
}
```

## 3. 数据依赖

Skill 必须基于真实广告数据判断，不得编造数据。缺少字段时，对应规则返回 `insufficient_data`。

### 3.1 账户层级数据

| 数据项 | 用途 |
|---|---|
| advertiser_id | 定位账户 |
| advertiser_name | 展示账户名称 |
| account_status | 判断账户是否正常 |
| account_balance | 判断余额不足 |
| payment_status | 判断付款失败 |
| currency | 统一金额口径 |
| timezone | 统一今日和小时数据口径 |
| data_updated_at | 判断数据同步新鲜度 |

### 3.2 广告系列层级数据

| 数据项 | 用途 |
|---|---|
| campaign_id / campaign_name | 定位广告系列 |
| campaign_status | 判断是否开启 |
| objective | 区分转化、App、Shop、直播购物等场景 |
| campaign_budget_type | 判断日预算或总预算 |
| campaign_daily_budget | 判断日预算进度 |
| campaign_total_budget | 判断总预算进度 |
| campaign_cumulative_spend | 判断总预算接近耗尽 |
| campaign_today_spend | 判断花费异常 |
| campaign_conversions | 判断转化异常 |
| campaign_conversion_value | 判断 GMV / ROAS |
| campaign_roas | 判断回报异常 |
| campaign_cpa | 判断成本异常 |

### 3.3 广告组层级数据

| 数据项 | 用途 |
|---|---|
| adgroup_id / adgroup_name | 定位广告组 |
| adgroup_status | 判断是否开启 |
| review_status | 判断审核是否影响投放 |
| bid_strategy | 区分 Maximum Delivery、Cost Cap、ROAS 目标等 |
| bid / cost_cap | 判断不起量或 CPA 异常 |
| targeting_summary | 判断定向是否过窄 |
| adgroup_daily_budget | 判断预算耗尽 |
| adgroup_total_budget | 判断总预算耗尽 |
| current_spend / today_spend | 判断消耗异常 |
| last_3_5h_spend | 判断突然不花费 |
| last_24h_spend | 判断此前是否有花费 |
| clicks | 判断点击正常但无转化 |
| impressions | 判断曝光和素材表现 |
| ctr | 判断点击吸引力 |
| cvr | 判断转化承接能力 |
| cpa | 判断转化成本 |
| roas | 判断广告回报 |
| conversions | 判断转化异常 |
| frequency | 判断素材疲劳 |

### 3.4 广告和素材层级数据

| 数据项 | 用途 |
|---|---|
| ad_id / ad_name | 定位广告 |
| creative_id / creative_name | 定位素材 |
| creative_status | 判断素材是否可投放 |
| creative_review_status | 判断审核不通过 |
| creative_age_days | 判断素材生命周期 |
| impressions | 判断曝光量 |
| clicks | 判断点击量 |
| ctr | 判断素材吸引力 |
| cvr | 判断转化承接 |
| cpa | 判断素材成本 |
| roas | 判断素材回报 |
| last_7d_avg_ctr | 判断 CTR 是否下降 |
| last_7d_avg_cvr | 判断 CVR 是否下降 |
| video_3s_rate | 判断前 3 秒吸引力 |
| video_6s_rate | 判断前 6 秒表现 |
| completion_rate | 判断完播表现 |
| negative_comments | 判断负面反馈，若系统支持 |

### 3.5 转化追踪和后端数据

| 数据项 | 用途 |
|---|---|
| pixel_status | 判断 Web 转化回传 |
| events_api_status | 判断服务器事件回传 |
| mmp_status | 判断 App 回传 |
| conversion_event_name | 判断事件是否被修改 |
| event_dedup_status | 判断去重异常 |
| purchase_value / revenue | 判断 ROAS、VBO、价值优化 |
| value_currency | 判断币种是否一致 |
| attribution_window | 判断归因延迟 |
| landing_page_status | 判断页面慢、断链、支付失败、缺货 |
| stock_status | 判断库存不足 |
| payment_path_status | 判断支付链路 |

## 4. 执行步骤

### Step 1：识别意图

当用户命中触发词，或提出数据、预警、优化、监控、素材分析、广告表现排查相关请求时，进入 TikTok 巡检流程。

### Step 2：确认 TikTok 范围

默认只巡检 TikTok。若用户要求其他平台，返回：

```json
{
  "reply_type": "blocked",
  "natural_language_summary": "当前广告巡检 Skill 仅支持 TikTok，暂不处理其他媒体渠道。"
}
```

### Step 3：确定巡检对象

按以下优先级确定对象：

1. 用户明确指定的广告账户、广告系列、广告组、广告或素材。
2. Hermes 当前页面中已选中的广告对象。
3. 最近创建成功的 TikTok 广告对象。
4. 当前 TikTok 账户下所有开启中的投放对象。

若无法确定广告账户，返回 `clarification_required`。

### Step 4：拉取数据

需要拉取：

1. 今日数据。
2. 当前小时数据。
3. 过去 3.5 小时花费。
4. 过去 24 小时花费。
5. 过去 7 日平均数据。
6. 过去 7 日同期小时平均数据。
7. 预算、余额、状态、审核状态。
8. Pixel、Events API、MMP、落地页、库存、支付链路数据，若 Hermes 支持。

### Step 5：数据标准化

执行判断前统一口径：

```text
CPA = spend / conversions
ROAS = conversion_value / spend
CVR = conversions / clicks
CTR = clicks / impressions
CPC = spend / clicks
CPM = spend / impressions * 1000
budget_progress = today_spend / daily_budget
time_progress = elapsed_hours_today / 24
```

要求：

- 所有金额统一币种。
- 所有时间统一为广告账户时区。
- 分母为 0 时返回 null，不得输出 Infinity。
- 0 值必须区分真实为 0、数据缺失、数据未同步。

### Step 6：执行异常判断

依次判断：

1. 花费异常。
2. 转化异常。
3. CPA 异常。
4. ROAS 异常。
5. 预算耗尽。
6. 素材疲劳。

每条命中的异常必须输出：异常类型、命中规则、当前值、对比值、严重程度、证据、可能原因、优先排查、止损建议、优化建议。

### Step 7：合并同因异常

多个异常同时命中时，不要机械罗列。优先合并为根因问题：

- 花费暴涨 + CPA 升高 + ROAS 下降：低效放量风险。
- 点击正常 + 转化归零 + CVR 下降：点击后承接异常或转化回传异常。
- CTR 下降 + CVR 下降 + CPA 升高：素材疲劳风险。
- 预算耗尽 + CPA / ROAS 达标：预算限制导致机会损失。

### Step 8：按优先级排序

处理顺序：

1. 预算耗尽且效果好：优先提示机会损失和加预算建议。
2. 花费暴涨且 CPA / ROAS 恶化：优先止损建议。
3. 转化归零 / 转化异常：优先查 Pixel、Events API、MMP、落地页、库存、支付链路。
4. CPA 异常：先拆 CPC 和 CVR。
5. ROAS 异常：先拆 CPC、CVR、AOV、value 回传。
6. 素材疲劳：优先新增素材变体、保留稳定素材、替换疲劳素材。

### Step 9：生成结论

结论分为：

- critical：严重异常，需要立即处理。
- high：高风险异常，需要优先排查和止损。
- medium：中度异常，需要优化或观察。
- low：轻度提醒。
- normal：未发现明显异常。
- insufficient_data：数据不足，无法判断。

### Step 10：每日巡检询问

当用户查看数据、触发巡检，或广告创建成功后，追加询问：

```text
是否需要开启每日自动巡检数据预警？开启后可每天检查花费、转化、CPA、ROAS、预算耗尽和素材疲劳，并在 AI助手页面展示异常通知、优化建议和数据分析。
```

本 Skill 只输出询问，不自动创建定时任务。

## 5. 判断规则

如果规则所需字段缺失，对该规则返回 `insufficient_data`。

### 5.1 花费异常

#### A. 花费过快

规则名：`spend_fast`

```text
current_spend > daily_budget * time_progress * 1.3
AND current_spend > last_7d_same_period_avg_spend * 1.5
```

必需字段：`current_spend`, `daily_budget`, `time_progress`, `last_7d_same_period_avg_spend`

判断说明：当前消耗速度高于预算进度，且明显高于历史同期。

严重程度：

- CPA / ROAS 同时恶化：high 或 critical。
- 转化同步增长且 CPA / ROAS 正常：medium。

优先排查：预算是否刚提高、定向是否放宽、出价策略是否切换、是否复制广告组或新增素材、CPM 是否上涨、转化是否同步增长、是否使用 Maximum Delivery。

建议：若 CPA / ROAS 恶化，建议降低预算 20%-30% 或暂停低效素材 / 广告组；若是新广告组学习期，建议等花费达到目标 CPA 的 1.5-2 倍且无转化后再止损；后续扩量建议每次提高预算 20%-50%。

#### B. 花费过慢

规则名：`spend_slow`

```text
current_spend < daily_budget * time_progress * 0.5
AND ad_runtime_hours > 3
AND ad_status_normal == true
```

必需字段：`current_spend`, `daily_budget`, `time_progress`, `ad_runtime_hours`, `ad_status_normal`

优先排查：广告组、广告、素材是否开启并审核通过；预算是否过低；Cost Cap 出价是否过低；定向是否过窄；素材 CTR 是否偏低；余额、付款、政策、Pixel / MMP 是否异常。

建议：若连续 3 小时以上无花费且审核、预算、余额正常，建议提高 Cost Cap 出价约 20%；定向过窄时建议放宽兴趣、人群包、年龄或版位；预算过低时建议提高到目标 CPA 的 5-10 倍以上；补充 3-5 条差异明显素材。

#### C. 突然不花费

规则名：`no_spend`

```text
last_3_5h_spend == 0
AND last_24h_spend > 0
AND adgroup_status == enabled
```

必需字段：`last_3_5h_spend`, `last_24h_spend`, `adgroup_status`

优先排查：账户余额、预算耗尽、审核状态、Campaign / Ad Group / Ad / Creative 状态、出价、定向、政策限制。

#### D. 花费暴涨

规则名：`spend_spike`

```text
today_spend >= daily_budget * 0.25
```

必需字段：`today_spend`, `daily_budget`

说明：如果发生在较早时间段，可能代表预算消耗过快。必须结合 CPA / ROAS 判断是有效放量还是低效烧钱。

### 5.2 转化异常

#### A. 转化数下降

规则名：`conversion_drop`

```text
today_conversions < last_7d_avg_conversions * 0.5
AND today_spend >= last_7d_avg_spend * 0.7
```

必需字段：`today_conversions`, `last_7d_avg_conversions`, `today_spend`, `last_7d_avg_spend`

优先排查：Pixel、Events API、MMP、转化事件变更、事件去重、落地页速度、断链、支付失败、库存、CVR 断崖式下跌。

建议：若转化归零且花费达到目标 CPA 的 1.5-2 倍，建议暂停该广告组或降低预算；若多个广告组同时下降，先查回传、页面、支付、库存、MMP，不要先调广告。

#### B. 转化归零

规则名：`zero_conversion`

```text
today_conversions == 0
AND today_spend >= target_cpa * 1.5
```

必需字段：`today_conversions`, `today_spend`, `target_cpa`

优先排查：Pixel / Events API / MMP、落地页、支付链路、素材是否吸引错误人群。

#### C. 点击正常但无转化

规则名：`clicks_normal_no_conversion`

```text
today_clicks >= last_7d_avg_clicks * 0.8
AND today_conversions == 0
```

必需字段：`today_clicks`, `last_7d_avg_clicks`, `today_conversions`

优先排查：素材是否只吸引点击不筛选人群、落地页与广告承诺是否一致、价格 / 运费 / 优惠 / 支付方式是否一致、TikTok 到支付的完整链路。

建议：花费达到目标 CPA 的 2 倍仍无转化时，建议暂停素材或广告组；CTR 很高但 CVR 极低时，优先替换该素材；多个素材都无转化时，暂停扩量并修复页面或事件回传。

### 5.3 CPA 异常

#### A. CPA 高于目标

规则名：`cpa_high`

```text
current_cpa > target_cpa * 1.3
AND conversions >= 3
```

必需字段：`current_cpa`, `target_cpa`, `conversions`

严重条件：

```text
current_cpa > target_cpa * 1.5
AND conversions >= 3
```

止损条件：

```text
current_cpa > target_cpa * 2
```

诊断公式：

```text
CPA = CPC / CVR
```

优先排查：CPC 是否变贵、CVR 是否变差、CPM 是否上涨、CTR 是否下降、落地页 / 商品 / 价格 / 支付 / 事件回传是否异常、是否刚调整预算 / 出价 / 定向 / 素材。

建议：CPA 大于目标 1.5 倍且转化数大于等于 3，建议降低预算 20%-30% 或暂停低效广告组；CPA 大于目标 2 倍时建议暂停，除非仍处于学习期且样本不足；Cost Cap 不建议因 CPA 变差就立刻大幅降低 bid。

#### B. CPA 相比历史升高

规则名：`cpa_history_high`

```text
current_cpa > last_7d_avg_cpa * 1.5
AND conversions >= 3
```

必需字段：`current_cpa`, `last_7d_avg_cpa`, `conversions`

#### C. 无转化 CPA 风险

规则名：`cpa_no_conversion_risk`

```text
current_spend >= target_cpa * 1.5
AND conversions == 0
```

必需字段：`current_spend`, `target_cpa`, `conversions`

严重条件：

```text
current_spend >= target_cpa * 2
AND conversions == 0
```

#### D. CPA 升高但 ROAS 未下降

规则名：`cpa_up_roas_ok_note`

这是诊断说明，不应直接判定为负向异常。若 ROAS 达标，不建议只因 CPA 升高暂停。

优先排查：AOV 是否上升、购买数减少但单笔金额变高、优化目标是否为 Purchase / Value / ROAS、是否存在归因延迟。

### 5.4 ROAS 异常

#### A. ROAS 低于目标

规则名：`roas_low`

```text
current_roas < target_roas * 0.7
AND current_spend >= daily_budget * 0.3
```

必需字段：`current_roas`, `target_roas`, `current_spend`, `daily_budget`

严重条件：

```text
current_roas < target_roas * 0.5
AND sample_valid == true
```

诊断公式：

```text
ROAS = conversion_value / spend = CVR * AOV / CPC
```

优先排查：CPC 是否变贵、CVR 是否变差、AOV 是否下降、折扣 / 运费 / 库存 / 优惠券是否变化、Purchase value 和 currency 是否正常、是否吸引低购买力人群。

建议：ROAS 低于目标 50% 且样本有效时，建议降低预算或暂停；连续 2 天低于目标 70%，建议减少预算 20%-50%，并将预算转移到高 ROAS 组；有花费无 GMV 时，先查 Purchase 事件、value 参数、支付链路和库存。

#### B. ROAS 相比历史下降

规则名：`roas_history_drop`

```text
current_roas < last_7d_avg_roas * 0.6
AND current_spend >= last_7d_avg_spend * 0.7
```

必需字段：`current_roas`, `last_7d_avg_roas`, `current_spend`, `last_7d_avg_spend`

#### C. 有消耗无收入

规则名：`spend_no_revenue`

```text
current_spend >= target_aov * 0.5
AND conversion_value == 0
```

必需字段：`current_spend`, `target_aov`, `conversion_value`

优先排查：Purchase value 是否正常回传、currency 是否一致、支付链路、库存。

#### D. ROAS 下降但 CPA 正常

规则名：`roas_down_cpa_ok_note`

这是诊断说明，不一定要立即暂停。优先检查 AOV 是否下降、低价商品成交占比是否提升、优惠是否过大、Purchase value 是否漏传或传错、是否只优化购买数没有优化购买价值。

### 5.5 预算耗尽

#### A. 日预算即将耗尽

规则名：`budget_near_exhausted`

```text
today_spend / daily_budget >= 0.70
```

必需字段：`today_spend`, `daily_budget`

#### B. 日预算已耗尽

规则名：`budget_exhausted`

```text
today_spend / daily_budget >= 0.95
```

必需字段：`today_spend`, `daily_budget`

判断：若 CPA / ROAS 达标，这是机会损失；若 CPA / ROAS 不达标，不建议加预算。

#### C. 总预算即将耗尽

规则名：`total_budget_near_exhausted`

```text
cumulative_spend / total_budget >= 0.90
```

必需字段：`cumulative_spend`, `total_budget`

#### D. 预算耗尽导致停止投放

规则名：`budget_stop_delivery`

```text
remaining_budget == 0
```

必需字段：`remaining_budget`

优先排查：预算耗尽时间、耗尽前 CPA / ROAS、预算是否太低、Campaign Budget 和 Ad Group Budget 分配、活动周期是否未结束但预算已接近耗尽、低效广告组是否抢预算。

建议：效果不达标时不要加预算；效果达标且预算过早耗尽时，建议逐步提高预算；活动型 Campaign 建议按预热期、爆发期、返场期拆预算；电商广告追加预算前确认库存、物流、价格、优惠券。

### 5.6 素材疲劳

#### A. CTR 明显下降

规则名：`creative_ctr_drop`

```text
current_ctr < last_7d_avg_ctr * 0.7
```

必需字段：`current_ctr`, `last_7d_avg_ctr`

#### B. CVR 明显下降

规则名：`creative_cvr_drop`

```text
current_cvr < last_7d_avg_cvr * 0.7
```

必需字段：`current_cvr`, `last_7d_avg_cvr`

#### C. 素材生命周期过长

规则名：`creative_lifecycle_fatigue`

```text
creative_age_days >= 7
AND ctr_drop_rate >= 0.30
```

必需字段：`creative_age_days`, `ctr_drop_rate`

优先排查：素材投放天数、曝光量、频次、CTR 走势、3 秒播放率、6 秒播放率、完播率、评论负面反馈、同一素材是否在多个广告组重复投放。

建议：CTR 连续下滑且 CPA 升高时，建议降低预算或暂停疲劳素材；频次升高且新用户触达下降时，建议补充新素材；个别素材疲劳时不要关整个广告组，优先替换或新增素材；保留高转化素材结构，替换开头、人物、场景、CTA、前三秒文案。

## 6. 风险边界

### 6.1 只输出建议，不执行操作

本 Skill 是巡检、诊断和建议 Skill。即使用户确认，也只能输出建议，不得直接执行以下操作：

1. 暂停广告账户。
2. 暂停广告系列。
3. 暂停广告组。
4. 暂停广告。
5. 删除广告对象。
6. 修改预算。
7. 修改出价。
8. 修改定向。
9. 修改投放目标。
10. 修改转化事件。
11. 修改素材。
12. 自动复制广告组。
13. 自动新建广告系列。
14. 自动提交审核。
15. 发送外部通知。
16. 创建每日定时巡检任务。

允许输出：

- 建议降低预算。
- 建议暂停低效广告组。
- 建议替换素材。
- 建议提高出价。
- 建议放宽定向。
- 建议检查 Pixel / Events API / MMP。
- 建议优化落地页。
- 建议开启每日巡检，但不得自动创建。

所有建议必须带上：

```json
{
  "execution_mode": "recommendation_only",
  "execution_allowed_in_this_skill": false
}
```

### 6.2 不得编造数据

缺少数据时返回：

```text
当前缺少【字段名】数据，无法判断【异常类型】。
```

不得输出虚假的 CPA、ROAS、转化数、预算进度、素材疲劳结论或未经数据支持的确定性判断。

### 6.3 不得过度承诺效果

不得使用：

- 一定能降低 CPA。
- 一定能提高 ROAS。
- 保证恢复消耗。
- 保证转化提升。

应使用：

- 建议优先排查。
- 可能原因是。
- 建议观察 24-48 小时。
- 如 CPA / ROAS 持续恶化，再考虑止损。
- 需要结合更多数据确认。

### 6.4 学习期保护

对新广告组或刚调整预算、出价、定向、素材的广告组，不应过早判定失败。

规则：

1. 样本不足时输出观察中。
2. 未达到目标 CPA 的 1.5-2 倍花费，不建议立即暂停。
3. 学习期只给轻度提醒和观察建议。
4. 除非触发严重止损条件，否则避免建议大幅调整。

### 6.5 回传异常保护

转化或 ROAS 异常时，优先判断回传是否异常。存在 Pixel、Events API、MMP、Purchase value、currency、归因窗口、落地页、支付、库存异常时，不应直接判定广告无效。

## 7. 输出格式

### 7.1 Hermes JSON 输出

```json
{
  "reply_type": "inspection_result | alert_required | clarification_required | sync_required | insufficient_data | blocked",
  "natural_language_summary": "不超过100个中文字符的摘要",
  "slot_updates": {
    "platform": "tiktok",
    "inspection_scope": "account | campaign | adgroup | ad | creative",
    "date_range": "today"
  },
  "ui_actions": [],
  "ui_payload": {
    "inspection_summary": {
      "platform": "tiktok",
      "site": "https://tiktok.vidau.ai/Ai",
      "inspection_time": "YYYY-MM-DD HH:mm:ss",
      "timezone": "account_timezone",
      "data_updated_at": "YYYY-MM-DD HH:mm:ss",
      "overall_status": "normal | warning | critical | insufficient_data",
      "risk_level": "none | low | medium | high | critical",
      "total_anomalies": 0,
      "critical_anomalies": 0,
      "warning_anomalies": 0,
      "main_conclusion": ""
    },
    "scope": {
      "advertiser_id": "",
      "advertiser_name": "",
      "campaign_id": "",
      "campaign_name": "",
      "adgroup_id": "",
      "adgroup_name": "",
      "ad_id": "",
      "ad_name": "",
      "creative_id": "",
      "creative_name": ""
    },
    "metric_cards": [
      {
        "metric": "spend | conversions | cpa | roas | ctr | cvr | budget_progress",
        "label": "中文指标名",
        "current_value": 0,
        "benchmark_value": 0,
        "unit": "",
        "status": "normal | warning | critical | insufficient_data"
      }
    ],
    "anomaly_cards": [
      {
        "anomaly_type": "spend_fast | spend_slow | no_spend | spend_spike | conversion_drop | zero_conversion | clicks_normal_no_conversion | cpa_high | cpa_history_high | cpa_no_conversion_risk | roas_low | roas_history_drop | spend_no_revenue | budget_near_exhausted | budget_exhausted | total_budget_near_exhausted | budget_stop_delivery | creative_ctr_drop | creative_cvr_drop | creative_lifecycle_fatigue | insufficient_data",
        "object_level": "account | campaign | adgroup | ad | creative",
        "object_id": "string or unknown",
        "object_name": "string or unknown",
        "severity": "info | low | medium | high | critical",
        "evidence": [],
        "rule_matched": "exact rule name",
        "possible_causes": [],
        "recommended_checks": [],
        "suggested_actions": [],
        "execution_mode": "recommendation_only",
        "execution_allowed_in_this_skill": false
      }
    ],
    "priority_order": [
      {
        "rank": 1,
        "issue": "",
        "reason": "",
        "suggested_next_step": ""
      }
    ],
    "chart_specs": [
      {
        "chart_type": "line | bar | progress | table",
        "title": "",
        "metrics": [],
        "x_axis": "",
        "y_axis": "",
        "data": []
      }
    ],
    "recommended_actions": [
      {
        "title": "",
        "reason": "",
        "action_type": "check | optimize | stop_loss | observe",
        "execution_mode": "recommendation_only",
        "execution_allowed_in_this_skill": false
      }
    ],
    "daily_monitoring_inquiry": {
      "should_ask_user": true,
      "message": "是否需要开启每日自动巡检数据预警？开启后可每天检查花费、转化、CPA、ROAS、预算耗尽和素材疲劳，并在 AI助手页面展示异常通知、优化建议和数据分析。",
      "schedule_creation_allowed_in_this_skill": false
    }
  },
  "tool_followups": [],
  "risk_warnings": [
    "本 Skill 仅输出建议，不会自动修改预算、出价、定向、素材或广告状态。"
  ]
}
```

### 7.2 AI助手页面展示模块

页面建议展示：

1. 巡检摘要卡片：对象、时间、异常数、严重异常数、风险等级。
2. 核心指标卡片：花费、转化、CPA、ROAS、CTR、CVR、预算消耗进度。
3. 异常预警列表：异常类型、风险等级、命中规则、当前值、对比值、影响对象。
4. 原因诊断：优先排查项、可能原因、数据证据。
5. 优化建议：止损建议、优化建议、后续观察建议。
6. 图表数据：花费趋势、转化趋势、CPA 趋势、ROAS 趋势、素材 CTR / CVR 对比、预算消耗进度。

### 7.3 自然语言摘要

```text
本次已完成 TikTok 广告巡检，共发现 X 个异常，其中 X 个严重异常、X 个中度异常。最需要优先处理的是【XXX】，原因是【XXX】。建议优先【XXX】。本 Skill 仅输出建议，不会自动修改广告设置。
```

## 8. 异常处理

### 8.1 数据缺失

返回：

```json
{
  "reply_type": "insufficient_data",
  "natural_language_summary": "当前缺少关键数据，无法完成部分巡检判断。",
  "ui_payload": {
    "missing_data": [
      {
        "field": "",
        "impact": "",
        "fallback": ""
      }
    ]
  }
}
```

示例：缺少目标 CPA 时，不判断“CPA 高于目标”，但可判断“CPA 高于过去 7 日均值”。

### 8.2 历史数据不足

| 情况 | 处理方式 |
|---|---|
| 历史数据大于等于 3 天但少于 7 天 | 使用已有天数平均值，并标记样本较少 |
| 历史数据少于 3 天 | 不做历史均值判断，只做预算、状态、花费、转化基础判断 |
| 新广告组投放少于 3 小时 | 不判断花费过慢 |
| 新素材投放少于 7 天 | 不判断素材生命周期疲劳 |

### 8.3 转化延迟

若可能存在归因延迟，返回观察建议：

```text
当前转化数据可能存在归因延迟，建议结合后续 24 小时数据继续观察，不建议仅凭当前小时数据立即暂停广告。
```

适用：高客单价产品、App 深度事件、MMP 回传延迟、Web Purchase 延迟、直播购物或 TikTok Shop 成交延迟。

### 8.4 预算字段缺失

缺少日预算或总预算时：

- 不判断预算耗尽。
- 不判断花费是否超过预算进度。
- 仍可判断今日花费与历史花费变化。

### 8.5 目标 CPA / ROAS 缺失

- 缺少目标 CPA：不判断 `cpa_high`，可判断 `cpa_history_high`。
- 缺少目标 ROAS：不判断 `roas_low`，可判断 `roas_history_drop`。

### 8.6 状态异常

如果广告对象状态不是开启，优先提示状态问题，不直接判断投放异常。

需要检查：Campaign 状态、Ad Group 状态、Ad 状态、Creative 审核状态、账户状态、余额状态、付款状态。

### 8.7 API 或平台请求失败

返回：

```json
{
  "reply_type": "sync_required",
  "natural_language_summary": "广告数据请求失败，暂时无法完成巡检。",
  "ui_payload": {
    "inspection_summary": {
      "overall_status": "insufficient_data",
      "error_type": "api_error",
      "suggested_action": "请稍后重试，或检查广告账户授权、接口权限、数据同步状态。"
    }
  }
}
```

不得输出未经数据支持的异常结论。

### 8.8 多异常同时命中

优先合并同因异常，并按优先级排序：

1. 预算耗尽且效果好。
2. 花费暴涨且 CPA / ROAS 恶化。
3. 转化归零 / 转化异常。
4. CPA 异常。
5. ROAS 异常。
6. 素材疲劳。

### 8.9 页面展示失败

若 AI助手页面组件展示失败，返回简化 Markdown 文本：

```text
巡检完成，但页面组件展示失败。以下为文本版巡检结果：
1. 异常摘要：XXX
2. 命中规则：XXX
3. 当前数据：XXX
4. 建议处理：XXX
```
