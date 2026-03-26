# AI CLI Agent 輕量化全面網路資訊獲取層需求規格

## 1. 文件目的

本文件定義一個專為 AI CLI Agent 設計的輕量化但全面的網路資訊獲取層，暫稱 `WebR`。

目標不是打造一套重量級通用爬蟲平台，而是建立一個對 AI Agent 最友善、低維護、低操作負擔、可持續擴充的網路資訊能力層，讓 Agent 能以最少的工具、最少的決策成本，完成查找、讀取、抽取、追蹤與引用網路資訊的任務。

## 2. 產品目標

`WebR` 的核心目標如下：

1. 讓 AI Agent 用極少數工具即可完成大部分網路資訊取得任務。
2. 自動判斷應使用搜尋、HTTP 抓取、內容萃取、瀏覽器渲染或結構化抽取。
3. 回傳乾淨、可引用、可驗證、可追溯的結果，而不是雜亂的原始 HTML。
4. 支援最新資訊查找、指定來源過濾、有限度爬取與批次處理。
5. 盡量輕量化部署與維護，不引入過早複雜化。

## 3. 設計原則

### 3.1 單一入口

對 Agent 而言，最理想的情況不是工具越多越好，而是工具越少越好。

`WebR` 應該盡可能讓 Agent 只需要少數幾個穩定能力，例如：

- `search_web`
- `read_url`
- `extract_from_url`
- `crawl_site`

更理想時，可進一步收斂為單一高階入口，由系統自動選擇底層策略。

### 3.2 自動路由

系統需自動判斷：

- 輸入是關鍵字還是 URL
- 該先搜尋還是直接抓取
- 該走靜態 HTTP 解析還是瀏覽器渲染
- 該回摘要、全文還是結構化資料
- 是否需要追蹤同站連結

### 3.3 內容優先於頁面

對 Agent 來說，真正有價值的是正文、日期、作者、重點欄位與來源證據，不是整份 HTML。

系統應優先提供：

- 乾淨正文
- 標題
- 日期
- 作者
- 摘要
- 重要連結
- 原始來源
- 抽取信心

### 3.4 引用與可追溯

每一筆資訊都應盡量可追溯，至少包含：

- 原始 URL
- 最終跳轉後 URL
- 網域
- 抓取時間
- 發布時間
- 可回溯的證據片段

### 3.5 必要時才升級

預設先走低成本的抓取方式，只有在必要時才啟用瀏覽器渲染。

優先順序如下：

1. 官方 API
2. JSON / RSS / Sitemap
3. HTTP 抓取 + 內容解析
4. 瀏覽器渲染

## 4. 主要使用場景

`WebR` 應能覆蓋以下常見任務：

1. 查找最新官方資訊並附上來源。
2. 讀取單一網址並提取乾淨正文。
3. 從頁面中抽取固定欄位，例如價格、日期、作者、標題、規格。
4. 在指定網域內做有限度 crawl。
5. 比較多個來源內容並彙整重點。
6. 監控指定頁面變化並回傳 diff。
7. 批次處理多個 URL 或多組查詢。

## 5. 核心能力需求

### 5.1 搜尋能力

需要提供可控搜尋能力，用於取得候選來源，而不是直接將搜尋結果視為答案。

最低需求：

- 支援關鍵字搜尋
- 支援網域過濾
- 支援時間範圍或新鮮度限制
- 支援官方來源優先
- 支援回傳標題、摘要、URL、排名資訊

### 5.2 網頁讀取能力

需能讀取單頁內容並自動清洗。

最低需求：

- 支援 HTML 頁面
- 支援自動正文抽取
- 支援 metadata 萃取
- 支援轉成純文字或 Markdown
- 支援去除導覽、廣告、頁尾等雜訊

### 5.3 動態頁面支援

當 HTTP 抓取無法取得內容時，應支援瀏覽器模式作為 fallback。

最低需求：

- 支援 JavaScript 渲染
- 支援等待內容載入
- 支援基本互動，例如點擊、滾動、等待 selector
- 支援抓取渲染後 DOM

### 5.4 結構化抽取能力

Agent 應能直接要求系統依 schema 抽取資料，而不是只回全文。

最低需求：

- 可輸入欄位 schema
- 支援單頁抽取
- 支援多頁統一 schema 抽取
- 支援回傳欄位信心與缺漏欄位

### 5.5 有限度爬取能力

不做重型網站鏡像，但需支援中小型研究型 crawl。

最低需求：

- 支援同網域限制
- 支援深度限制
- 支援頁數限制
- 支援 URL pattern 過濾
- 支援去重與已訪問記錄

### 5.6 批次處理能力

系統應可處理多筆任務，避免 Agent 對每筆資料逐次操作。

最低需求：

- 批次搜尋
- 批次 URL 讀取
- 批次 schema 抽取
- 批次結果統一 JSON 輸出

### 5.7 快取與變更追蹤

為降低成本並提升重複使用價值，應內建快取與更新比較。

最低需求：

- URL 快取
- 內容雜湊
- 重複內容去重
- 更新差異比對
- 可指定是否強制重抓

## 6. AI Agent 最需要的功能排序

若從 AI CLI Agent 真正使用價值排序，優先級如下：

1. 單一入口與低決策負擔
2. 自動路由與 fallback
3. 乾淨正文讀取
4. 結構化抽取
5. Citation 與來源可信度
6. 新鮮度控制
7. 有限度 crawl
8. 快取與 diff
9. 批次任務
10. 監控與排程

