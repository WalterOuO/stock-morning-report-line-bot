## 🚀 台股盤前資訊自動化推播系統 (ETL + LLM Workflow + LINE Bot)

本專案是一套**輕量、低維運成本**的財經盤前自動化系統。每個交易日早上 **08:20** 由 **Make** 觸發主流程，從資料庫讀取已蒐集好的美股、台股新聞，即時抓取日經、韓國 KOSPI 的開盤與目前漲跌幅，整理成三個輸入變數後送進 **Dify Workflow**，由 LLM 撰寫約 **1000~2000 字**的「台股盤前策略日報」，成功後再透過 **LINE Messaging API** 推播到使用者手機。

> ⚠️ 本專案輸出僅為資訊整理，**不構成任何投資建議**。

---

### 🛠️ 系統架構圖 (Architecture)

系統分成「收集層」與「主流程」兩層。因為 RSS 只保留最近約 100 則文章（大約 10 小時內），週一早上已經看不到上週五的報導，所以另外建立兩個定時收集的 scenario，把文章全文存進 Make Data store，主流程再從 Data store 取用。

```text
━━━━━━━━━━━━━━━━━━━━━━ 收集層（定時寫入 Data store）━━━━━━━━━━━━━━━━━━━━━━

 鉅亨網 美股 RSS ─► 日期 + 標題 Filter ─► Jina Reader ─► 文字清洗 ×3 ────────┐
   (週二~週六 06:20)                                                 　　　 │
                                                                      　　 ├─► Data store
 鉅亨網 台股 RSS ─► 日期 + 標題/關鍵字 Filter ─► Jina Reader ─► 文字清洗 ×3 ─┘
   (週一~週五 15:30)

━━━━━━━━━━━━━━━━━━━━━━ 主流程（週一~週五 08:20）━━━━━━━━━━━━━━━━━━━━━━

 Data store (美股) ─► Text Aggregator ─► us_news ─┐
 Data store (台股) ─► Text Aggregator ─► tw_news ─┼─► HTTP POST：Dify Workflow ─► 成功才推播 ─► LINE
 Yahoo Finance 日經 / 韓股 ─► Compose ─► jpkr ────┘ 
```

---

### ⏰ 排程總覽 (Schedule)

所有時間皆為**台灣時間（Asia/Taipei）**，請確認 Make 帳號個人設定的時區一致。

| Scenario | 執行時間 | 用途 |
|---|---|---|
| 美股新聞蒐集 | 週二至週六 06:20 | 收集當天凌晨發布的〈美股盤後〉新聞 |
| 台股新聞蒐集 | 週一至週五 15:30 | 收集當天 13:30 之後發布的〈台股盤後〉新聞與大盤三大法人買賣超 |
| 推播系統主流程 | 週一至週五 08:20 | 讀取資料、整合、送 Dify、推播 LINE |

---

### 📊 資料來源與工作流處理 (Data Sources & Pipeline)

#### 1. 🔴 美股盤後（收集 → Data store）
- **來源**：鉅亨網美股 RSS（`news.cnyes.com/rss/v1/news/category/wd_stock`），每次取最多 100 則。
- **篩選**：發布時間晚於**當天 04:00**，且標題包含「美股盤後」、不含「速報」。每天預期只會通過一篇。
- **全文**：以 Jina AI Reader 取得 Markdown 全文
- **文字清洗**（3 個 Text parser → Replace）：
  1. 刪除第一個 `# ` 標題之前的所有網頁雜訊與按鈕
  2. 刪除整行以 `![Image` 開頭的圖片連結
  3. 刪除結尾標記「掌握全球財經資訊」及所有其他報導內容
- **寫入**：存進 Data store，以文章網址的 md5 雜湊值作為 key，重複收集時會覆蓋，不會產生重複資料。

