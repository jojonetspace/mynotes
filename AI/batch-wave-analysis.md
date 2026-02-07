---
inclusion: fileMatch
fileMatchPattern: "**/WaveCreateFacade.cs,**/WaveCreateRuleMatcherPartial.cs,**/WaveDataLoader.cs,**/WaveFacade.cs,**/WaveCreateRuleFactory.cs,**/WaveCreateRuleBase*.cs,**/NewWaveRule.cshtml"
---

# 批量波次生成条件有效性分析

> 分析日期: 2026-02-07
> 分析范围: "选择策略生成"按钮（勾选多条策略后点击生成），即 `IsBatchCreateWave==true` 路径
> 状态: 已发现6个问题待修复

## 调用链

```
前端 oneKeyGen('MultiSelect')
  → POST /wms/wave/OneKeyGenerateWave {WarehouseId, WareRuleIds}
  → WaveController.OneKeyGenerateWave (WaveController.cs Line 73)
  → WaveFacade.OneKeyGenerateWave (WaveFacade.cs Line 452)
    → 从DB加载所有规则，按WareRuleIds过滤
    → OptimizedCreateWaves (Line 493, IsBatchCreateWave==true)
      → WaveCreateFacade.CreateWaves(rules)
        → WaveDataLoader.LoadInouts(rules)
          → MergeWaveCreateRules(rules) → 合并为宽松规则
          → InoutDA.Get(mergedCondition) → SQL查询
        → foreach rule:
          → DataLoader.GetWaveCreateRule(rule, true)
          → FilterInoutsByRule(waveCreateRule, inoutList)
            → waveCreateRule.IsInoutMatch(io) → 29个Check方法
            → IsInoutMatch(io, orderList) → MopId检查
            → IsInoutMatch(io, skuLabelDict) → 商品标签检查
          → waveCreateRule.Execute(matchingInouts, context)
```

## 两层过滤机制

- 第1层 SQL查询: 使用 MergeWaveCreateRules 合并后的条件，仅保留 WarehouseId/PrintStatus/MinSkuCount/MinSkuQty/WaveType
- 第2层 内存过滤: IsInoutMatch 使用每个原始规则的完整条件

## 关键文件清单

| 文件 | 职责 |
|------|------|
| `Jst.Global.Web/Areas/Wms/Views/Wave/NewWaveRule.cshtml` | 前端条件配置页面 |
| `Jst.Global.Web/Areas/Wms/Controllers/WaveController.cs` | 控制器入口 |
| `Jst.Global.Facade/Wms/WaveFacade.cs` | OneKeyGenerateWave/OptimizedCreateWaves |
| `Jst.Global.Facade/Wms/WaveCreateRuleImpl/WaveCreateFacade.cs` | CreateWaves/FilterInoutsByRule |
| `Jst.Global.Facade/Wms/WaveCreateRuleImpl/WaveCreateRuleMatcherPartial.cs` | IsInoutMatch + 所有Check*方法 |
| `Jst.Global.Facade/Wms/WaveCreateRuleImpl/WaveCreateRuleFactory.cs` | GetWaveCreateRuleByRule/GetWaveCreateCondition |
| `Jst.Global.Facade/Wms/WaveCreateRuleImpl/WaveDataLoader.cs` | LoadInouts/MergeWaveCreateRules |
| `Jst.Global.BLL/Wms/WaveCreateRuleData.cs` | ExtendData字段定义 |
| `Jst.Global.BLL/Wms/WaveCreateCondition.cs` | 条件字段定义 |
| `Jst.Global.Wms/DBAccess/InoutDA.cs` | InitParameters SQL条件构建 |

## 条件有效性对比表

