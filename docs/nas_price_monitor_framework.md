# UGREEN NAS 竞品价格自动化监控项目框架（MVP 方案）

## 1. 项目目标理解
本项目的目标是建立一个以 Amazon 为第一阶段监控平台、基于 Keepa API 的 NAS 竞品价格监控体系，支持 US、DE、JP 站点（UK 先预留）。

系统输出将用于解释 UGREEN NAS 在 CVR、AOV、Sessions、GMV 等核心营销指标异动，并为周报提供可复用的数据产出（CSV + Excel）。

---

## 2. 推荐项目目录结构

```text
ugreen_nas_price_monitor/
├── README.md
├── requirements.txt
├── config/
│   ├── markets.yaml
│   ├── brands.yaml
│   ├── keywords_include.txt
│   ├── keywords_exclude.txt
│   ├── alert_rules.yaml
│   └── keepa_api.yaml.example
├── input/
│   ├── seed_asins.csv
│   ├── ugreen_focus_models.csv
│   └── manual_exclusion_asins.csv
├── data/
│   ├── raw/
│   │   ├── keepa_product_query/
│   │   └── keepa_price_history/
│   ├── processed/
│   │   ├── discovered_competitor_asins.csv
│   │   ├── price_snapshot_YYYYMMDD.csv
│   │   └── price_alerts_YYYYMMDD.csv
│   └── reports/
│       └── weekly_price_report_YYYYMMDD.xlsx
├── src/
│   ├── main.py
│   ├── discover/
│   │   ├── discover_asins.py
│   │   └── filtering_rules.py
│   ├── snapshot/
│   │   ├── fetch_prices.py
│   │   └── normalize_prices.py
│   ├── alerts/
│   │   └── detect_alerts.py
│   ├── report/
│   │   └── build_excel_report.py
│   ├── common/
│   │   ├── keepa_client.py
│   │   ├── io_utils.py
│   │   ├── date_utils.py
│   │   └── logging_utils.py
│   └── qa/
│       └── data_quality_checks.py
├── scripts/
│   ├── run_daily.sh
│   └── run_weekly_report.sh
└── docs/
    ├── business_definition.md
    ├── field_dictionary.md
    └── operation_guide.md
```

---

## 3. 各文件职责
- `config/*`：业务规则与参数配置。
- `input/*`：人工输入的种子 ASIN、UGREEN 对标机型、排除列表。
- `src/discover/*`：竞品候选发现与过滤。
- `src/snapshot/*`：价格抓取与字段标准化。
- `src/alerts/*`：价格异常检测。
- `src/report/*`：Excel 周报生成。
- `src/qa/*`：数据质量检查。
- `data/processed/*`：可直接用于分析的结构化输出。
- `docs/*`：口径说明、字段字典、操作指南。

---

## 4. 数据流转流程
1. 读取配置与输入文件。
2. 使用 Keepa 进行竞品候选 ASIN 发现。
3. 执行过滤规则，排除非 NAS 整机。
4. 生成并更新 `discovered_competitor_asins.csv`。
5. 抓取每日价格快照，生成 `price_snapshot_YYYYMMDD.csv`。
6. 基于 7/30 天基线计算价格异常，生成 `price_alerts_YYYYMMDD.csv`。
7. 汇总生成 `weekly_price_report_YYYYMMDD.xlsx`。

---

## 5. 输入文件设计
- `seed_asins.csv`：初始竞品样本池。
- `ugreen_focus_models.csv`：UGREEN 重点机型映射。
- `manual_exclusion_asins.csv`：人工黑名单。

可选：`manual_inclusion_asins.csv`（强制纳入 ASIN）。

---

## 6. 输出文件设计
- `discovered_competitor_asins.csv`
- `price_snapshot_YYYYMMDD.csv`
- `price_alerts_YYYYMMDD.csv`
- `weekly_price_report_YYYYMMDD.xlsx`

---

