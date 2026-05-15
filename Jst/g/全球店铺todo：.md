### 🟢 P3（可选优化）

#### P3-1: `CreateGlobalItem` 中 `Thread.Sleep(3000)` 硬编码等待时间

- **文件**：`G\API\Platform\Shopping\Global\Shopee\Item\Cnsc\ShopItemCnscPlfService.Publish.cs` 第 79 行
- **问题**：父类使用 `Thread.Sleep(5000)` 并有注释说明 Shopee 要求等待 5 秒。CNSC 版本改为 3 秒但没有说明原因。建议添加注释说明为何全球商品 API 等待时间不同，或统一为 5 秒。
- **修改方案**：添加注释说明等待时间差异的原因，或与父类保持一致。

---

#### P3-2: `SyncGlobalModels` 方法体为空（TODO 注释）

- **文件**：`G\API\Platform\Shopping\Global\Shopee\Item\Cnsc\ShopItemCnscPlfService.Publish.cs` 第 195-204 行
- **问题**：方法内只有 TODO 注释和日志，实际不执行任何同步逻辑。更新全球商品时变体不会被同步。
- **修改方案**：如果当前版本不需要变体同步，建议在方法注释中明确标注"当前版本不支持变体同步"，并在调用处添加日志提示。

---

#### P3-3: `BuildShopItemTree` 中 CNSC 子站点信息使用匿名类型

- **文件**：`G\Item\ShopItem\ShopItemService.cs` 第 2338-2343 行
- **问题**：`cnscSubShops` 使用匿名类型 `new { s.shop_id, s.region, s.name, s.plf_shop_id }`，虽然序列化为 JSON 没问题，但如果后续需要在其他地方复用这个结构，建议定义一个简单的 DTO 类。
- **修改方案**：当前可接受，后续如有复用需求再提取为 DTO。