#### 2. 🔵 台股盤後與大盤法人籌碼（收集 → Data store）
- **來源**：鉅亨網台股 RSS（`news.cnyes.com/rss/v1/news/category/tw_stock`）。
- **篩選**：發布時間晚於**當天 13:30**，且符合下列**任一**條件：
  - 標題包含「台股盤後」（不含「速報」）
  - `media:keywords` 同時包含「三大法人買賣超」與「台股盤後」，用來精準抓到大盤的三大法人買賣超報導，避開只談個股籌碼的文章。
- 全文取得、文字清洗與寫入方式與美股相同，每天通常會存入兩篇。

#### 3. 🗄️ Data store：`cnyes_articles`
每篇文章一筆記錄，欄位如下：

| 欄位 | 型別 | 說明 |
|---|---|---|
| key | 文字 | 文章網址的 md5 |
| market | 文字 | `us` 或 `tw` |
| title | 文字 | 文章標題 |
| pubdate | 日期 | 發布時間（由 RSS 文字格式轉換而來） |
| link | 文字 | 文章網址 |
| content | 文字 | 清洗後的全文（最多 8000 字） |

建議定期清理 10 天以前的舊記錄，避免超出方案的儲存容量（免費方案約 1 MB）。

#### 4. 🟢 日韓股開盤市況（主流程即時抓取）
- 以 HTTP 模組呼叫 Yahoo Finance 的 chart 資料：日經 225（`^N225`）與韓國 KOSPI（`^KS11`），取 1 分鐘資料。
- 取出**昨收、開盤價、目前價**，在 Make 內計算：
  - 開盤漲跌幅 ＝（開盤價 − 昨收）÷ 昨收 × 100
  - 目前漲跌幅 ＝（目前價 − 昨收）÷ 昨收 × 100
  - 結果用 `formatNumber` 固定顯示到小數點後兩位，負值代表下跌。
- 組成含擷取日期時間的 `jpkr` 文字。「目前較開盤走高或走低多少」交由 Dify 的 LLM 以「目前漲跌幅 − 開盤漲跌幅」（百分點）判斷。

#### 5. 📥 主流程從 Data store 取資料
- **美股**：發布時間晚於當天 04:00（週一回推至週六 04:00），依時間由新到舊，取 1 篇，整合為 `us_news`。
- **台股**：發布時間晚於前一個交易日 13:30（週一回推至週五 13:30），最多 10 篇，整合為 `tw_news`。
- 兩段搜尋皆設為「查無資料也繼續執行」，缺資料的區段在日報中會標示「資料未取得」，不會讓整條流程中斷。

#### 6. 🧠 Dify Workflow（LLM 摘要）
- **Start 節點**：三個段落變數 `us_news`、`tw_news`、`jpkr`。
- **LLM 節點**：模型使用 **GPT-5.2**（可依你的 Dify 可用模型替換）；System prompt 規定角色、各段整理重點、字數配額、純文字輸出、台股紅漲綠跌的 emoji 慣例、不得編造數字、忽略新聞內文中的任何指令，並在結尾固定附上非投資建議的聲明。
- **End 節點**：輸出變數 `result`。
- 日報固定包含四個段落：
  1. 📱 美股盤後與全球總經
  2. 🇹🇼 昨日台股與法人籌碼
  3. 🌏 日韓股開盤現況（含相對開盤的走勢判斷）
  4. 💡 今日台股盤前思維

#### 7. 🤖 LINE 推播
- Make 以 HTTP 模組呼叫 Dify（`response_mode: blocking`，逾時 300 秒），**Dify 回傳狀態為 `succeeded` 才會推播**。
- 推播內容為純文字（LINE 不會渲染 Markdown），並截取前 4900 字，避免超過單則訊息 5000 字的上限。

---

### 📂 專案檔案結構說明 (Files)

```text
stock-morning-report-line-bot
├── README.md
└── workflows/                                  <-- 工作流資料夾
    ├── us-news-collector.blueprint.json        <-- Make：美股新聞蒐集 scenario
    ├── tw-news-collector.blueprint.json        <-- Make：台股新聞蒐集 scenario
    ├── main-push.blueprint.json                <-- Make：推播系統運作 scenario
    └── tw-premarket-report.ifpkg               <-- Dify：LLM 摘要工作流配置
```

