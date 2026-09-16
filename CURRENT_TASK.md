# CURRENT_TASK.md
# 目前唯一要做的事情

**最後更新：** 2026-09-16
**目前所在：** Phase 1 — Data Engineering ／ **M3 — 小型人工驗證（Pilot Validation）**
**上一個 milestone：** M2 — 建立標準財報資料 Schema ✅ Done（`data/schema/fundamentals_schema.md`）
**這份文件的角色：** 只放「現在」要做的事。長期規劃看 `ROADMAP.md`，不要把長期任務搬來這裡。

---

## 這個 Milestone 要交付什麼

用 M2 產出的 schema，人工核對 **10 家上市一般業公司 × 4 季（Q1–Q4）＝ 40 筆觀測**，證實 `data_sources.md` 的資料來源假設與 `fundamentals_schema.md` 的欄位設計在真實資料上站得住腳。這是**在大規模回補歷史資料之前**的必要檢查，不是先斬後奏。

**明確不做的事（避免範圍蔓延）：**
- 不做大規模歷史回補（那是 M4 的事）
- 不寫自動化擷取程式（M3 是人工核對，允許用試算表或筆記，不需要寫爬蟲）
- 不建資料庫
- 不驗證上櫃（TPEx）公司——本次 M1 資料來源盤點只涵蓋 TWSE 上市，上櫃驗證要等資料來源盤點補齊後才有意義（見下方 Review 沿用項目）

---

## Tasks（照順序做，做完打勾）

### A. 選樣
- [ ] A1. 選 10 家 TWSE 上市一般業公司，條件：涵蓋不同產業（至少 5 個不同產業別）、涵蓋不同規模（大型權值股與中小型股都要有，避免只選好查的大公司）、涵蓋至少 3 個不同年度
- [ ] A2. 記錄選樣理由與清單（公司代號、名稱、產業、選樣年度範圍），存成 `research/pilot_validation_sample.md` 或等效檔案，讓之後審視時知道樣本不是隨機湊的也不是刻意挑好查的

### B. 逐筆核對（每家公司 Q1–Q4 各一筆，共 40 筆）
對每一筆觀測，依 `data_sources.md` §4 的驗證清單逐項確認：

- [ ] B1. 在 MOPS 財務報告公告查詢中，能否識別該季財報的**首次**申報日期（不是最新版本日期）
- [ ] B2. 該筆公告能否連到同一季度、同一口徑（合併／個別）的財報書／iXBRL
- [ ] B3. 損益表中能否找到營業收入，並清楚識別是**單季值**還是**年初至今累計值**（對應 schema 的 `is_ytd_value`）
- [ ] B4. 若為累計值，能否找到前一期可比累計值以執行 `Q(t) = YTD(t) - YTD(t-1)`，且兩期口徑一致（合併範圍、會計政策未變）
- [ ] B5. 損益表能否取得 `operating_expenses`、`EPS`、`income_tax`、`non_operating_items`（這 4 個是 M2 判斷 `data_sources.md` 遺漏、依 `RESEARCH_SPEC.md` §4.2 補進去的欄位，**必須實測，不能延用假設**）
- [ ] B6. 資產負債表 8 個欄位（`cash_and_equivalents`、`total_debt`、`net_debt`、`accounts_receivable`、`inventory`、`deferred_revenue`、`shareholders_equity`、`shares_outstanding`）能否從同一份財報取得（M2 已標記此為未驗證風險，這裡是第一次實測）
- [ ] B7. 月營收三個月加總與季報營收的差異是否可解釋（合理誤差 vs. 需要旗標的異常）
- [ ] B8. 從 40 筆中找出至少 1 筆更正／重編案例，確認舊版與新版的 `filing_date`、`revision_date`、數值能否同時保留、不互相覆蓋
- [ ] B9. 用 TWSE OpenAPI 抓一筆同公司同季資料，比對欄位與數值是否與 MOPS 原始資料一致；記錄 API 的 `出表日期` 與 MOPS `filing_date` 的實際差距（用來判斷 M1 的判斷——不能拿 API 出表日當 filing date——是否成立）

