# fundamentals_schema.md
# 台股基本面研究專案 — 標準財報資料 Schema（資料字典）

**依據：** `RESEARCH_SPEC.md`（唯一權威來源）＋ `data_sources.md`（查核日 2026-09-16，僅作可得性與擷取機制參考）
**本文件性質：** 純資料字典，**不是**資料庫實作、不是擷取程式。定義「公司—季度—版本」這個觀測單位要存哪些欄位，以及每個欄位的型別／單位／可否為空／來源優先層級／規格出處。
**狀態：** Phase 1 — Data Engineering — M2 產出。尚未經過 M3 的人工驗證，不代表這些欄位在實務上一定能 100% 取得。
**表格粒度：** 一列 = 一家公司、一個財報季度、一個版本（as-reported 或某次 revision）。`as_of_date`（訊號建立截點）不屬於本表，留給 Phase 1 M4 的 `features_as_reported/` 表。

---

## 0. 命名對照表（Task A2）

`RESEARCH_SPEC.md` 與 `data_sources.md` 對同一概念用了不同名稱。以下對照表**只影響本 schema 文件的命名選擇，不修改原始兩份文件的用字**。

| 概念 | RESEARCH_SPEC 用字 | data_sources.md 用字 | 本 Schema 採用 | 選擇理由 |
|---|---|---|---|---|
| 期末日 | `fiscal_period_end` | `period_end_date` | `period_end_date` | 與 fiscal_year/fiscal_quarter 命名風格一致 |
| 首次公告日 | `publication_date` | `filing_date` | `filing_date` | data_sources.md 對應到 MOPS「財務報告公告」的具體法源語意較明確 |
| 首次公告時間 | `publication_time` | `filing_time` | `filing_time` | 同上 |
| 更正版本日 | `revision_date` | `revision_date`（+`revision_flag`） | `revision_date` + `revision_flag` | 採 data_sources.md 較細的拆分，方便查詢是否為更正版本而不必比對日期是否為空 |
| 原始文件識別 | `source_document_id` | `source_document_id`、`source_url`、`raw_file_hash`、`retrieved_at` | 全部採用（4 個獨立欄位） | data_sources.md 拆分較細，符合 §13.2 metadata 要求 |
| 產業分類 | 「產業分類與有效日」（未給確切欄位名） | `industry_at_filing` | `industry_code` + `industry_effective_date` + `industry_classification_source` | 兩份文件都只給概念，本 schema 拆成三欄以滿足 §2.2「歷史時點有效」要求 |
| 合併/個別口徑 | 「合併口徑」（文字描述） | `statement_scope` | `statement_scope` | data_sources.md 已給明確欄位名 |
| 營業成本 | `cost_of_revenue/COGS` | `COGS` | `cogs` | 統一小寫，避免大小寫不一致造成之後程式比對出錯 |
| 資本支出 | `capital_expenditure` | `capex_cash_paid` | `capex_cash_paid` | data_sources.md 的命名明確標出「現金支付額」，對應 §4.2 FCF 定義要求「不得以資產負債表增量替代」 |

以下欄位**只在其中一份文件出現**，本次判斷是否納入 schema：

| 欄位 | 只出現在 | 是否納入 | 理由 |
|---|---|---|---|
| `as_of_date` | RESEARCH_SPEC §3.1 | **不納入本表** | 屬於訊號截點，粒度不同，留給 Phase 1 M4 特徵表 |
| `market`、`ticker`、`unit_scale`、`is_ytd_value`、`derived_from_ytd`、`statutory_deadline` | data_sources.md | **納入** | 屬於實務上必要的識別／版本／QC 欄位，不與 RESEARCH_SPEC 衝突，只是 RESEARCH_SPEC 沒有明講細節 |
| `operating_expenses`、`EPS`、`income_tax`、`non_operating_items`、全部 8 個資產負債表欄位、`stock_based_compensation` | RESEARCH_SPEC §4.2 | **納入（必須）** | RESEARCH_SPEC 是唯一權威來源，data_sources.md 的精簡清單只是它自己聲明的「最小擷取設計」草案，不能拿來限縮正式 schema |

