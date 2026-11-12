
--Global.Inventory.SkuInventoryHistory(SkuInventoryHistoryTask) --gl_dwd.sku_inventory_snapshot_1--this.CalcRunNext(1, 1, 0, 20, Company.TimeZone);--02:11 ok

  

--(库龄)Report.CalAdsSkuInvAgeTask 库龄分析 --gl_ads.sku_inventory_age_1,gl_ads.sku_inventory_age_analyze_1--this.CalcRunNext(1, 2, 0, 5, companyTimeZone);-04:58 nn -->03

-

--（仓租）Report.Bms.CalInvAgeChargeTask --userruntime--this.CalcRunNext(1, 4, 0, 20, uc.CompanyEntity.Company.TimeZone); --04:11

--（发货）Report.Bms.CalInoutChargeTask --userruntime--this.CalcRunNext(1, 4, 0, 20, uc.CompanyEntity.Company.TimeZone);

--（计费）Report.Bms.ChargeDayAggregateTask --gl_ads.bms_charge_item_accountdate_1 --{"date_list":["2024-05-30","2024-05-31","2024-06-01","2024-06-02","2024-06-03","2024-06-04","2024-06-05","2024-06-06"]}--this.CalcRunNext(1, 5, 0, 20, uc.CompanyEntity.Company.TimeZone);