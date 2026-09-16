# CURRENT_TASK.md
# 目前唯一要做的事情

**最後更新：** 2026-09-16
**目前所在：** Phase 1 — Data Engineering ／ **M2 — 建立標準財報資料 Schema**
**這份文件的角色：** 只放「現在」要做的事。長期規劃看 `ROADMAP.md`，不要把長期任務搬來這裡。

---

## 這個 Milestone 要交付什麼

一份**資料字典（data dictionary）文件**，定義「公司—季度—版本」這個觀測單位要存哪些欄位、每個欄位的型別／單位／是否可為空／來源優先層級。

**明確不做的事（避免範圍蔓延）：**
- 不寫擷取程式、不寫 API 串接
- 不建資料庫、不決定用 PostgreSQL / SQLite / Parquet 等實作細節
- 不做欄位的實際回補或驗證（那是 M3 的事）
- 不定義訊號或目標的計算程式（那是 Phase 2 的事）

輸出檔案建議位置：`data/schema/fundamentals_schema.md`（或你的 repo 慣用的等效路徑；路徑可以調整，但「這是純文件、不是程式」這件事不能變）。

---

## 為什麼欄位要這樣選（依據，不是我自己想的）

`RESEARCH_SPEC.md` 是唯一權威來源，欄位需求主要來自：
- §3.1（日期與版本欄位）
- §4.2（Phase 1 資料字典最低集合）
- §5、§6（比率與五大訊號公式，用來反推「算得出來需要哪些欄位」）
- §7.2（目標字典，同樣用來反推欄位需求）
- §13.3（程式不變量，用來確認 schema 有沒有存到之後能檢查這些不變量的欄位）

`data_sources.md` 提供的是**可得性與命名慣例**（§3.1 的擷取欄位清單、filing_date 的處理規則），但它的欄位清單比 `RESEARCH_SPEC.md` §4.2 精簡很多（只涵蓋 revenue/COGS/gross_profit/operating_income/net_income/OCF/capex）。**判斷：以 `RESEARCH_SPEC.md` §4.2 為準，`data_sources.md` 只用來決定「這些欄位實務上怎麼取得、怎麼保存版本」。**

---

## Tasks（照順序做，做完打勾）

### A. 欄位盤點與整併
- [ ] A1. 把 `RESEARCH_SPEC.md` §3.1（日期／版本欄位）、§4.2（財報欄位）與 `data_sources.md` §3.1（擷取欄位）三份清單並排列表，找出重複、找出命名不一致的地方（例如 `filing_date` vs `publication_date`、`period_end_date` vs `fiscal_period_end`）
- [ ] A2. 決定最終欄位命名（可以選其中一份文件的命名，或另訂一致命名），並寫一張「命名對照表」記錄舊名稱怎麼對應到新名稱——**不要回頭改 `RESEARCH_SPEC.md` 或 `data_sources.md` 裡的名稱，只在新的 schema 文件裡做對照**

### B. 主檔／識別欄位
- [ ] B1. 定義 `company_id`、`ticker`、`company_name`、`market`（上市/上櫃，見下方 Review 項目）
- [ ] B2. 定義產業分類欄位與其生效日（`RESEARCH_SPEC.md` §2.2 要求「歷史時點有效」的分類，不能用現在的分類回填過去）——欄位至少要能存「分類代碼」＋「生效日」＋「分類版本來源」

### C. 日期與版本欄位（`RESEARCH_SPEC.md` §3.1 + `data_sources.md` §1.1/§2）
- [ ] C1. `fiscal_year`、`fiscal_quarter`、`period_end_date`
- [ ] C2. `filing_date`、`filing_time`（可為空，且要記錄「為什麼是空的」而不是留白）
- [ ] C3. `availability_date`（依 `RESEARCH_SPEC.md` §3.2 規則：無時間戳一律視為下一交易日可用）
- [ ] C4. `revision_flag`、`revision_date`（更正／重編另建版本，不覆寫原始值）
- [ ] C5. `source_document_id`、`source_url`、`raw_file_hash`、`retrieved_at`
- [ ] C6. `statutory_deadline`（僅作 QC 用，不可當公告日，依 `data_sources.md` §2 說明加註）

> 注意：`as_of_date` 是「建立訊號時使用的資訊截點」，屬於訊號／特徵表的欄位，不是這張原始財報表的欄位。這裡只需要確保 `availability_date` 存在且定義正確，`as_of_date` 留到 Phase 1 M4／Phase 2 再處理，避免這次 schema 混淆兩個不同粒度的表。

