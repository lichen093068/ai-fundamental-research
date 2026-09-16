
TWSE OpenAPI 目前列有「上市公司綜合損益表（一般業）」`/opendata/t187ap06_L_ci`，亦列有一般業資產負債表端點；官方 Swagger 頁將其歸在「財務報表」。[TWSE OpenAPI 財務報表目錄](https://openapi.twse.com.tw/)

**適用：** 每次執行擷取時的最新公開財報截面、欄位名稱／數值的程式化交叉核對。

**不應假設：**

- OpenAPI 回傳的 `出表日期` 或 HTTP 下載時間，就是某公司某季財報的首次申報日；
- 最新 API 數值代表歷史當時可得版本；
- 端點自行提供完整歷年、逐筆版本與公告時間歷史。

因此，若使用此端點，必須在 raw 層保存完整 JSON、請求 URL、擷取 UTC 時間、回應雜湊與端點 schema；並以 MOPS 主來源回填 `filing_date`。在未實測確認欄位與歷史行為前，不將它列為 point-in-time 主資料。

### 1.4 TWSE OpenAPI／MOPS：每月營業收入

官方 OpenAPI 列有「上市公司每月營業收入彙總表」`/opendata/t187ap05_L`；資料表包含公司、產業別、當月營收、上月營收、去年同月營收及累計營收等欄位。公開資料集的欄位說明亦確認其更新頻率為每月。[TWSE OpenAPI](https://openapi.twse.com.tw/)；[政府資料開放平臺的資料集說明](https://data.gov.tw/applications/135898)

**法定時效：** 發行人應在每月 10 日前公告並申報前一月份營運情形；這是月營收可早於季報取得的制度基礎。[證券交易法第 36 條](https://twse-regulation.twse.com.tw/TW/law/DOC01.aspx?FLCODE=FL007009&FLNO=36)

**可做：**

- 將同一曆月／會計季度的三個月月營收相加，作為季報營收的**輔助勾稽**；
- 記錄月營收首次公告日，供未來「較早公開營運資訊」研究使用；
- 找出季報與月營收加總不一致的公司，回查合併範圍、幣別、停牌、更正或資料版本。

**不可做：**

- 把三個月月營收加總標示為「季度財報營收」；
- 用 API 的批次 `出表日期` 取代公司原始月營收申報日期；
- 用月營收公告日當作季報可得日。

月營收對台灣上市櫃公司通常是合併營收的早期營運披露；與 IFRS 季報的收入認列、報告期間或更正狀態若有差異，應由原始公告與財報附註決定，不可自動判為資料錯誤。

### 1.5 重大訊息：時間戳補強來源

MOPS 提供當日／歷史重大訊息與公告查詢；TWSE OpenAPI 亦列有「上市公司每日重大訊息」`t187ap04_L`。[MOPS](https://mops.twse.com.tw/)；[TWSE OpenAPI](https://openapi.twse.com.tw/)

部分公司會發布「董事會通過某季合併財務報告」之重大訊息，且公告資料可能包含發言日期與時間。這可用於驗證日內時點或發現財報通過事件，但有兩項限制：

- 重大訊息可能早於、晚於或不完全等同正式財務報告／iXBRL 的申報完成時間；
- 重大訊息文字與標題不是可替代的結構化財報數據來源。

因此應建立 `mops_filing_date` 與 `material_announcement_datetime` 兩個不同欄位，優先以前者定義季度財報 availability，後者僅作審計軌跡與敏感度分析。

---

## 2. 季度財報公告期限：可用於 QC，不可代替實際公告日

法定一般期限是：年度終了後三個月內；第一、二、三季終了後四十五日內；上月營運情形於每月十日前公告申報。[證券交易法第 36 條](https://twse-regulation.twse.com.tw/TW/law/DOC01.aspx?FLCODE=FL007009&FLNO=36)

這些期限只能用於資料品質檢查，例如標示遲延申報或發現日期解析錯誤，**不能**把截止日當作所有公司的公告日。不同年度、實收資本額、第一上市公司或主管機關特別規定可能導致不同截止日；例如 TWSE 會逐期發布上市公司財報公告申報期限與適用群組。[TWSE 2026 Q1 公告](https://www.twse.com.tw/staticFiles/news/news/tsecnews/8a8216d69dbea9fd019e16a235e70191.pdf)

資料表應同時保留：

| 欄位 | 用途 |
|---|---|
| `period_end_date` | 財報實際涵蓋期間 |
| `statutory_deadline` | 依當期規則計算／記錄的最後申報期限，只作 QC |
| `filing_date` | MOPS 實際首次公告／申報日；主鍵日期 |
| `filing_time` | 有官方證據才填入 |
| `availability_date` | 研究使用的可得日；無時間時採下一交易日 |
| `source_document_id`、`raw_file_hash` | 可重現與版本追蹤 |

---

## 3. 建議的最小資料擷取設計

### 3.1 一筆「公司—季度—版本」的必要欄位

```text
company_id, ticker, market, industry_at_filing, fiscal_year, fiscal_quarter,
period_end_date, statement_scope, filing_date, filing_time,
availability_date, revision_flag, revision_date, source_document_id,
source_url, raw_file_hash, retrieved_at,
revenue, COGS, gross_profit, operating_income, net_income,
operating_cash_flow, capex_cash_paid, currency, unit_scale,
is_ytd_value, derived_quarter_value, data_quality_note
```

### 3.2 擷取與核對流程

1. 以 MOPS 財務報告公告查詢取得上市一般業公司的公司、年度、季別、首次申報日期與文件識別。
2. 下載同一申報版本的合併財報／iXBRL，擷取正式營收與其他報表欄位。
3. 保存原始檔、雜湊、擷取時間與欄位擷取規則；不覆寫更正前版本。
4. 以 MOPS 損益表或 TWSE OpenAPI 交叉核對最新數字；發生差異時以該期 MOPS 原始申報文件優先。
5. 以三個月月營收加總做營收合理性檢查；差異留下旗標與原始證據，不自動修改季報值。
6. 以法定截止日作日期 QC；若 `filing_date` 晚於截止日，確認是否為延期、適用不同規定、資料解析錯誤或更正申報。

---

## 4. 實作前必做的小型驗證（不是完整系統）

先取 10 家上市一般業公司，涵蓋不同產業、不同規模與至少 3 個年度；每家公司抽 Q1、Q2、Q3、Q4 各一筆。對每一筆人工驗證：

- MOPS 的財報公告紀錄能否識別首次申報日期；
- 該日期能否連到同季度、同口徑的合併財報／iXBRL；
- 損益表是否可找到營業收入，並識別單季或累計口徑；
- 月營收三個月加總與季報營收差異是否可解釋；
- 更（補）正是否能保留舊版與新版日期；
- OpenAPI 的欄位、`出表日期`、資料範圍與 MOPS 原始資料的關係是否符合本文假設。

若這六項有任何一項無法驗證，不應直接大量回補；應把無法驗證的來源降級為輔助，並記錄限制。

---

## 5. 目前可採用的資料來源結論

- **季度正式營收：可取得，可靠。** 採 MOPS 同季合併財報損益表／iXBRL。
- **季度財報公告日期：可取得，可靠。** 採 MOPS 財務報告公告紀錄的實際首次申報日期，保存原始證據。
- **精確公告時間：可能可取得，但不能預設完整。** 優先查 MOPS 原始紀錄；重大訊息時間可交叉驗證，不能自動取代正式申報時間。
- **月營收：可取得且較早，可靠。** 用於輔助、勾稽或另一個獨立研究問題，不取代季度財報。
- **TWSE OpenAPI：官方且適合自動化輔助。** 但在歷史 point-in-time、逐筆首次公告日與更正版本未完成實測前，不作 Phase 1 的唯一資料庫。

本盤點未修改 `RESEARCH_SPEC.md`，也未宣稱已完成歷史資料抓取或 API 可行性驗證。
篩選檔案