## 7. 建議工具介面

### 7.1 基礎介面

第一版建議先暴露以下四個工具：

```text
search_web(query, filters)
read_url(url, mode)
extract_from_url(url_or_urls, schema)
crawl_site(start_url, rules)
```

### 7.2 理想高階介面

若要進一步降低 Agent 的決策成本，可考慮收斂為一個高階入口：

```text
web_get(goal, input, options)
```

其中：

- `goal`：任務目的，例如「找最新官方價格頁」
- `input`：查詢字串或 URL
- `options`：schema、freshness、domain、official_only、mode 等

系統內部自動完成：

- 搜尋
- 選源
- 抓取
- 內容解析
- 結構化抽取
- citation 生成

## 8. 輸出資料格式

所有工具盡量統一回傳結構，便於 Agent 組裝與推理。

建議格式如下：

```json
{
  "url": "https://example.com/page",
  "final_url": "https://example.com/page",
  "domain": "example.com",
  "title": "Example Title",
  "published_at": "2026-03-26T00:00:00Z",
  "fetched_at": "2026-03-26T12:00:00Z",
  "source_type": "html",
  "author": "Author Name",
  "summary": "Short summary",
  "content": "Cleaned main text",
  "links": [],
  "metadata": {},
  "evidence_snippets": [],
  "confidence": 0.92,
  "cache_hit": false
}
```

若為結構化抽取，則增加：

```json
{
  "schema_result": {
    "plan_name": "Pro",
    "price": "$20",
    "billing_period": "monthly"
  },
  "field_confidence": {
    "plan_name": 0.95,
    "price": 0.88,
    "billing_period": 0.83
  }
}
```

## 9. 內容模式設計

為降低 token 成本，建議每個工具支援三種輸出模式：

1. `brief`
只回 metadata、摘要、關鍵證據。

2. `structured`
回固定 JSON 結構與抽取欄位。

3. `full`
回完整清洗後內容。

## 10. 系統自動決策流程

建議內部路由邏輯如下：

1. 判斷輸入是 URL 還是 query。
2. 若是 query，先搜尋候選來源。
3. 依官方性、時間、網域可信度做初步排序。
4. 嘗試 HTTP 抓取。
5. 若內容不足或頁面依賴 JS，啟用瀏覽器 fallback。
6. 依任務需求回傳摘要、全文或 schema 抽取。
7. 附上 citation、抓取時間與信心分數。
8. 快取結果並記錄內容雜湊。

## 11. 推薦技術選型

若以輕量化優先，建議技術堆疊如下：

- API 層：`FastAPI`
- HTTP 抓取：`httpx`
- HTML 正文抽取：`trafilatura`
- HTML 備援解析：`BeautifulSoup` / `lxml`
- 瀏覽器 fallback：`Playwright`
- PDF 解析：`PyMuPDF`
- RSS / Sitemap：`feedparser` 與 XML parser
- 資料模型：`Pydantic`
- 快取與任務記錄：`SQLite`
- 排程：`APScheduler` 或系統 `cron`

## 12. MVP 範圍

第一版不要追求全能，先完成最有價值的最小集合。

### MVP 必須包含

1. 搜尋能力，支援網域與時間限制。
2. 單頁讀取，支援正文清洗。
3. Playwright fallback。
4. schema 抽取。
5. 統一 JSON 輸出。
6. 快取與去重。
7. citation metadata。

### MVP 可暫緩

1. 分散式 crawler
2. 大規模代理池
3. 複雜互動腳本錄製
4. 多租戶權限管理
5. 向量資料庫整合
6. 高階 workflow 編排

## 13. 非目標

以下項目不應作為第一階段目標：

1. 取代所有專業爬蟲平台
2. 對抗高強度反爬系統
3. 執行違反網站條款的資料抓取
4. 建立超大規模分散式抓取基礎設施
5. 一開始就支援所有網站與所有資料格式

## 14. 成功標準

若 `WebR` 成功，應達成以下效果：

1. Agent 大多數情況不需要手動切換多個工具。
2. Agent 不需要頻繁自行判斷是否該用瀏覽器。
3. 回傳內容比傳統 HTML 抓取更乾淨、更可引用。
4. 針對最新資訊任務，能明確給出時間與來源。
5. 針對結構化資料任務，能直接輸出 schema 結果。
6. 在小到中型研究任務上，能以很低操作成本完成資訊蒐集。

## 15. 建議開發順序

建議採三階段推進：

### Phase 1

- `search_web`
- `read_url`
- `extract_from_url`
- SQLite cache
- citation metadata

### Phase 2

- Playwright fallback
- `crawl_site`
- PDF / RSS / Sitemap 支援
- diff 與監控

### Phase 3

- 批次任務優化
- 更精準的來源評分
- 高階單入口 `web_get`
- 任務模板與更完整的策略路由

## 16. 結論

這套系統的核心不是做出更多抓網頁工具，而是建立一個對 AI CLI Agent 極度友善的資訊層。

它應該具備以下特性：

- 工具少
- 自動化高
- 路由聰明
- 輸出乾淨
- 可引用
- 可追溯
- 易於擴充

若以實作優先，應先完成能讓 Agent 穩定執行 `search -> read -> extract -> cite` 的最小閉環，再逐步補上瀏覽器 fallback、crawl、diff 與監控能力。