### D. 損益表欄位（`RESEARCH_SPEC.md` §4.2）
- [ ] D1. `revenue`、`cost_of_revenue`(COGS)、`gross_profit`、`operating_expenses`、`operating_income`、`net_income`、`EPS`、`income_tax`、`non_operating_items`
- [ ] D2. 每個欄位標記：合併口徑／個別口徑要不要分開存？（建議：用 `statement_scope` 欄位區分，而不是拆成兩組欄位，避免欄位爆炸）
- [ ] D3. 標記單季值與 YTD 累計值的處理：`is_ytd_value`（布林）、`derived_from_ytd`（布林）、`derived_quarter_value`（若為推算值存推算後數字，並保留原始 YTD 於另一欄或另一筆記錄）——依 `RESEARCH_SPEC.md` §4.3 公式 `Q(i,t) = YTD(i,t) - YTD(i,t-1)`

### E. 現金流量表欄位
- [ ] E1. `operating_cash_flow`、`capital_expenditure`（現金支付額，非應計）
- [ ] E2. `free_cash_flow`：**不存原始揭露值**，改存「計算欄位」＋公式版本註記，公式固定為 `OCF - CapEx`（`RESEARCH_SPEC.md` §4.2）；若 CapEx 現金支付額無法辨認，此欄位必須是缺失，不得用資產負債表增量替代
- [ ] E3. `stock_based_compensation`（如適用，允許缺失）

### F. 資產負債表欄位
- [ ] F1. `cash_and_equivalents`、`total_debt`、`net_debt`、`accounts_receivable`、`inventory`、`deferred_revenue`、`shareholders_equity`、`shares_outstanding`

### G. 營運資料欄位（選用，僅供 S2B）
- [ ] G1. `units_sold`、`ASP`、產品／地區營收、產品組合、主要客戶集中度——標記為**選用欄位**，只有公司正式揭露時才收錄，並附 `estimated`／`disclosure_scope` 說明欄位，對應 `RESEARCH_SPEC.md` §6.2 Level B 的規則

### H. 品質與缺失欄位
- [ ] H1. `currency`、`unit_scale`（例如仟元／百萬元）
- [ ] H2. `comparability_flag`（合併、分割、會計準則改變等不可比事件，§5.2）
- [ ] H3. `data_quality_note`、`missing_reason`（缺失要有分類，不能只留空值；§4.5 禁止用 0／產業平均／未來值／模型預測填補）
- [ ] H4. `source_tier`、`extraction_method`、`retrieved_at`（§4.1 要求每個數值都要能追溯來源優先層級）

### I. 型別、單位、Null 規則
- [ ] I1. 逐欄位標註資料型別（date / datetime / string / decimal / boolean / enum）
- [ ] I2. 逐欄位標註單位（貨幣、股數、比率是否已轉小數——依 §5.1「所有百分比以小數儲存」）
- [ ] I3. 逐欄位標註 nullable 與否，若可為空要對應到 H3 的 `missing_reason` 分類，而不是「隨便空著」

### J. 用五大訊號與目標字典反向檢查（不可省略）
- [ ] J1. 用 S1 公式（需要 `R(t)`, `R(t-1)`, `R(t-4)`, `R(t-5)`）確認 schema 能透過「同一公司多筆季度記錄」取得這些歷史值（不需要額外欄位，但要確認資料表設計是長表 by company+quarter，能撈到 t-1/t-4/t-5）
- [ ] J2. 用 S2A（`CostRatio = COGS/Revenue`）、S3（`GM_Delta`, `OM_Delta`）確認 D、E 區欄位足夠
- [ ] J3. 用 S4（TTM 版本）確認 schema 能撈到連續四季，且 `NI_TTM > 0` 才定義的規則有對應的缺失標記方式
- [ ] J4. 用 §7.2 目標字典（`Y_RG_h`、`Y_GM_h` 等）確認同一批欄位在「未來 t+h」季度也適用，不需要另建欄位，只需要確認同一張表可以同時當特徵來源與目標來源（這是設計上的一致性檢查，不是新欄位）

