[https://shopify.dev/](https://shopify.dev/)
shedou@jushuitan.com
Jst!QzPm@RFV123%

5bc96e55e06e2a9bd824cf1ca2c1ed24
77949bcb3ab721927afbba8b92da0829

https://wujingtest-3xl0a5as.myshopify.com

| 权限代码                           | 作用                                      |
| ------------------------------ | --------------------------------------- |
| `read_metaobjects`             | 读取元对象条目                                 |
| `write_metaobjects`            | 创建、修改元对象条目（类目属性的值存在这里）                  |
| `read_metaobject_definitions`  | 读取元对象定义                                 |
| `write_metaobject_definitions` | 启用标准元对象定义（如颜色 `shopify--color-pattern`） |

| 系统里的叫法 | 存储字段                                     | Shopify 对应                                                                |
| ------ | ---------------------------------------- | ------------------------------------------------------------------------- |
| 商品属性   | `product_properties`（`PRODUCT_PROPERTY`） | 后台叫"类别元字段"（category metafields）。可选哪些属性由分类决定，所以其他平台也叫它类目属性。颜色、年龄组、适用性别都在这里 |
| 销售属性   | `sales_properties`                       | 商品选项 `productOptions`，也就是变体维度                                             |
| 其他属性   | `other_properties`                       | 产品元字段、系列等                                                                 |
![[Pasted image 20261125094911.png]]

todo:
1. Metaobject的id为gid://shopify/Metaobject/669699309699；这个前缀都一样的，优化一下吧
2. 跨店复制和批量修改属性页。跨店复制时值是 A 店铺的条目 ID，到 B 店铺会写入失败。这个放到之后做复制和批量适配时一起处理。
3. 还有截图中的已更新库存这个，我记得这个商品之前好像是不跟踪的，是不是我们发布后改为跟踪了？这个我不确定，你帮我看看
4. 添加颜色弹窗界面优化


| [名称](https://dev.shopify.com/dashboard/129472160/stores?direction=asc&sort=name)                               | [类型](https://dev.shopify.com/dashboard/129472160/stores?direction=asc&sort=type) | [状态](https://dev.shopify.com/dashboard/129472160/stores?direction=asc&sort=status) | [组织](https://dev.shopify.com/dashboard/129472160/stores?direction=asc&sort=organization) | 套餐  | [上次访问时间](https://dev.shopify.com/dashboard/129472160/stores?direction=asc&sort=last_accessed) | 行动  |
| -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | --- | --------------------------------------------------------------------------------------------- | --- |
| [wujingtest](https://dev.shopify.com/dashboard/129472160/stores/73659482243)wujingtest-3xl0a5as.myshopify.com  | 开发                                                                               | —                                                                                  | 上海聚水潭网络科技有限公司                                                                            | —   | 2026年11月23日                                                                                   |     |
| [jst001](https://dev.shopify.com/dashboard/129472160/stores/85772697899)jst001.myshopify.com                   | 客户转让                                                                             | 开发中                                                                                | 上海聚水潭网络科技有限公司                                                                            | —   | 2026年11月16日                                                                                   |     |
| [longchuantest](https://dev.shopify.com/dashboard/129472160/stores/68123296021)longchuantest.myshopify.com     | 客户转让                                                                             | 开发中                                                                                | 上海聚水潭网络科技有限公司                                                                            | —   | 2026年7月29日                                                                                    |     |
| [yilong](https://dev.shopify.com/dashboard/129472160/stores/80810705135)yilong-2.myshopify.com                 | 开发                                                                               | —                                                                                  | 上海聚水潭网络科技有限公司                                                                            | —   | 2026年6月14日                                                                                    |     |
| [wujingtest](https://dev.shopify.com/dashboard/129472160/stores/57636061238)wujingtest.myshopify.com           | 客户转让                                                                             | 开发中                                                                                | 上海聚水潭网络科技有限公司                                                                            | —   | 2026年3月20日                                                                                    |     |
| [longchuantest24](https://dev.shopify.com/dashboard/129472160/stores/59631075406)longchuantest24.myshopify.com | 客户转让                                                                             | 开发中                                                                                | 上海聚水潭网络科技有限公司                                                                            | —   | 2025年8月12日                                                                                    |     |