## 🚀 台股盤前資訊自動化推播系統 (ETL + LLM Workflow + Line Bot)

本專案是一個兼具**高即時性、輕量化**的財經盤前策略自動化系統。每天早上 **08:15** 主動觸發，透過 **Make** 進行多源異質數據採集與資料清洗（ETL），並結合 **Dify Workflow (GPT-5)** 進行跨日金融資訊推理與文本提煉，最終在 **08:30** 前將 1000~2000 字的 Markdown 盤前策略日報主動推播至用戶的 **LINE**。

---

### 🛠️ 系統架構圖 (Architecture)

```text
【來源 A：美股總經】 ──► RSS 擷取 ─► 時間與標題 Filter ─► Jina AI Reader ──► Text Aggregator ───┐
                                                                                             │
【來源 B：台股市況】 ──► RSS 擷取 ──► pubDate Filter ──► Jina AI Reader ──► Text Aggregator ───┼─► 【Compose a string】 ──► 【HTTP POST：Dify】 ──► 【LINE Messaging API】
                                                                                             │      (變數 all_news)        (GPT-5 結構化推理)         (每日 08:30 推播)
【來源 C：日韓開盤】 ───────────────────────────────────────────────────► Jina AI Reader ──────┘
```

---

### 📊 資料來源與工作流處理 (Data Sources & Pipeline)

#### 1. 🔴 來源 A：美股市況與總體經濟事件
透過財經媒體 RSS 採集美股收盤後的核心大盤分析（如 Fed 官員談話、通膨數據、企業財報）。利用標題篩選器過濾垃圾資訊後，經由 Jina AI 將多篇文章整合為標準 Markdown 字串。

#### 2. 🔵 來源 B：台股昨日盤後與籌碼
透過 RSS 網頁蒐集台股前一天新聞，並過濾出《台股盤後》與《三大法人買賣超》相關資訊，並跨日蒐集當天早晨《台股盤前》報導，確保掌握最新資訊。

#### 3. 🟢 來源 C：日韓股開盤市況
透過 Jina AI 讀取即時亞洲股市漲跌幅表格，並在 Target Selector（目標選擇器）鎖定日韓股表格內容，整合為 Markdown 表格結構。

#### 4. 🧠 Dify 推理語言模型
將上述三路資料聚合傳入 Dify。透過 Prompt Engineering 限制 GPT-5 的推理邊界。

#### 5. 🤖 LINE Bot 訊息推播
將 Dify 回傳的結構化 Markdown 文本，透過 Make 的 LINE Message API 模組，於每日早上 08:30 自動推播至使用者手機，確保在台股開盤 30 分鐘前提供高可讀性的精華選集。

---

### 📂 專案檔案結構說明 (Files)
```text
stock-morning-report-line-bot
├── README.md
└── workflows/                             <-- 工作流資料夾
    ├── make_etl_workflow.bluepoint.json   <-- Make 自動化資訊蒐集與 LINE 觸發工作流程
    └── dify_llm_agent.ifpkg               <-- Dify 語言模型工作流配置檔案
```

---

### 🚀 部署使用指南 (How to Use)

#### 步驟 1：Dify LLM 設定
1. 登入 Dify 後台，點擊「建立 App -> 匯入 DSL 檔案」。
2. 上傳本專案中的 workflows/dify_llm_agent.ifpkg。
3. 發布工作流，並在「存取 API」選單中生成 app- 開頭的 API 金鑰（API Secret Key）。

#### 步驟 2：Make 自動化管道部署
1. 登入 Make 後台，建立新 Scenarios 並點擊右上角「... -> Import Blueprint」。
2. 上傳本專案中的 workflows/make-etl-workflow.json。

#### 步驟 3：節點連線與認證設定
1. 資料源網址對齊：請自行準備您習慣閱讀的財經媒體、證交所資訊之 RSS 網址或網頁 URL，並填入 Make 的 RSS 與 Jina AI 模組中。
2. HTTP 模組連線：將 Make 的 HTTP 模組 URL 改為您個人的 Dify 工作流 API 路徑，並在 Headers 寫入 Authorization: Bearer app-您的Dify金鑰。
3. LINE 觸發設定：開通您的 LINE 官方帳號 Messaging API，在 Make 內綁定 Channel Access Token，並在 To 欄位填入您本人的 LINE User ID 或群組 Group ID。

#### 步驟 4：啟用 Make 自動排程
將 Make 右上角的 Inactive 開關切換為 Active，並自行設定每天早上自動執行時間。