---

## 1. 主檔／識別欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `company_id` | string | 內部主鍵，建議用統一編號或交易所代碼 | 否 | — | — | RESEARCH_SPEC §4.2 |
| `ticker` | string | 市場交易代號 | 否 | — | — | data_sources.md §3.1 |
| `company_name` | string | 申報當時的公司名稱 | 否 | — | — | RESEARCH_SPEC §4.2 |
| `market` | enum | `TWSE_LISTED` / `TPEX_LISTED`（見下方 Review 註記，`TPEX_LISTED` 目前未經 M1 資料來源驗證） | 否 | — | — | data_sources.md 範圍聲明；RESEARCH_SPEC §2.1 母體定義 |
| `industry_code` | string | 申報當時有效的交易所／主管機關產業分類代碼 | 否 | `not_disclosed` | — | RESEARCH_SPEC §2.2 |
| `industry_effective_date` | date | 該分類代碼生效日 | 否 | — | — | RESEARCH_SPEC §2.2（禁止用現在分類回填過去） |
| `industry_classification_source` | string | 分類版本／來源說明（例如採用哪個年度的產業分類對照表） | 否 | — | — | RESEARCH_SPEC §2.2 |

---

## 2. 日期與版本欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `fiscal_year` | integer | 公司申報之會計年度 | 否 | — | — | RESEARCH_SPEC §3.1 |
| `fiscal_quarter` | integer | 1–4 | 否 | — | — | RESEARCH_SPEC §3.1 |
| `period_end_date` | date | 財報涵蓋期間結束日 | 否 | — | — | RESEARCH_SPEC §3.1（命名採 data_sources.md） |
| `statement_scope` | enum | `consolidated` / `standalone` | 否 | — | — | data_sources.md §1.2；RESEARCH_SPEC §4.2 |
| `filing_date` | date | MOPS 財務報告公告首次申報日 | 否（若真的查無日期則整筆記錄視為不可用，不建立此觀測） | — | 1 | RESEARCH_SPEC §3.1；data_sources.md §1.1 |
| `filing_time` | time | 若可得則填，含時區假設 Asia/Taipei | 是 | `not_available_from_source`（官方未提供時間戳，非資料缺失） | 1 | RESEARCH_SPEC §3.1；data_sources.md §1.1 |
| `availability_date` | date | 依 RESEARCH_SPEC §3.2 規則計算：有時間戳記則依收盤前後判斷；僅有日期則一律為公告日後下一交易日 | 否 | — | 衍生欄位（非原始揭露） | RESEARCH_SPEC §3.2 |
| `statutory_deadline` | date | 依當期法規計算的最後申報期限，**僅作 QC，不得當公告日** | 否 | — | — | data_sources.md §2 |
| `revision_flag` | boolean | 是否為更正／重編版本 | 否，預設 `false` | — | — | data_sources.md §1.1 |
| `revision_date` | date | 若為更正版本，其公開日 | 是（僅 `revision_flag=true` 時填） | — | — | RESEARCH_SPEC §3.1 |
| `source_document_id` | string | 原始文件唯一識別 | 否 | — | — | RESEARCH_SPEC §3.1 |
| `source_url` | string | 原始公告／文件連結 | 是（若連結後續失效仍保留歷史值，不刪除） | `link_no_longer_valid` | — | data_sources.md §1.1 |
| `raw_file_hash` | string | 原始檔案雜湊值，供可重現性稽核 | 否 | — | — | RESEARCH_SPEC §13.2 |
| `retrieved_at` | datetime (UTC) | 本次擷取時間 | 否 | — | — | RESEARCH_SPEC §13.2 |
| `source_tier` | enum (1–5) | 對應 RESEARCH_SPEC §4.1 的五層來源優先序 | 否 | — | — | RESEARCH_SPEC §4.1 |
| `extraction_method` | enum | `manual` / `table_extraction` / `api` | 否 | — | — | RESEARCH_SPEC §4.1 |