### C. 記錄結果
- [ ] C1. 每一筆觀測記錄：是否通過 B1–B9、卡在哪一步、原因是什麼（介面問題／欄位真的沒揭露／需要人工判讀）
- [ ] C2. 彙總 40 筆的通過率，特別標出資產負債表欄位（B6）與 4 個新補欄位（B5）的通過率——這兩組是本次驗證的重點，因為 M2 判斷它們有較高的不確定性
- [ ] C3. 若任一驗證項目在多數樣本中失敗，依 `data_sources.md` 的原則處理：**降級為輔助來源、記錄限制，不直接大量回補**——這個決定請你自己下，我不會替你判斷「多少比例算失敗」

### D. 收尾
- [ ] D1. 若驗證結果與 M2 schema 的假設有落差（例如某欄位實際上取不到、或需要調整型別／單位），**回頭修改 `data/schema/fundamentals_schema.md` 本身**（這是允許的，因為這是我們自己的工作文件，不是 `RESEARCH_SPEC.md` 或 `data_sources.md`），並在文件變更處註明修改日期與原因
- [ ] D2. 把驗證結果摘要（通過率、發現的問題、schema 是否有調整）寫進 `research_log.md`
- [ ] D3. 若驗證發現 Review 項目中的「上市/上櫃」「資產負債表可得性」等問題有新證據，一併更新 Review 記錄的狀態（不是刪除，是補充「已驗證：結果是 X」）
- [ ] D4. `git add` → `git commit`（訊息建議：`Phase1 M3: pilot validation of fundamentals schema (10 companies x 4 quarters)`）→ `git push`
- [ ] D5. 回到 `ROADMAP.md`，把 Phase 1 M3 狀態改成 ✅ Done（或若失敗率高，改成「⚠️ Blocked，見 research_log」），並依驗證結果決定下一步是 M4（資料管線）還是要先回頭處理資料來源缺口

---

## 沿用中、尚未解決的 Review 項目

1. **Universe 範圍與資料來源不對等**（上市 vs 上市+上櫃）——本次 M3 **不驗證上櫃公司**，因為 `data_sources.md` 從未涵蓋上櫃資料來源；這個缺口仍然存在，尚未處理。
2. **資產負債表欄位可得性未經驗證**——本次 M3 的 B6 就是第一次實測，結果請記錄進 D2/D3。
3. **`net_debt` 定義未固定**——若 M3 發現公司沒有直接揭露 `net_debt`，需要衍生計算，公式版本必須在看任何目標結果前鎖定（比照 `free_cash_flow` 處理方式），這件事本身也記錄為待辦，不要在 M3 匆忙決定公式後就直接用於後續分析。

---

## 完成這個 Milestone 的 Definition of Done

全部符合才算完成：

1. 40 筆觀測（10 家公司 × 4 季）已依 B1–B9 逐項人工核對並記錄結果。
2. 通過率彙總已產出，且特別標出資產負債表欄位與 4 個新補損益表欄位的通過情形。
3. 至少 1 筆更正／重編案例已驗證，證實新舊版本可以同時保留。
4. 若 schema 需要因為實測結果調整，`fundamentals_schema.md` 已更新並註明修改原因。
5. 驗證結果（含通過率、問題、決策）已寫入 `research_log.md`。
6. Review 項目狀態已更新（不是解決，是補充驗證證據）。
7. 已 commit 並 push。
8. `ROADMAP.md` 的 Phase 1 M3 狀態已更新。

做完以上八項，才進入下一個 Milestone（Phase 1 M4 — 歷史資料回補與資料管線），**除非驗證結果顯示某個來源假設不成立，此時應先處理該缺口，而不是硬著頭皮進 M4**。