| #   | 前端条件                     | DB字段                                    | SQL层             | IsInoutMatch层                                 | 结论       |
| --- | ------------------------ | --------------------------------------- | ---------------- | --------------------------------------------- | -------- |
| 1   | 平台(platform)             | Plateforms                              | ✅ InitParameters | ✅ CheckPlatform                               | ✅ 正常     |
| 2   | 店铺(shop)                 | Shops                                   | ✅ InitParameters | ✅ CheckShop                                   | ✅ 正常     |
| 3   | 货主(linkCompany)          | LinkCompanys                            | ✅ InitParameters | ✅ CheckLinkCompany                            | ✅ 正常     |
| 4   | 货主店铺(linkShop)           | LinkShops                               | ✅ InitParameters | ✅ CheckLinkShopId                             | ✅ 正常     |
| 5   | 快递公司(logistic)           | LogisticsCompanyCodes                   | ✅ InitParameters | ✅ CheckLogistics                              | ✅ 正常     |
| 6   | 地区(region)               | ExtendData.Region                       | ❌                | ✅ CheckRegion                                 | ✅ 正常     |
| 7   | 代收货款(Cods)               | Cods                                    | ✅ InitParameters | 🔴 无CheckCods                                 | 🔴 P4    |
| 8   | 订单应付金额(Amount)           | IsSetAmount/MinAmount/MaxAmount         | ✅ InitParameters | 🔴 无CheckAmount                               | 🔴 P5    |
| 9   | 支付时间(payTime)            | ExtendData.PayTimeBegin/End             | ✅ InitParameters | ✅ CheckPayTime                                | ✅ 正常     |
| 10  | 审单时间(auditTime)          | ExtendData.AuditTimeBegin/End           | ✅ InitParameters | ✅ CheckAuditTime                              | ✅ 正常     |
| 11  | 预发货时间(presendTime)       | ExtendData.PresendTimeBegin/End         | ✅ InitParameters | ✅ CheckPresendTime                            | ✅ 正常     |
| 12  | 剩余收货时间                   | ExtendData.Min/MaxRemainderDeliveryTime | ✅ InitParameters | ✅ CheckRemainderDeliveryTime                  | ✅ 正常     |
| 13  | 限定标签(includeLabels)      | IncludeLabels                           | ✅ InitParameters | ✅ CheckLabel                                  | ✅ 正常     |
| 14  | 排除标签(excludeLabels)      | ExcludeLabels                           | ✅ InitParameters | ✅ CheckExcludeLabel                           | ✅ 正常     |
| 15  | 买家留言(buyerMessage)       | BuyerMessage                            | ✅ InitParameters | ✅ CheckBuyerMessage                           | ✅ 正常     |
| 16  | 卖家备注(sellerRemark)       | ExtendData.SellerRemark                 | ✅ InitParameters | ✅ CheckSellerRemark                           | ✅ 正常     |
| 17  | 多渠道订单(mopId)             | ExtendData.MopId                        | ✅ InitParameters | ✅ IsInoutMatch(io,orders) via NeedOrder       | ✅ 正常     |
| 18  | 打印状态(printStatus)        | PrintStatus                             | ✅ InitParameters | ✅ CheckPrintExpress                           | ✅ 正常     |
| 19  | 发货分拣码(firstSortCode)     | ExtendData.FirstSortCode                | ❌                | ✅ CheckFirstSortCode                          | ✅ 正常     |
| 20  | 商品分类(categoryId)         | CategoryId                              | ✅ InitParameters | 🔴 无CheckCategoryId                           | 🔴 P6    |
| 21  | 限定商品(skuId)              | SkuId/SkuIds                            | ✅ InitParameters | ✅ CheckConditionSkuIds+CheckSkuIdWithPriority | ✅ 正常     |
| 22  | 排除商品(excludeSkuId)       | ExcludeSkuId                            | ✅ InitParameters | ✅ CheckExcludeSkuId                           | ✅ 正常     |
| 23  | 限定商品(defineSkuId)        | DefineSkuId                             | ✅ InitParameters | ✅ CheckDefineSkuId                            | ✅ 正常     |
| 24  | 订单商品数量(skuQty)           | MinSkuQty/MaxSkuQty                     | ✅ InitParameters | ✅ CheckSkuQty                                 | ✅ 正常     |
| 25  | 订单款式数量(itemCount)        | ExtendData.Min/MaxOrderItemCount        | ❌                | ✅ CheckItemCount                              | ✅ 正常     |
| 26  | 订单商品种类(skuKindQty)       | MinSkuKindQty/MaxSkuKindQty             | ✅ InitParameters | ❌ 无Check (SQL层已过滤)                            | ⚠️ 仅SQL层 |
| 27  | 波次款式数量(waveItemCount)    | ExtendData.Min/MaxWaveItemCount         | ❌                | ❌ (波次级别,Execute阶段)                            | ⚠️ 波次级别  |
| 28  | 波次商品种类(waveSkuKindCount) | MinSkuCount/MaxSkuCount                 | ❌                | ❌ (波次级别,Execute阶段)                            | ⚠️ 波次级别  |
| 29  | 商品重量(skuWeight)          | MinSkuWeight/MaxSkuWeight               | ✅ InitParameters | ✅ CheckSkuWeight                              | ✅ 正常     |
| 30  | 商品体积(skuCube)            | MinSkuCube/MaxSkuCube                   | ✅ InitParameters | ✅ CheckSkuCube                                | ✅ 正常     |
| 31  | 到期天数(expireDays)         | MinExpireDays/MaxExpireDays             | ❌                | 🔴 无CheckExpireDays                           | 🔴 P1    |
| 32  | 限定商品标签(includeSkuLabels) | ExtendData.IncludeSkuLabels             | ✅ InitParameters | ✅ IsInoutMatch(io,skuLabels) via NeedSkuLabel | ✅ 正常     |
| 33  | 排除商品标签(excludeSkuLabels) | ExtendData.ExcludeSkuLabels             | ✅ InitParameters | ✅ IsInoutMatch(io,skuLabels) via NeedSkuLabel | ✅ 正常     |
| 34  | 限定仓位区域(bins)             | Bins                                    | ❌                | 🔴 无CheckBins                                 | 🔴 P2    |
| 35  | 排除仓位区域(excludeBins)      | ExcludeBins                             | ❌                | 🔴 无CheckExcludeBins                          | 🔴 P3    |