---

## 3. 損益表欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `revenue` | decimal | 依 `currency`／`unit_scale`；單季值 | 是 | `not_disclosed` / `non_comparable_period` | 依 `source_tier` | RESEARCH_SPEC §4.2 |
| `cogs` | decimal | 同上 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `gross_profit` | decimal | 同上 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `operating_expenses` | decimal | 同上 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2（data_sources.md 未列，依 SPEC 補入） |
| `operating_income` | decimal | 同上 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `net_income` | decimal | 同上 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `eps` | decimal | 元/股 | 是 | `not_disclosed` | 同上 | RESEARCH_SPEC §4.2（data_sources.md 未列，依 SPEC 補入） |
| `income_tax` | decimal | 依 `currency`／`unit_scale` | 是 | `not_disclosed` | 同上 | RESEARCH_SPEC §4.2（同上） |
| `non_operating_items` | decimal | 同上 | 是 | `not_disclosed` | 同上 | RESEARCH_SPEC §4.2（同上） |
| `is_ytd_value` | boolean | 該筆原始揭露是否為年初至今累計值 | 否，預設 `false` | — | — | data_sources.md §1.2 |
| `derived_from_ytd` | boolean | 本記錄的單季值是否由 YTD 相減推算 | 否，預設 `false` | — | — | RESEARCH_SPEC §4.3 |
| `ytd_value_raw` | decimal | 若 `is_ytd_value=true`，保存原始累計值（稽核用，不覆寫） | 是（僅 YTD 揭露時填） | — | — | RESEARCH_SPEC §4.3 |
| `prior_period_source_document_id` | string | 若 `derived_from_ytd=true`，記錄用於相減的前期來源文件 ID | 是 | — | — | RESEARCH_SPEC §4.3 |

> `Q(i,t) = YTD(i,t) - YTD(i,t-1)`：若因重編、會計年度變更、合併範圍改變或缺少可比前期而無法可信拆分，`derived_from_ytd` 仍設 `true` 但主要數值欄位（如 `revenue`）設為缺失，`missing_reason = 'ytd_split_not_reliable'`（RESEARCH_SPEC §4.3）。

---

## 4. 現金流量表欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `operating_cash_flow` | decimal | 依 `currency`／`unit_scale`；單季值 | 是 | `not_disclosed` | 依 `source_tier` | RESEARCH_SPEC §4.2 |
| `capex_cash_paid` | decimal | 現金支付之資本支出，**非**資產負債表增量 | 是 | `not_disclosed` / `capex_not_identifiable` | 同上 | RESEARCH_SPEC §4.2；data_sources.md §1.1 |
| `free_cash_flow` | decimal（衍生欄位） | 公式固定：`operating_cash_flow - capex_cash_paid` | 是（`capex_cash_paid` 缺失時必為缺失，**不得**用資產負債表增量替代） | `capex_not_identifiable` | 衍生（非原始揭露） | RESEARCH_SPEC §4.2 |
| `free_cash_flow_formula_version` | string | 記錄計算公式版本，供未來公式變更時追溯 | 否 | — | — | RESEARCH_SPEC §13.2（可重現性） |
| `stock_based_compensation` | decimal | 如適用；依 `currency`／`unit_scale` | 是 | `not_disclosed` / `not_applicable` | 同上 | RESEARCH_SPEC §4.2 |

---

