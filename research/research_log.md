# research_log.md

> 本檔案目前為空，以下是第一筆記錄的草稿。若你的 repo 已有既定的 log 格式，請改用你自己的格式，這裡只提供最小可用版本。

---

## 2026-09-16 — Phase 1 M2：建立標準財報資料 Schema

**做了什麼：**
- 比對 `RESEARCH_SPEC.md`（§3.1、§4.2）與 `data_sources.md`（§3.1）的欄位需求，產出命名對照表。
- 判斷 `data_sources.md` 的欄位清單為「最小擷取設計草案」，不能取代 `RESEARCH_SPEC.md` §4.2 的正式最低欄位集合；schema 最終以 §4.2 為準，`data_sources.md` 只採用其版本追蹤欄位（`raw_file_hash`、`retrieved_at` 等）。
- 產出 `data/schema/fundamentals_schema.md`，涵蓋主檔、日期/版本、損益表、現金流量表、資產負債表、營運資料（選用）、品質/缺失共 7 類欄位。
- 用 S1–S5 訊號公式與 §7.2 目標字典逐一反向檢查欄位是否足夠，結論：全部訊號與目標可用該 schema 計算；S2B（產品層）為條件通過，資料可得性風險留給 M3 驗證。

**發現，需要 Review（未修改任何規則文件，只記錄）：**
1. **Universe 範圍與資料來源不對等**：`RESEARCH_SPEC.md` §2.1 母體定義含「上市與上櫃」，`data_sources.md` 範圍聲明只涵蓋 TWSE 上市，全文未提及上櫃（TPEx）資料來源。Schema 中 `market` 欄位已預留 `TPEX_LISTED` 占位，但**在資料來源盤點補齊上櫃之前，不應對上櫃公司做任何資料回補假設**。
2. `RESEARCH_SPEC.md` 與 `data_sources.md` 部分欄位命名不同步（`filing_date` vs `publication_date`、`period_end_date` vs `fiscal_period_end` 等），已在 schema 文件的命名對照表記錄，未修改任一原始文件。
3. 資產負債表 8 個欄位（`cash_and_equivalents`、`total_debt`、`net_debt`、`accounts_receivable`、`inventory`、`deferred_revenue`、`shareholders_equity`、`shares_outstanding`）的來源可行性目前完全沒有經過 `data_sources.md` 那樣的研究，只是依 `RESEARCH_SPEC.md` 要求收錄，尚未驗證是否能穩定從 MOPS 取得。

**狀態：** Phase 1 M2 完成，進入 Phase 1 M3（小型人工驗證，10 家公司 × 4 季 = 40 筆觀測）。

**下一步：** 見 `CURRENT_TASK.md`。
