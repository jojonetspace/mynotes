**限流：：：**
下面把前面从 Shopee 官方 `v2.product.update_price` 接口文档(open.shopee.com)查到的限流相关错误码整理成表格。内容为合规做了转述。

## Shopee 限流 / 调用量限制错误码

|错误码|所属分类|含义|性质|系统是否自动重试|
|---|---|---|---|---|
|`error_rate_limit`|Common Error（网关层）|请求过于频繁，已触达频率上限，请稍后再试|短时频率限制（店铺级/App 级）|否（不在白名单）|
|`error_limit`|Common Error（网关层）|当天 API 调用总量已达每日上限，需等到次日 00:00（UTC+8）后恢复|每日配额限制|否（不在白名单）|

> 补充：Shopee 文档里还有 `error_too_many_request`（请求太多）这一类频率超限码,它在本系统的重试白名单 `RetriableErrorCodes` 里，命中后会走延迟重试、不终止任务。而上面两个 `error_rate_limit` / `error_limit` 目前不在白名单。

## 对照：本次报错的 error_update_price_fail（证明不是限流）

|错误码|所属分类|含义|是否限流|
|---|---|---|---|
|`error_update_price_fail`|Business Error（业务层）|更新价格失败，请稍后再试（笼统兜底文案，真因在 `failure_list[].failed_reason`）|否|

## 系统侧重试白名单（`ShopItemPlfService.cs` 的 `RetriableErrorCodes`）

命中以下错误码才会被判为"临时性错误、可重试"，其余（含 `error_update_price_fail`）直接判 Fatal 终止：

|错误码|含义|
|---|---|
|`error_server`|平台服务端临时错误|
|`error_too_many_request`|请求频率超限|
|`error_system_busy`|平台系统繁忙|
|`error_network`|网络超时|
|`error_inner_timeout`|平台内部处理超时|

一句话结论：真正的"限流"码是 `error_rate_limit` 和 `error_limit`（外加白名单里的 `error_too_many_request`）；你这次遇到的 `error_update_price_fail` 是业务错误，不是限流。

需要的话我可以把这几张表整理成一个 Markdown 文档放到 `Doc/` 下，方便团队以后查。要不要？