## 5. 資產負債表欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `cash_and_equivalents` | decimal | 期末餘額 | 是 | `not_disclosed` | 依 `source_tier` | RESEARCH_SPEC §4.2 |
| `total_debt` | decimal | 期末餘額 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `net_debt` | decimal | 若非公司直接揭露，需標記為衍生值並保留公式版本（比照 FCF 處理方式） | 是 | `not_disclosed` / `derived_definition_pending` | 同上 | RESEARCH_SPEC §4.2 |
| `accounts_receivable` | decimal | 期末餘額 | 是 | `not_disclosed` | 同上 | RESEARCH_SPEC §4.2 |
| `inventory` | decimal | 期末餘額 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `deferred_revenue` | decimal | 期末餘額 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `shareholders_equity` | decimal | 期末餘額 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |
| `shares_outstanding` | decimal | 股數 | 是 | 同上 | 同上 | RESEARCH_SPEC §4.2 |

> 本次判斷：`data_sources.md` 完全沒有討論資產負債表欄位的可得性與公告日處理，這 8 個欄位目前只有「RESEARCH_SPEC 要求要收」的依據，尚未經過類似季度損益表那樣的資料來源可行性研究。**建議 Phase 1 M3 小型人工驗證時，一併驗證這 8 個欄位能否從同一份 MOPS 財報／iXBRL 取得，不要假設它一定跟損益表一樣好取得。**

---

## 6. 營運資料欄位（選用，僅供 S2B／描述性分析）

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `units_sold` | decimal | 僅公司正式揭露時填 | 是（預期大量缺失） | `not_disclosed` | 依 `source_tier` | RESEARCH_SPEC §6.2 Level B |
| `asp` | decimal | 計算欄位：`revenue_by_product / units_sold` | 是 | `not_disclosed` / `units_not_disclosed` | 衍生 | RESEARCH_SPEC §6.2 |
| `product_segment` | string | 產品／地區別標籤 | 是 | `not_disclosed` | — | RESEARCH_SPEC §4.2 |
| `product_revenue` | decimal | 該產品／地區別營收 | 是 | `not_disclosed` | 依 `source_tier` | RESEARCH_SPEC §4.2 |
| `allocated_cogs` | decimal | **僅**公司直接揭露且可與產品營收一一對應時填，不得用總 COGS 除單一產品銷量冒充 | 是 | `not_disclosed` / `allocation_not_available` | 依 `source_tier` | RESEARCH_SPEC §6.2 |
| `customer_concentration` | decimal | 主要客戶集中度（如前十大客戶占比） | 是 | `not_disclosed` | 依 `source_tier` | RESEARCH_SPEC §4.2 |
| `estimated_flag` | boolean | 標記本列是否含估計值（例如 `S2B` 相關欄位） | 否，預設 `false` | — | — | RESEARCH_SPEC §6.2（`S2B` 永遠附帶 `estimated=true`） |
| `disclosure_scope_note` | string | 說明揭露範圍（例如只揭露前三大產品） | 是 | — | — | RESEARCH_SPEC §6.2 |

---

## 7. 品質與缺失欄位

| 欄位 | 型別 | 單位/格式 | Nullable | 缺失原因分類適用 | 來源優先層級 | 規格出處 |
|---|---|---|---|---|---|---|
| `currency` | string | ISO 貨幣代碼，預設 `TWD` | 否 | — | — | RESEARCH_SPEC §4.2 |
| `unit_scale` | enum | `unit` / `thousand` / `million` | 否 | — | — | data_sources.md §3.1 |
| `comparability_flag` | boolean | 合併、分割、會計準則改變、主要業務出售／取得等不可比事件 | 否，預設 `false` | — | — | RESEARCH_SPEC §5.2 |
| `comparability_note` | string | 說明不可比原因 | 是（僅 `comparability_flag=true` 時應填） | — | — | RESEARCH_SPEC §5.2 |
| `data_quality_note` | string | 自由文字補充說明 | 是 | — | — | RESEARCH_SPEC §4.2 |
| `missing_reason` | enum | 統一分類，例如：`not_disclosed` / `not_applicable` / `ytd_split_not_reliable` / `capex_not_identifiable` / `non_comparable_period` / `allocation_not_available` / `units_not_disclosed` / `link_no_longer_valid` / `not_available_from_source` | 是（每個財務欄位為空時，理論上都應能對應到這裡的一個分類） | — | — | RESEARCH_SPEC §4.5（禁止用 0／產業平均／未來值／模型預測填補） |

