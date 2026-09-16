# ROADMAP.md
# AI × Finance 台股基本面研究專案 — 長期路線圖

**文件角色：** 長期規劃參考。不是本週待辦，待辦請看 `CURRENT_TASK.md`。
**權威來源：** 若本文件與 `RESEARCH_SPEC.md` 衝突，一律以 `RESEARCH_SPEC.md` 為準；本文件不得修改研究規則，只安排「先做什麼、後做什麼」。
**更新規則：** 每完成一個 Milestone，回來這裡把狀態改成 `Done`，並在 `research_log.md` 記一筆，再開下一個 Milestone 對應的 `CURRENT_TASK.md`。

---

## 進度總覽

| Phase | 名稱 | 狀態 |
|---|---|---|
| 0 | Research Specification | ✅ Done |
| 1 | Data Engineering | 🟡 In Progress（M1、M2 Done，M3 進行中） |
| 2 | Fundamental Signal Research | ⬜ Not Started |
| 3 | Statistical Validation | ⬜ Not Started |
| 4 | Market Validation | ⬜ Not Started（對應 RESEARCH_SPEC 所稱的「Phase 2」） |
| 5 | MVP / Automation | ⬜ Not Started |
| 6 | Monetization | ⬜ Not Started |

> 注意命名差異：`RESEARCH_SPEC.md` 內部把「基本面研究」稱為它自己的 *Phase 1*、「市場反應」稱為它自己的 *Phase 2*。本 ROADMAP 用 Phase 0–6 管理整個專案生命週期，兩者編號系統不同，避免對照時混淆：本 Roadmap 的 **Phase 2+3** 合起來才等於 `RESEARCH_SPEC.md` 的 *Phase 1*；本 Roadmap 的 **Phase 4** 才對應 `RESEARCH_SPEC.md` 的 *Phase 2*。

---

## Phase 0 — Research Specification

**狀態：✅ Done**

### Objective
定義研究問題、母體、訊號、目標、統計方法與禁止事項，讓後續所有工作有唯一依據。

### Milestones
- M1：`RESEARCH_SPEC.md v0.1` 完成並鎖定 ✅

### Tasks
- [x] 定義核心研究問題與 H0/H1
- [x] 定義 Universe、納入／排除規則
- [x] 定義 point-in-time 規則與日期欄位
- [x] 定義 S1–S5 五大訊號
- [x] 定義 Phase 1 目標字典（Y_*_h）
- [x] 定義偏誤防制、訓練/驗證/測試切分原則
- [x] 定義可重現性與交付規格

### Definition of Done
`RESEARCH_SPEC.md` 已存在、有版本號，且被視為單一權威來源。

### Dependencies
無（起點文件）。

### 主要風險
規格本身若有內部矛盾或遺漏（例如 Universe 範圍與資料可得性不一致），會在後續階段才被發現，屆時只能新增修訂版本，不能回頭偷改 v0.1。→ 目前已發現一項，見 `CURRENT_TASK.md` 的「需要 Review」區塊。

---

## Phase 1 — Data Engineering

### Objective
把「公開、可稽核、point-in-time 正確」的台股財報資料，變成可重複產出的資料層（raw → processed → features/targets），且不違反 `RESEARCH_SPEC.md` 第 3、4、13 節的規則。

### Milestones

**M1 — 資料來源盤點與決策**
狀態：✅ Done（`research/data_sources.md`）

- Tasks:
  - [x] 盤點台股季度財報候選資料來源
  - [x] 決定 MOPS 財務報告公告為 point-in-time 主來源
  - [x] 決定 TWSE OpenAPI 的定位（輔助，非歷史 point-in-time 主庫）
  - [x] 決定月營收、重大訊息的定位（輔助勾稽，不可取代季報）
- DoD：`data_sources.md` 對「季度營收」與「公告可得日期」給出明確、有法源依據的資料來源決策，並列出實作前必做的小型驗證清單。
- Dependencies：Phase 0 完成。
- 主要風險：**範圍缺口** — 本文件只涵蓋 TWSE 上市，未涵蓋 `RESEARCH_SPEC.md` 母體定義中的「上櫃」公司資料來源（見 CURRENT_TASK 的 Review 項目）。若不補齊，Universe 定義與可用資料會長期不一致。

