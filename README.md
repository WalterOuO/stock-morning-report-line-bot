# 🚀 全球宏觀經濟與台股盤前事件自動化集散地系統 (ETL + LLM Workflow)

本專案是一個兼具**高即時性、輕量化、低成本**的財經盤前策略自動化系統。每天早上 **08:15** 主動觸發，透過 **Make** 進行多源異質數據採集與資料清洗（ETL），並結合 **Dify Workflow (GPT-4o)** 進行跨日期籌碼推理與文本提煉，最終在 **08:30** 前將 1000~2000 字的精美 Markdown 盤前策略日報主動推播至用戶的 **LINE 私訊**。

---

## 🛠️ 系統架構圖 (Architecture)

```text
【來源 A：美股總經】 ──► RSS 擷取 (50則) ──► 時間與標題 Filter ──► Jina AI Reader ──► Text Aggregator ──┐
                                                                                                      │
【來源 B：台股大事】 ──► RSS 擷取 (200則) ─► 昨14:00跨日 pubDate Filter ─► Jina AI Reader ─► Text Aggregator ─┼─► 【Compose a string】 ──► 【HTTP POST：Dify】 ──► 【LINE Messaging API】
                                                                                                      │   (大變數 all_news)       (GPT-4o 結構化推理)    (08:30 盤前主動推播)
【來源 C：日韓開盤】 ───────────────────────────────────────────────────► Jina AI Reader ────────────────┘
                                                       (Target Selector: #index-asia)
```

---

## ✨ 核心技術亮點與痛點解決 (Key Features & Bug Fixes)

### 1. 🔴 來源 A：美股總經資料流 ── 多篇新聞動態聚合
* **痛點**：清晨收盤後會有多篇核心大盤分析（如：美股盤後、Fed 官員談話、通膨數據），個別爬取極度消耗 Token 且容易夾雜機器人「速報」垃圾訊息。
* **解法**：利用 Make 的 RSS 模組搭配多重條件篩選器（Filter），排除包含「速報」之項目，僅放行包含「大盤、焦點、分析」之核心文章。經由 Jina AI 轉為 Markdown 後，透過 **Text Aggregator（文字聚合器）** 將多篇內文重組為標準 Markdown 大字串。

### 2. 🔵 來源 B：台股昨日盤後與籌碼 ── 跨日時間防火牆
* **痛點**：專業操盤手必看的《台股盤後》與《三大法人買賣超》於前一天下午 14:00 至 20:00 發布，在當天早上 08:15 執行時，極易因媒體發文量不固定而被擠出 RSS 首頁（N=50外）。
* **解法**：將 RSS 回傳項目擴大至最大防禦值（N=200），並在 Filter 中創新引入 **`pubDate` 時間反推函數**。強制系統動態推算並「只放行晚於前一天下午 14:00:00 的文章」，100% 確保跨日籌碼數據不漏接。

### 3. 🟢 來源 C：日韓亞股開盤 ── CSS 節點精準清洗
* **痛點**：日韓股於 08:00 開盤，08:15 採集時需即時摘錄最新漲跌幅表格，若爬取完整網頁（包含側欄、廣告、非亞股市場）會導致 LLM 讀取雜訊過多且容易發生數據錯漏。
* **解法**：不依賴不穩定的 RSS，直接請 Jina AI 讀取固定數據頁面，並在 **Target Selector（目標選擇器）中死死寫入網頁代碼節點 `#index-asia`**。以手術刀式精準切除亞洲股市以外的所有幾萬字雜訊，只回傳乾淨的亞股大盤表格結構。

### 4. 🧠 Dify 核心推理大腦 ── 嚴格邊界提示詞 (Prompt Engineering)
* **防幻覺與鐵律限制**：由於各媒體採集數據不固定（如部分日子不提匯率），Prompt 嚴格禁止 LLM 無中生有（防止幻覺）。並下達「美股段落不允許做台股預測」之**推理邊界限制**，將所有綜合交叉推演與操作思維，嚴格集中於第四大段統一輸出。
* **系統金鐘罩**：LLM 思考節點全面開啟「失敗時重試策略」（3次 / 間隔2000ms），防禦早盤數據高峰期的 API 網路塞車；錯誤處理機制設定為中斷報錯，防止 LINE 推播空白垃圾訊息。

---

## 📂 專案檔案結構說明 (Files)

* `/Make_Blueprint_Workflow.json` : Make 自動化採集、清洗與 LINE 觸達的完整工作流 Blueprint 導出檔案。
* `/Dify_App_DSL_Workflow.yml` : Dify 核心思考大腦（包含 System/User Prompt、GPT-4o 模型配置、重試防禦策略）的 DSL 導出檔案。

---

## 🚀 如何復原與使用 (How to Use)

1. **Dify 端復原**：在 Dify 後台點擊「建立 App -> 匯入 DSL 檔案」，上傳本倉庫的 `.yml` 檔案。發布工作流並在「存取 API」生成 `app-` 金鑰。
2. **Make 端復原**：在 Make 後台點擊「... -> Import Blueprint」，上傳本倉庫的 `.json` 檔案。
3. **連線設定**：
   * 將 Make 的台美股 RSS 網址與日韓股 URL 對齊。
   * 將 Make 的 HTTP 模組 URL 改為你的 Dify 工作流路徑，並在 Headers 寫入 `Authorization: Bearer app-你的金鑰`。
   * 開通你的 LINE 官方帳號 Messaging API，在 Make 內綁定 `Channel Access Token`，並在 `To` 欄位填入你本人的 `Your user ID`。
4. **通電運行**：將 Make 左下角的 Scheduling 開關切換為 **Active (ON)**，設定每天早上 **08:15** 執行。