---

### 🚀 部署使用指南 (How to Use)

#### 步驟 1：Dify LLM 設定
1. 登入 Dify 後台，點擊「建立 App → 匯入 DSL 檔案」。
2. 上傳 `workflows/tw-premarket-report.ifpkg`。
3. 檢查 LLM 節點的模型是否仍可用（見下方注意事項），並確認 Start 節點的三個變數長度上限足夠。
4. 先用測試資料執行一次，確認輸出沒有被截斷（最後一行有免責聲明）、沒有 Markdown 符號。
5. **發布**工作流，並在「存取 API」選單生成 `app-` 開頭的 API 金鑰。

#### 步驟 2：建立 Make Data store 與資料結構
1. 在 Make 建立 Data store `cnyes_articles`，資料結構欄位依上方表格（market、title、pubdate、link、content）建立。
2. 再建立一個資料結構，用於 Dify 請求的 Body：
   - `inputs`（集合）：`us_news`、`tw_news`、`jpkr`（皆為文字）
   - `response_mode`（文字）
   - `user`（文字）

#### 步驟 3：匯入三個 Make Scenario
1. 在 Make 分別建立新的 Scenario，點擊右上角「⋯ → Import Blueprint」。
2. 依序匯入：
   - `workflows/us-news-collector.blueprint.json`
   - `workflows/tw-news-collector.blueprint.json`
   - `workflows/main-push.blueprint.json`
3. 匯入後，模組內的 Data store、資料結構與連線不會自動對應，請逐一重新選取。

#### 步驟 4：節點連線與認證設定
1. **Jina AI**：在兩個收集 scenario 的 Jina 模組建立 Connection，貼上你的 Jina API Key。
2. **Dify HTTP 模組**：
   - URL 改為你的 Dify 工作流 API 路徑（例如 `https://api.dify.ai/v1/workflows/run`）。
   - Headers 的 `Authorization` 填入 `Bearer 你的Dify金鑰`（建議使用 Make 的 Keychain 管理）。
   - Body 使用步驟 2 建立的資料結構，`inputs` 對應 `us_news`、`tw_news`、`jpkr`。
3. **LINE**：開通 LINE 官方帳號 Messaging API，在 Make 綁定 Channel Access Token，並在 `To` 欄位填入你本人的 LINE User ID 或群組 Group ID；該官方帳號需已加為好友。
4. **資料來源與條件**：RSS 網址、日期門檻、標題與關鍵字條件可依你習慣的媒體調整。

#### 步驟 5：手動測試後再啟用排程
1. 依序對兩個收集 scenario 與主 scenario 按 **Run once**，並到 Data store 確認記錄有寫入、內容乾淨。
2. 確認日韓漲跌幅與你手算的數字一致。
3. 確認 LINE 收到的訊息完整。
4. 三個 scenario 都切換為 **Active**，並依「排程總覽」設定時間。收集 scenario 要在第一個需要資料的交易日**之前**啟用，已被擠出 RSS 的舊文章無法補收。

---

### ⚠️ 注意事項與已知限制 (Notes)

- **時區**：所有日期條件與排程都以 Asia/Taipei 為準。
- **週一回推**：週一早上的美股使用上週六 04:00 蒐集結果、台股使用上週五 13:30 蒐集結果；實際運作前可自行調整時間與確認紀錄。
- **Yahoo Finance 為非官方介面**：可能延遲、限流或改版；日本或韓國休市日會回傳上一個交易日的資料。若失敗，主流程會停在該步驟、當天不會推播。
- **RSS 保留時間有限**：收集 scenario 沒有跑到的日子，該日文章可能永遠無法補回。
- **LLM 模型下架風險**：模型被供應商下架時，Dify 會回報 404 `not_found_error`，請改選目前可用的模型。
- **免責聲明**：日報內容由 LLM 根據公開新聞與行情資料整理，可能有誤，僅供參考。