**M2 — 建立標準財報資料 Schema**
狀態：✅ Done（`data/schema/fundamentals_schema.md`）

- Tasks: 見 `CURRENT_TASK.md` 歷史版本（已完成，不重複列出）
- DoD：產出一份資料字典（data dictionary，純文件，非程式碼、非資料庫），欄位涵蓋 `RESEARCH_SPEC.md` §3.1 與 §4.2 的最低集合，且已用 S1–S5 公式與 §7.2 目標字典逐一核對「算得出來」。✅ 全部符合。
- Dependencies：M1 完成。
- 主要風險（已驗證發生）：`data_sources.md` §3.1 的精簡清單確實漏掉了 `operating_expenses`、`EPS`、`income_tax`、`non_operating_items`、全部資產負債表欄位；已依 `RESEARCH_SPEC.md` §4.2 補齊，記錄於 schema 文件第 0 節命名對照表。

**M3 — 小型人工驗證（Pilot Validation）**
狀態：🟡 進行中（目前 milestone，詳見 `CURRENT_TASK.md`）

- Objective：在大量回補歷史資料前，先用少量樣本人工證實 M2 的 schema 與 M1 的資料來源假設真的可行。
- Tasks（依 `data_sources.md` §4 的六項驗證清單）：
  - [ ] 選 10 家上市一般業公司，涵蓋不同產業、不同規模、至少 3 個年度
  - [ ] 每家抽 Q1–Q4 各一筆，共 40 筆觀測
  - [ ] 人工確認 MOPS 財報公告可識別首次申報日期
  - [ ] 人工確認可連到同季度、同口徑的合併財報／iXBRL
  - [ ] 人工確認損益表可找到營業收入，並識別單季或累計口徑
  - [ ] 人工核對月營收三個月加總與季報營收差異是否可解釋
  - [ ] 找至少 1 筆更正／重編案例，確認新舊版本與日期可並存
  - [ ] 確認 TWSE OpenAPI 欄位、`出表日期` 與 MOPS 原始資料的關係
- DoD：40 筆觀測全部人工核對完成並記錄結果；若任一驗證項目失敗，該資料來源在 `data_sources.md` 中降級為輔助並記錄限制（不得直接大量回補）。
- Dependencies：M2 完成（要有 schema 才知道驗證時要填哪些欄位）。
- 主要風險：人工樣本太小或選樣有偏（例如都選大型權值股），無法代表未來大規模回補會遇到的例外狀況（更正、合併範圍改變、停牌等）。

**M4 — 歷史資料回補與資料管線（Raw → Processed → Features/Targets）**
狀態：⬜ Not Started

- Objective：依 `RESEARCH_SPEC.md` §13.1 的資料分層要求，建立可重跑、不可覆寫的資料層。
- Tasks（草案，待 M3 結果調整）：
  - [ ] 建立 `data/raw/`：原始文件與其雜湊、擷取時間
  - [ ] 建立 `data/processed/`：標準化欄位、品質旗標、轉換紀錄
  - [ ] 建立 `data/features_as_reported/`：逐公司—季度訊號與 `as_of_date`
  - [ ] 建立 `data/targets/`：未來目標，與特徵分開生成
  - [ ] 實作 §13.3 的 8 條程式不變量測試（availability_date ≤ as_of_date 等）
  - [ ] 建立 `configs/`：來源對照、產業分類對照、已鎖定參數
- DoD：可對任一歷史日期重新產出當時「應該看到」的資料集，且自動測試涵蓋 §13.3 全部 8 條不變量。
- Dependencies：M2、M3 完成。
- 主要風險：這是全專案工程量最大的階段，容易被拆得太粗；建議之後再依「上市」「上櫃」「更正處理」「YTD 拆分」等拆成子 milestone，屆時再寫對應的 `CURRENT_TASK.md`，現在先不展開成任務清單，避免一次規劃太遠。

**M5 — 資料品質與覆蓋率報告**
狀態：⬜ Not Started

- Objective：在進入訊號研究前，先量化「這個資料集到底能不能撐起 Phase 2/3」。
- Tasks（草案）：
  - [ ] 產出樣本流統計（母體數、可用樣本數、各欄位缺失數、退出公司數）
  - [ ] 產出產業與年度覆蓋率報告
  - [ ] 確認存續者／退出者比例是否合理（§9.2 survivorship bias 檢查）