## 发现的问题汇总

### P1 🔴 到期天数(expireDays) — 完全无效

- 字段: `MinExpireDays` / `MaxExpireDays`
- SQL层: `InitParameters` 中无对应过滤条件
- 内存层: `WaveCreateRuleMatcherPartial.cs` 中无 `CheckExpireDays` 方法
- 影响: 无论单规则还是多规则，该条件完全不生效

### P2 🔴 限定仓位区域(bins) — 完全无效

- 字段: `Bins`
- SQL层: `InitParameters` 中无对应过滤条件
- 内存层: 无 `CheckBins` 方法
- 影响: 无论单规则还是多规则，该条件完全不生效

### P3 🔴 排除仓位区域(excludeBins) — 完全无效

- 字段: `ExcludeBins`
- SQL层: `InitParameters` 中无对应过滤条件
- 内存层: 无 `CheckExcludeBins` 方法
- 影响: 无论单规则还是多规则，该条件完全不生效

### P4 🔴 代收货款(Cods) — 多规则时失效

- 字段: `Cods`
- SQL层: `InitParameters` 中有过滤（单规则有效）
- 内存层: 无 `CheckCods` 方法
- 失效原因: `MergeWaveCreateRules` 合并时丢失 `Cods` 条件；内存层无对应Check方法兜底
- 影响: 勾选多条规则时，Cods条件完全失效

### P5 🔴 订单应付金额(Amount) — 多规则时失效

- 字段: `IsSetAmount` / `MinAmount` / `MaxAmount`
- SQL层: `InitParameters` 中有过滤（单规则有效）
- 内存层: 无 `CheckAmount` 方法
- 失效原因: `MergeWaveCreateRules` 合并时丢失金额条件；内存层无对应Check方法兜底
- 影响: 勾选多条规则时，金额条件完全失效

### P6 🔴 商品分类(CategoryId) — 多规则时失效

- 字段: `CategoryId`
- SQL层: `InitParameters` 中有过滤（单规则有效）
- 内存层: 无 `CheckCategoryId` 方法
- 失效原因: `MergeWaveCreateRules` 合并时丢失分类条件；内存层无对应Check方法兜底
- 影响: 勾选多条规则时，分类条件完全失效

## MergeWaveCreateRules 关键分析

位置: `WaveDataLoader.cs` → `MergeWaveCreateRules` 方法

### 单规则 vs 多规则行为差异

- 单规则(rules.Count==1): 直接使用原始规则的完整条件 → SQL查询精确
- 多规则(rules.Count>1): 合并为"最宽松"条件，仅保留以下字段:
  - `WarehouseId` (取第一条)
  - `PrintStatus` (取并集)
  - `MinSkuCount` (取最小值)
  - `MinSkuQty` (取最小值)
  - `WaveType` (取第一条)
  - 其他所有条件字段被丢弃

### 设计意图与问题

设计意图: 多规则时用宽松SQL查询加载更多数据，再由 `IsInoutMatch` 逐规则精确过滤。

问题: P4/P5/P6 的条件在 SQL 层被 `MergeWaveCreateRules` 丢弃后，`IsInoutMatch` 中又没有对应的 Check 方法来兜底，导致这些条件在多规则场景下完全失效。

## 修复建议

### 方向A: 补充缺失的Check方法（推荐）

在 `WaveCreateRuleMatcherPartial.cs` 的 `IsInoutMatch` 中补充6个缺失的Check方法:
- `CheckExpireDays` — 检查到期天数范围
- `CheckBins` — 检查限定仓位区域
- `CheckExcludeBins` — 检查排除仓位区域
- `CheckCods` — 检查代收货款
- `CheckAmount` — 检查订单应付金额范围
- `CheckCategoryId` — 检查商品分类

优点: 改动集中在一个文件，不影响SQL层逻辑，与现有Check方法模式一致
缺点: SQL层仍然会加载多余数据（P1/P2/P3在SQL层也无过滤）

### 方向B: 同时补充SQL层和Check方法

除方向A外，还在 `InoutDA.InitParameters` 中补充 P1/P2/P3 的SQL过滤条件。

优点: 双层过滤都完整，性能更优
缺点: 改动范围更大，需要同时修改 `MergeWaveCreateRules` 的合并逻辑

### 方向C: 重构MergeWaveCreateRules

修改 `MergeWaveCreateRules` 使其保留更多条件字段（取并集而非丢弃）。

优点: 从根源解决多规则合并丢失条件的问题
缺点: 合并逻辑复杂度高，可能影响SQL查询性能，风险较大
