
一、设计目标
将 SKU 表格从笛卡尔积驱动改为数据驱动 + 维度辅助模式：

SKU 列表是唯一数据源，不再被 sales_properties 的笛卡尔积覆盖
属性面板变为"辅助工具"——帮助用户批量操作 SKU，而非控制 SKU 结构
切换分类、变更维度时保留已有 SKU 数据
二、核心概念变化
维度	当前设计	新设计
SKU 列表来源	sales_properties 笛卡尔积生成	数据库/用户操作累积
sales_properties 的角色	SKU 的"基因"（决定 SKU 存亡）	SKU 的"标签工具"（辅助操作）
属性面板的勾选	触发笛卡尔积重建	触发"展开/收起 SKU 行"交互
切换分类	可能重建所有 SKU	只影响属性面板，不动 SKU 数据
发布校验	无（前端保证完整笛卡尔积）	发布前校验是否为完整笛卡尔积
三、产品交互设计
3.1 属性面板行为变化
当前行为：勾选颜色"红色"→ 自动生成"红色×S、红色×M、红色×L"三行 SKU

新行为：勾选颜色"红色"→ 弹出操作面板：


┌─────────────────────────────────────────────┐
│  选中颜色：红色                               │
│                                              │
│  [展开为新 SKU 行]   [应用到已有 SKU]  [取消]  │
│                                              │
│  展开选项：                                   │
│  ☑ S  ☑ M  ☑ L                             │
│  （自动勾选另一维度的所有值）                    │
│                                              │
│  继承数据来源：                                │
│  ○ 空白（手动填写售价/库存）                    │
│  ● 复制首个匹配 SKU 的数据                     │
└─────────────────────────────────────────────┘
展开为新 SKU 行：在表格末尾新增 N 行（红色 × 每个另一维度值），继承指定数据
应用到已有 SKU：将"红色"值填充到用户选中的已有 SKU 行的颜色列
取消勾选行为变化：

当前：取消勾选"红色"→ 立即删除所有含红色的 SKU 行

新行为：取消勾选"红色"→ 弹出确认：


"取消勾选'红色'将从属性面板移除该选项。
含'红色'的 SKU 行将保留，但颜色列将显示为未注册值（橙色标记）。
是否继续？"
[继续] [取消]
3.2 切换分类行为变化
当前：切换分类 → 新增必填维度 → 笛卡尔积重建 → SKU 丢失

新设计：

切换分类 → 属性面板更新（新增必填属性维度面板）
SKU 表格新增对应维度的空列
显示提示条："新分类要求填写 [尺码]，请为每个 SKU 设置尺码值"
用户可以：
手动逐行填写
使用批量修改：选中多行 → 设置同一尺码值
使用"展开"功能：选中某行 → 展开为多个尺码
3.3 SKU 表格新增操作
操作	描述
添加行	手动在表格末尾添加空 SKU 行
删除行	删除选中的 SKU 行（带确认）
展开行	选择一行 → 按某维度展开为多行（继承数据）
合并行	选择多行同维度 → 合并为一行（取第一行数据）
复制行	复制选中行的数据到新行
3.4 发布前校验
发布时（非草稿保存）增加校验：


1. 每个 SKU 的所有必填维度值是否已填写
2. SKU 集合是否构成维度值的完整笛卡尔积（平台要求）
3. 如果不完整，提示用户：
   "当前 SKU 不构成完整的变体组合。
   缺少以下组合：红色-L、蓝色-M、蓝色-L
   [自动补全] [手动处理]"
自动补全：为缺失的组合创建 SKU 行（售价/库存留空，需用户填写）

四、技术设计
4.1 数据模型不变
后端数据模型（ShopItem.sales_properties、ShopItemSku.sku_infos）保持不变，因为：

平台发布接口仍需要完整笛卡尔积
已有的数据下载逻辑不需要改
变化仅在前端编辑交互层
4.2 前端架构变化
4.2.1 移除 changeSkuList() 的笛卡尔积重建
将 changeSkuList() 从"完整重建"改为"同步维度列显示"：

javascript

// 原逻辑：根据 sales_properties 笛卡尔积重建 skus 数组
// 新逻辑：只同步 SKU 表格显示的维度列，不改变 skus 数据
syncSkuTableColumns() {
    // 从 sales_properties 中提取当前启用的维度名
    const themes = (this.itemInfo.data.sales_properties || [])
        .filter(theme => theme.is_checked && theme.id !== 'currency')

    // 更新表格列配置（显示哪些维度列）
    this.skuDimensions = themes.map(t => t.name)

    // 不修改 this.itemInfo.skus
}
4.2.2 新增 expandSkuRows() 方法
替代笛卡尔积重建，按需展开：