### K. 產出與收尾
- [ ] K1. 把 A–J 的結論整理成 `data/schema/fundamentals_schema.md`，格式建議：每個欄位一列，欄位＝名稱／型別／單位／nullable／缺失原因分類／來源優先層級／對應 RESEARCH_SPEC 章節出處
- [ ] K2. 文件開頭寫清楚：本 schema 依據 `RESEARCH_SPEC.md`（版本號）與 `data_sources.md`（查核日）制定；本 schema 只是資料字典，不代表已完成任何實際擷取或驗證
- [ ] K3. 把下方「需要 Review」項目原封不動抄一份到 `research_log.md`（若尚未有記錄格式，先用最簡單的「日期＋發現＋狀態」即可）
- [ ] K4. `git add` → `git commit`（訊息建議：`Phase1 M2: define standard fundamentals data schema`）→ `git push`
- [ ] K5. 回到 `ROADMAP.md`，把 Phase 1 M2 狀態改成 ✅ Done，並依 M3（小型人工驗證）改寫這份 `CURRENT_TASK.md`

---

## 需要 Review 的問題（本次發現，不自行修改規則）

> 依專案規則第 10 條：發現規則有問題只記錄，不擅自修改。以下項目請你自己決定怎麼處理，我不會替你改 `RESEARCH_SPEC.md` 或 `data_sources.md`。

1. **Universe 範圍與資料來源不對等。**
   `RESEARCH_SPEC.md` §2.1 定義母體為「台灣**上市與上櫃**一般產業公司」，但 `data_sources.md` 的範圍聲明只涵蓋「台灣證券交易所（TWSE）**上市**公司」，全文沒有提到上櫃（TPEx／櫃買中心）的資料來源、公告規則或申報期限是否與上市公司一致。
   → 影響：如果 Schema 直接照 `RESEARCH_SPEC.md` 的母體定義做，`market` 欄位理論上要能存上櫃，但目前完全沒有驗證過上櫃公司的 `filing_date`／`availability_date` 能不能用同一套邏輯取得。
   → 建議處理方式（僅供參考，不是指示）：可以在本次 schema 設計時把 `market` 欄位設計成可擴充的 enum（例如 `TWSE_LISTED` / `TPEX_LISTED`），但在正式回補資料前，回頭把 `data_sources.md` 的範圍問題另外處理（可能需要新增一節，或明確縮小 `RESEARCH_SPEC.md` 當前階段的 Universe 到「先做上市，上櫃另外排」——這屬於研究假設，需要你決定，不是我可以自己改的）。

2. **`RESEARCH_SPEC.md` 與 `data_sources.md` 的欄位命名不完全一致。**
   例如「財報公告日」在 `RESEARCH_SPEC.md` §3.1 稱 `publication_date`，在 `data_sources.md` 稱 `filing_date`；「期末日」一個叫 `fiscal_period_end`，一個叫 `period_end_date`。
   → 這不是規則衝突（兩份文件講的是同一個概念），純粹是命名不同步。本次 M2 已在 Task A 處理（做對照表），但值得之後找機會統一到兩份文件本身（由你決定要不要修訂，我不會自己動手）。

3. **`hypotheses.md`、`research_log.md`、`rejected_signals.md`、`README.md` 目前都是空檔案。**
   這不影響 M2 本身，但代表專案目前沒有「研究進度的敘事記錄」與「已經試過但不採用的訊號」的正式記錄地點。建議完成 M2 後，把第一筆 log 寫進 `research_log.md`（Task K3 已包含這個動作），`README.md` 則建議之後另外找時間補上專案簡介與如何使用 `ROADMAP.md`／`CURRENT_TASK.md` 的說明，但這不屬於本次 milestone 範圍，先不列進 Tasks 以免範圍蔓延。

---

## 完成這個 Milestone 的 Definition of Done（重新列一次，方便對照）

全部符合才算完成：

1. `data/schema/fundamentals_schema.md`（或等效路徑）存在，且每個欄位都有：名稱、型別、單位、nullable、缺失原因分類（如適用）、來源優先層級、對應到 `RESEARCH_SPEC.md` 章節出處。
2. 欄位涵蓋 `RESEARCH_SPEC.md` §3.1 全部日期／版本欄位、§4.2 全部財報欄位最低集合（含選用的營運資料欄位，並標記選用）。
3. 已完成 Task J 的反向檢查：S1–S5 與 §7.2 目標字典所需的欄位，schema 都能提供（含透過歷史滯後 t-1/t-4/t-5 取得的方式）。
4. 已完成命名對照表（Task A2），且沒有偷改 `RESEARCH_SPEC.md` 或 `data_sources.md` 本身的用字。
5. 上方「需要 Review」的項目已被記錄（抄到 `research_log.md`），不是被自己決定悄悄解決。
6. 已 commit 並 push。
7. `ROADMAP.md` 的 Phase 1 M2 狀態已更新為 Done。

做完以上七項，才進入下一個 Milestone（Phase 1 M3 — 小型人工驗證）。