- DoD：報告顯示的缺失與覆蓋率讓 Phase 2 可以誠實地知道能檢驗哪些訊號、不能檢驗哪些。
- Dependencies：M4 完成。
- 主要風險：如果到這步才發現資料覆蓋率太差（例如某些產業樣本數過小），代表要退回 M1/M3 重新評估來源，而不是硬做下去。

### Phase 1 整體 Definition of Done
`RESEARCH_SPEC.md` §15 定義的完成條件：交付「可重跑、無時間穿越（look-ahead）、完整揭露失敗與限制」的基本面資料與訊號結果——不是「找到漂亮訊號」。

### Phase 1 整體 Dependencies
Phase 0 鎖定。

### Phase 1 主要風險
- Point-in-time 資料建置在台灣市場屬於非標準工程，MOPS 介面／欄位可能無公開穩定的程式化存取保證，需要人工或半自動流程，工時容易低估。
- 上市／上櫃資料來源目前不對等（見 Review 項目），若不及早解決，會在 M4 才發現 Universe 做不全。

---

## Phase 2 — Fundamental Signal Research

### Objective
依 `RESEARCH_SPEC.md` §6 的正式定義，用 Phase 1 產出的資料計算 S1–S5（及其子項），並確認訊號本身的資料品質與描述統計正常，尚不做因果或顯著性宣稱。

### Milestones（草案，待 Phase 1 完成後展開細節）
- M1：實作 S1 Revenue Growth Acceleration、S2A Cost Ratio 代理值
- M2：實作 S3 Gross/Operating Margin Delta
- M3：實作 S4 Earnings→Cash Conversion（TTM 版本）
- M4：S2B（產品層）與 S5（component panel，非 composite）視資料揭露程度決定是否可行
- M5：訊號描述統計報告（§11.1 要求的觀測數、缺失比例等）

### Definition of Done
每個訊號都能用 as-reported 資料重現計算，且描述統計（含缺失比例、退出公司比例）已產出並檢視過，沒有明顯的計算錯誤或資料錯置。

### Dependencies
Phase 1 全部完成（尤其是 M4 資料管線與 M5 品質報告）。

### 主要風險
- S2B、S5 高度依賴公司主動揭露的產品層資料，台股普遍揭露率可能偏低，屆時可能大部分公司無法計算，需要誠實記錄樣本涵蓋率而非勉強湊數。
- 容易提早偷看目標資料來「調整」訊號定義，違反 §9.3 data snooping 規則；此階段任何訊號定義修改都必須發生在完全不看目標／測試結果的前提下。

---

## Phase 3 — Statistical Validation

### Objective
依 `RESEARCH_SPEC.md` §10、§11、§14，在鎖定 `ANALYSIS_PLAN.md` 之後，對訊號—目標配對做預先指定的統計檢驗。

### Milestones（草案）
- M1：撰寫並鎖定 `ANALYSIS_PLAN.md`（train/validation/test 切點、horizon embargo、多重比較校正、產業分組規則等 §14 所列未定參數）— **必須在看任何 test 結果前完成**
- M2：Train/Validation 階段檢驗（分位組、Spearman IC、panel 回歸）
- M3：Test 階段一次性最終評估
- M4：反偏誤與穩健性檢查報告（§9.4）

### Definition of Done
`ANALYSIS_PLAN.md` 已鎖定且早於任何 test 結果被查看；test 結果只執行一次；所有預註冊檢驗（含失敗的）都被報告，沒有選擇性呈現。

### Dependencies
Phase 2 完成，且 Phase 1 資料層已凍結（不再回填）。

### 主要風險
- 這是全案最容易「不小心」違反預註冊原則的階段（想再多試一種切分、多試一種校正方法）。一旦看過 test 結果，該次即視為 exploratory，需要全新未接觸樣本才能恢復 confirmatory 地位——代價很高，值得在 M1 多花時間把 `ANALYSIS_PLAN.md` 想清楚。
- 若樣本或有效期數不足以支撐推論，必須誠實報告為「不可判斷」，而不是硬生出 p-value。

---

## Phase 4 — Market Validation（對應 RESEARCH_SPEC 的「Phase 2」）