javascript

// 将指定 SKU 行按某个维度展开为多行
expandSkuRows(sourceSkuIndex, dimensionName, values) {
    const sourceSku = this.itemInfo.skus[sourceSkuIndex]
    const newSkus = values.map(val => {
        const newSku = JSON.parse(JSON.stringify(sourceSku))
        newSku.data.sku_infos[dimensionName] = val
        newSku.down_sku_id = ''  // 新行编码需要用户填写
        newSku.autoid = null     // 标记为新增
        return newSku
    })
    // 在源行后面插入新行
    this.itemInfo.skus.splice(sourceSkuIndex + 1, 0, ...newSkus)
}
4.2.3 新增发布前完整性校验
javascript

validateCartesianCompleteness(item) {
    if (!item.data.has_variant) return []

    const dimensions = (item.data.sales_properties || [])
        .filter(p => p.is_checked && p.id !== 'currency')

    if (dimensions.length === 0) return []

    // 收集所有维度的已用值
    const usedValues = dimensions.map(d => {
        const values = new Set()
        item.skus.forEach(sku => {
            const val = sku.data?.sku_infos?.[d.name]
            if (val) values.add(val)
        })
        return { name: d.name, values: [...values] }
    })

    // 生成完整笛卡尔积
    const fullCombinations = cartesianOf(usedValues.map(d => d.values))

    // 检查缺失的组合
    const missing = fullCombinations.filter(combo => {
        return !item.skus.some(sku =>
            dimensions.every((d, i) => sku.data?.sku_infos?.[d.name] === combo[i])
        )
    })

    return missing  // 空数组表示完整
}
4.3 属性面板交互改造
4.3.1 勾选操作改为"建议性"
属性面板的勾选不再直接修改 skus，而是：

更新 sales_properties（面板展示用）
弹出操作面板让用户选择对 SKU 的影响
4.3.2 维度值与 SKU 的关联显示
在属性面板上，每个值旁边显示"N 个 SKU 使用此值"：


名称：颜色
☑ 红色 (3个SKU)   ☑ 蓝色 (2个SKU)   ☑ 绿色 (0个SKU) ⚠️
0 个 SKU 使用的值标记警告（面板有此选项但无 SKU 使用）。

4.4 兼容性处理
4.4.1 新建商品保持现有流程
新建商品（skus 为空）时，用户添加属性值后仍然走笛卡尔积生成——因为从零开始时，笛卡尔积是最高效的创建方式。

区分逻辑：

javascript

notifySkuListChanged(isModify) {
    if (isModify) { return }  // 编辑模式不重建（已有修复）

    // 新建模式：如果 skus 已有用户数据，走新逻辑（弹窗确认）
    if (this.itemInfo.skus.some(s => s.sale_price || s.down_sku_id)) {
        this.showDimensionChangeDialog()
        return
    }

    // 新建模式 + skus 为空/无数据：走笛卡尔积生成（保持现有行为）
    this.$nextTick(() => { this.changeSkuList() })
}
4.4.2 发布流程兼容
发布时的数据格式不变（getSaveObj 输出格式不变），后端无需改动。只是前端发布前多一步完整性校验。

五、实施分期
Phase 1（当前已完成）
✅ 方案 C：编辑模式不重建 SKU
Phase 2（低风险，建议优先）
切换分类时弹窗确认
新增"自动补全缺失组合"功能（发布前校验时提供）
notifySkuListChanged 中增加"skus 有数据时弹窗确认"逻辑
Phase 3（中等风险）
属性面板的勾选行为改造（从"驱动生成"变为"操作面板"）
SKU 表格新增"添加行/删除行/复制行"操作
维度值与 SKU 的关联计数显示
Phase 4（较大改动）
"展开行"和"合并行"功能
完整笛卡尔积校验 + 自动补全
新建商品时的智能判断（空数据走笛卡尔积，有数据走新逻辑）
六、风险评估
风险	影响	缓解措施
用户习惯变化	老用户可能不适应新的勾选行为	Phase 2 先只加确认弹窗，不改交互模式
不完整笛卡尔积导致发布失败	平台报错	发布前强制校验+自动补全功能
维度值编辑后 sku_infos 不一致	保存/发布数据错误	保存时同步 sales_properties（从 skus 反向推导）
复杂度增加	代码维护成本上升	分阶段实施，每阶段可独立回归测试
这个设计你觉得方向是否 OK？有没有哪些方面需要调整或深入讨论的？