## 7. discovered_competitor_asins.csv 字段
- `snapshot_date`
- `marketplace`
- `asin`
- `title`
- `brand`
- `model_guess`
- `capacity_bay_guess`
- `category_path`
- `is_nas_core`
- `exclude_reason`
- `discovery_source`
- `confidence_score`
- `first_seen_date`
- `last_seen_date`
- `is_active`

---

## 8. price_snapshot_YYYYMMDD.csv 字段
- `snapshot_date`
- `snapshot_datetime_utc`
- `marketplace`
- `asin`
- `title`
- `brand`
- `model_guess`
- `currency`
- `list_price`
- `buybox_price`
- `new_price`
- `sales_price`
- `coupon_text`
- `coupon_value_est`
- `effective_price`
- `price_source`
- `is_in_stock`
- `seller_type`
- `data_quality_flag`

---

## 9. price_alerts_YYYYMMDD.csv 字段
- `alert_date`
- `marketplace`
- `asin`
- `brand`
- `model_guess`
- `current_effective_price`
- `baseline_7d_median`
- `baseline_30d_median`
- `change_vs_7d_pct`
- `change_vs_30d_pct`
- `alert_type`
- `severity`
- `trigger_rule_id`
- `trigger_rule_desc`
- `possible_business_impact`
- `notes`

---

## 10. Excel 报告建议 Sheet
1. `Executive_Summary`
2. `Market_Overview`
3. `Brand_Trend`
4. `ASIN_Alerts`
5. `UGREEN_Impact_View`
6. `Data_Quality`
7. `Rule_Definitions`

---

## 11. 竞品筛选规则（排除非 NAS 整机）

### 纳入规则（至少命中 2 项）
- 品牌在白名单中。
- 标题命中 NAS 核心词（NAS、DiskStation、LinkStation、TS-、DS-、Lockerstor 等）。
- 类目路径命中 NAS 相关类目。

### 强排除规则（命中任一排除）
- 存储介质：`HDD`, `SSD`, `Hard Drive`, `NVMe`, `SATA Drive`
- 内存：`RAM`, `Memory`, `DDR`, `SO-DIMM`
- 扩展柜/阵列：`Expansion`, `Enclosure`, `DAS`, `JBOD`
- 电源/线材：`Power Adapter`, `Cable`, `Cord`, `PSU`
- 许可软件：`License`, `Activation`, `Subscription`
- 成色：`Refurbished`, `Renewed`, `Used`, `Open Box`
- 配件：`Tray`, `Caddy`, `Bracket`, `Fan`, `Heatsink`

### 评分与灰度池
- NAS 核心词 +30
- 类目正确 +30
- 型号模式命中 +20
- 排除词命中 -100
- `confidence_score >= 60` 纳入正式池；`40~59` 进入人工复核。

---

## 12. 价格异常规则
- 短期突降：低于 7 日中位数 15%+
- 短期突涨：高于 7 日中位数 15%+
- 中期偏离：相对 30 日中位数偏离 20%+
- 促销开始：出现新 coupon 且到手价降幅 >8%
- 促销结束：coupon 消失且到手价回升 >8%
- 高频波动：7 天内 3 次以上 >10% 波动

严重级别：
- High：25%+
- Medium：15%~25%
- Low：8%~15%

---

## 13. MVP 版本路线

### 第一版（先做）
- Amazon US/DE/JP
- Keepa 数据拉取
- 候选 ASIN 发现
- 每日价格快照
- 基础异常规则
- CSV + Excel 输出

### 第二版（再扩展）
- UK 正式启用
- UGREEN 机型自动对标增强
- 告警聚合与分组
- 增加销量/BSR辅助解释
- 半自动人工复核流程
- 后续可对接数据库与 BI

---

## 14. 代码结构可维护性策略
- 配置与代码解耦（阈值/品牌/关键词全配置化）。
- 按功能模块拆分（discover/snapshot/alerts/report/qa）。
- 统一字段字典，固定分析口径。
- 每次运行产生日志与质量检查结果。
- 先稳定主链路，再扩展高级功能。
- 命名业务友好，降低分析师接手门槛。