### Objective
在 Phase 1–3（即 `RESEARCH_SPEC.md` 的 Phase 1）完全鎖定後，才研究基本面訊號是否具有市場資訊價值。**此階段需要另立獨立規格文件**，處理價格資料、交易成本、公告時點對齊、可交易性、停牌、公司行動、市場基準等問題，依 `RESEARCH_SPEC.md` §15 規定不得回頭改寫 Phase 1 規則。

### Milestones（僅列大方向，細節待新規格文件）
- M1：撰寫 `MARKET_VALIDATION_SPEC.md`（新文件，比照 RESEARCH_SPEC 的嚴謹度）
- M2：股價與市場資料來源盤點（比照 `data_sources.md` 的方法論）
- M3：訊號公告後市場反應的統計檢驗
- M4：交易成本與可交易性的現實性檢查

### Definition of Done
待 M1 產出的新規格文件自行定義；本 ROADMAP 不预先假設市場驗證的成功標準。

### Dependencies
**硬性依賴：** Phase 1–3（RESEARCH_SPEC 所稱的 Phase 1）必須完全鎖定，包括資料管線、訊號定義、結果與變更紀錄。

### 主要風險
- 提早進入這階段是整份規格明文禁止的（§15）；最大風險是團隊在基本面訊號還沒驗證完就想「順便看看股價」。
- 需要引入台股價格資料、除權息調整、停牌處理等全新複雜度，不能沿用 Phase 1 的資料假設。

---

## Phase 5 — MVP / Automation

### Objective
在確認訊號具有統計上有意義且經濟上可解釋的預測力（Phase 3）、且（若要往市場應用）通過市場驗證（Phase 4）後，才建立最小可行的自動化流程，讓研究結果可以持續、低成本地重跑。

### Milestones（僅列大方向）
- M1：資料擷取自動化（在 Phase 1 手動/半自動流程證實可行後才自動化）
- M2：訊號計算與報告自動排程
- M3：異常與資料品質告警

### Definition of Done
自動化流程產出的結果與人工／半自動流程在同一組歷史資料上一致（regression test 通過）。

### Dependencies
Phase 3（若涉及市場相關功能則含 Phase 4）完成且結論穩定。

### 主要風險
- 過早自動化會把還在變動的研究邏輯寫死，之後每次規則調整都要重構自動化系統；因此本階段刻意排在統計驗證之後。
- 不引入 Docker / AWS / PostgreSQL 等複雜 infrastructure 的原則，在此階段也要重新檢視是否真的有必要（依當時資料量與重跑頻率判斷），而不是預設要上雲端或上資料庫。

---

## Phase 6 — Monetization

### Objective
只有在研究結論穩定、自動化可靠之後，才評估任何商業化或產品化方向。

### Milestones（僅列大方向，故意不展開）
- M1：評估潛在應用場景與合規／法遵限制（投資建議相關的揭露與免責）
- M2：若決定推進，另立產品規格文件

### Definition of Done
待需要時再定義；本階段目前只是佔位，避免研究尚未成立就規劃商業化。

### Dependencies
Phase 3（及視需要 Phase 4、Phase 5）全部完成且結論穩健。

### 主要風險
- 最大風險是本末倒置：在研究結論還不穩定時就開始想商業模式，導致回頭修改研究假設去遷就商業敘事——這正是 `RESEARCH_SPEC.md` 全篇試圖防範的行為模式。

---

## 使用這份 ROADMAP 的方式

1. 每次要知道「現在該做什麼」→ 看 `CURRENT_TASK.md`，不要看這份文件。
2. 每次要知道「這個 milestone 做完之後接下來是什麼」→ 回來這份文件找下一個 Milestone。
3. 完成一個 Milestone 後：
   - 把這份文件裡對應的狀態改成 `✅ Done`
   - 在 `research_log.md` 記一筆（做了什麼、發現什麼、有沒有需要 review 的問題）
   - 依下一個 Milestone 重寫 `CURRENT_TASK.md`
   - `git commit` → `push`
4. 不要因為某個 Phase「看起來很近」就跳過中間 Milestone 直接展開遠期任務；遠期 Phase（2–6）目前只列大方向，細節留到真正抵達時才展開，避免規劃跟實際資料狀況脫節。