---

## 8. 用五大訊號與目標字典反向檢查（Task J）

| 檢查項目 | 需要欄位 | 結論 |
|---|---|---|
| S1 Revenue Growth Acceleration：需要 `R(t)`, `R(t-1)`, `R(t-4)`, `R(t-5)` | `revenue` + `fiscal_year`/`fiscal_quarter`（按公司分組排序即可取得歷史滯後值，不需要額外欄位） | ✅ 通過。前提：同一 `company_id` 底下每季都有一列，長表設計即可支援 |
| S2A Cost Ratio 代理值：`CostRatio = cogs/revenue`，需同比（t vs t-4） | `revenue`、`cogs` | ✅ 通過 |
| S2B 產品層：需要 `ASP`、`AllocatedCOGS`、`Units` | `product_revenue`、`allocated_cogs`、`units_sold`（第 6 節選用欄位） | ⚠️ 條件通過。這些欄位預期大量缺失，能不能真的支援 S2B 要等 M3 驗證，也要等 §6.2 要求的「彙總公式與權重預先註冊」完成才能進主檢驗 |
| S3 GM/OM Delta：需要 `GM(t)`, `GM(t-4)`, `OM(t)`, `OM(t-4)` | `revenue`、`cogs`、`gross_profit`、`operating_income` | ✅ 通過 |
| S4 現金轉換（TTM）：需要連續四季 `NI`、`OCF`、`FCF` | `net_income`、`operating_cash_flow`、`free_cash_flow`（衍生） | ✅ 通過。前提：schema 需保證同公司連續季度都能撈到，缺一季就整個 TTM 視為缺失（不得用 3 季代 4 季） |
| §7.2 目標字典（`Y_RG_h`、`Y_GM_h`、`Y_OM_h`、`Y_GM_delta_h`、`Y_OM_delta_h`、`Y_OCF_margin_h`、`Y_FCF_margin_h`、`Y_OCF_conversion_h`、`Y_FCF_conversion_h`） | 與訊號共用同一批欄位（`revenue`、`gross_profit`、`operating_income`、`operating_cash_flow`、`free_cash_flow`、`net_income`） | ✅ 通過。同一張表在「未來 t+h」季度同樣適用，不需要新增欄位，只需要在 Phase 1 M4 建 `data/targets/` 時從同一張原始表往未來方向取值，不與特徵表共用同一份 as-of 快照（避免 look-ahead） |

**結論：本 schema 的欄位集合足以支撐 RESEARCH_SPEC §6 全部五大訊號與 §7.2 全部目標的計算，唯一有條件保留的是 S2B（產品層），其資料可得性風險已在第 6 節與本節註記，留給 M3 實測驗證，不在此階段假設一定可行。**

---

## 9. 尚未解決、留給 M3／之後決定的事項

1. `market = TPEX_LISTED` 目前只是 schema 上的占位，`data_sources.md` 完全沒有驗證過上櫃公司的公告規則與可得性（見 `CURRENT_TASK.md` Review #1）。**在 M3 驗證前，不應該對上櫃公司的資料回補做任何假設。**
2. 資產負債表 8 個欄位的來源可行性未經 `data_sources.md` 討論，建議 M3 一併驗證（見第 5 節註記）。
3. `net_debt` 若無公司直接揭露，其定義（例如是否扣除受限制現金）尚未預先固定，M3 驗證階段若發現需要衍生計算，公式版本必須在看任何目標結果前鎖定，比照 `free_cash_flow` 的處理方式（RESEARCH_SPEC §13.2 可重現性要求）。
