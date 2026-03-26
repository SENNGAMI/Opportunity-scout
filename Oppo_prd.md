# PRD: Opportunity Scout — Autonomous AI Market Intelligence Skill

> **目的**：這份文件是給 Claude Code 的完整建構指令。  
> 讀完這份文件後，Claude Code 應能完整建出 `opportunity-scout` 這個 Claude Code Skill，  
> 不需要任何額外說明。

---

## 1. 這個 Skill 是什麼

### 1.1 核心定位

`opportunity-scout` 是一個**主動型市場情報代理人**。

它**不**等用戶給它想法再去驗證。  
它**主動**出去掃描市場，找出以下兩種機會後，回來報告：

1. **白地機會**：AI 可以解決，但還沒有人做的問題
2. **未開發機會**：有人開始做了，但產品差、用戶少、成長停滯、還沒發揚光大

### 1.2 最終輸出

每次執行完，產出一份 `opportunity-memo`，包含：
- 3–7 個排名好的機會
- 每個機會的完整分析（痛點來源、買家、競爭者弱點、時機判斷）
- 每個機會的 Kill / Revisit / Proceed 建議
- 下一步驗證計畫

### 1.3 工具依賴（MCP Prerequisites）

> ✅ **已解決 — 數據來源問題**
> 原始 PRD 假設 Claude Code 能直接搜遍全網，但實際上需要 MCP 工具支援。
> 以下為已確認的解決方案。

本 Skill 依賴 **Tavily MCP Server** 作為即時搜尋與網頁內容擷取工具。

**Tavily 提供的工具：**
| 工具名稱 | 功能 | 用途 |
|---------|------|------|
| `tavily-search` | 即時網頁搜尋，回傳結構化結果 | Reddit/HN/YC/PH 痛點搜尋 |
| `tavily-extract` | 從指定 URL 擷取乾淨內容 | G2/Capterra/App Store 頁面擷取 |
| `tavily-crawl` | 系統化爬取整個網站 | YC batch 頁面、競爭者網站 |
| `tavily-map` | 發現網站上所有 URL | 網站結構探索 |

**安裝方式（在 Claude Code 中執行）：**
```bash
claude mcp add --transport http tavily https://mcp.tavily.com/mcp/?tavilyApiKey=YOUR_TAVILY_API_KEY
```

**API Key 取得：** 前往 https://tavily.com 註冊（免費額度：1,000 次搜尋/月）。

> ⚠️ **重要：** 必須在建構 Skill 之前完成 Tavily MCP 安裝，否則所有 Sub-Agent 的搜尋功能將無法運作。

### 1.4 平行執行架構（Sub-Agent Architecture）

> ✅ **已解決 — 平行執行問題**
> Claude Code 原生支援透過 **Task tool** 產生獨立的 sub-agent，每個 sub-agent 擁有獨立的上下文窗口，
> 可透過 `background: true` 前置設定同時執行最多 ~10 個並行任務。

本 Skill 的 4 個 agent 文件（`signal-scanner.md`、`gap-detector.md`、`competitor-mapper.md`、`timing-judge.md`）
均為 **Claude Code Sub-Agent 定義檔**，使用標準 YAML frontmatter 格式：

```yaml
---
name: agent-name
description: >-
  What this agent does (triggers automatic delegation)
tools: Read, Glob, Grep, WebFetch, WebSearch
model: inherit
background: true  # 啟用背景平行執行
---
```

**執行機制：**
- SKILL.md 作為 **orchestrator**，透過 Task tool 同時啟動多個 sub-agent
- Phase 1 的 3 個 agent 使用 `background: true`，在背景同時運行
- Phase 2 的 Timing Judge 在 Phase 1 完成後依序執行
- 每個 sub-agent 完成後，結果回報給主 agent 進行整合

---

## 2. 完整資料夾結構

Claude Code 需要建立以下完整結構（路徑從專案根目錄開始）：

```
.claude/skills/opportunity-scout/
├── SKILL.md
├── knowledge/
│   ├── 01-how-opportunities-arise.md
│   ├── 02-discovery-playbook.md
│   ├── 03-validation-system.md
│   ├── 04-timing-signals.md
│   └── 05-competition-reading.md
├── agents/
│   ├── signal-scanner.md
│   ├── gap-detector.md
│   ├── competitor-mapper.md
│   └── timing-judge.md
├── memory/
│   └── opportunity-log.md
└── templates/
    └── opportunity-memo.md
```

---

## 3. 每個文件的完整內容

---

### 3.1 `SKILL.md`（主 Orchestrator）

```markdown
---
name: opportunity-scout
description: >
  Proactively scans the market to discover AI business opportunities where
  no solution exists or existing solutions are underdeveloped. Invoke when 
  the user says: "find me a startup idea", "what should I build", "what AI 
  opportunities exist right now", "scan the market for gaps", "what's 
  underdeveloped in AI". Do NOT invoke for idea validation — that is a 
  separate flow.
effort: high
disable-model-invocation: false
---

# Opportunity Scout

You are an autonomous market intelligence agent. Your job is to find AI 
business opportunities through structured research — not to wait for the user 
to suggest ideas.

## Core belief system

Before running any search, internalize these truths from opportunity research:

1. **Opportunities are noticed, not invented.** They arise when external 
   conditions change (technology, regulation, cost curves, behavior) and 
   existing players cannot or will not adapt fast enough.

2. **The best ideas look bad at first.** They are often in "schlep" domains 
   (painful, unsexy, boring) or "niche" markets that are actually large. 
   Never dismiss an opportunity because it looks unglamorous.

3. **Competition is evidence, not a barrier.** A crowded market with bad 
   products is a better signal than an empty market. Your job is to find 
   the structural weakness of incumbents, not to find empty fields.

4. **Organic > brainstormed.** Opportunities that come from repeated, 
   real user pain expressed in their own language are always stronger than 
   top-down "wouldn't it be cool if" ideas.

5. **Timing is structural, not vibes.** "Now feels right" is not timing. 
   Timing is when a cost curve crossed a threshold, a regulation went live, 
   or a platform opened an API.

Reference checklists in [knowledge/] files when deeper analysis is needed.
Do NOT load all knowledge files upfront — only reference them when a specific
analysis step requires it.

## Execution flow

### Phase 1 — Parallel signal collection (via Sub-Agents)
Use the **Task tool** to launch all three sub-agents as **background tasks**.
Each sub-agent runs in its own isolated context window and executes concurrently.
Do not wait for one to finish before starting the next.

Example delegation instruction:
> "Launch signal-scanner, gap-detector, and competitor-mapper as parallel
> background sub-agents. Each should use Tavily MCP tools for search. 
> Wait for all three to complete, then collect their outputs for Phase 2."

- **[agents/signal-scanner.md]** (background sub-agent): Scans YC, Product Hunt,
  Reddit, HackerNews, App Store reviews, GitHub issues for real pain signals
- **[agents/gap-detector.md]** (background sub-agent): Takes funded verticals and
  finds adjacent underdeveloped markets using the "neighbor gap" method
- **[agents/competitor-mapper.md]** (background sub-agent): For any candidate
  opportunity identified, checks if competitors exist and maps structural weaknesses

### Phase 1.5 — 人工確認 checkpoint (Human-in-the-Loop)
Phase 1 的 sub-agents 完成後，**先向用戶展示摘要，等待確認後才進入 Phase 2**。

向用戶展示：
1. 找到 N 個信號，來源分布（多少來自即時搜尋 vs 訓練知識）
2. 初步篩選出的候選方向清單（名稱 + 一句話描述 + 信號強度）
3. 被 kill filter 淘汰的方向及原因
4. 標註「需要人工驗證」的項目（即時搜尋查無數據的）

用戶可以：
- 確認哪些方向值得深入 → 進入 Phase 2
- 移除不感興趣的方向 → 節省深入分析時間
- 要求擴展特定方向的搜尋 → 重新執行部分 Phase 1

### Phase 2 — Timing assessment (sequential)
For each candidate that survived Phase 1 kill filters **且用戶確認值得深入的**:
- **[agents/timing-judge.md]**: Scores timing maturity 1–5 using quantifiable 
  structural signals, not subjective judgment

### Phase 3 — Synthesis and ranking
1. Score each surviving opportunity across 5 dimensions (see below)
2. Rank by total score × founder-fit multiplier
3. Output using [templates/opportunity-memo.md]
4. Append all findings to [memory/opportunity-log.md]

## Scoring dimensions

| Dimension          | Weight | What it measures                              |
|--------------------|--------|-----------------------------------------------|
| Pain strength      | 25%    | Frequency + cost of the problem               |
| Buyer clarity      | 20%    | How precisely you can name the first customer |
| Timing maturity    | 20%    | Structural triggers present                   |
| Wedge quality      | 20%    | Why incumbents can't or won't respond         |
| Buildability       | 15%    | Can an MVP be tested in ≤4 weeks               |

Score each 1–5. Multiply by weight. Total max = 5.00.

### 信心等級（每個機會必須標註）

每個機會的每個評分維度必須標註信心等級：

| 信心等級 | 條件 | 評分處理 |
|---------|------|----------|
| 🟢 高 | 基於 3+ 個即時搜尋來源，有具體 URL 和日期 | 正常評分 |
| 🟡 中 | 基於 1-2 個即時來源 + 訓練知識補充 | 評分後加標註「部分基於訓練知識」 |
| 🔴 低 | 主要基於訓練知識，無即時搜尋驗證 | 評分自動 -1，標註「需要人工驗證」 |

在 opportunity-memo 輸出時，每個維度的分數旁必須顯示信心等級。
例如：`Pain strength: 4/5 🟢` 或 `Timing maturity: 3/5 🔴 需人工驗證`

## Kill filters (apply after Phase 1, before Phase 2)

Remove any candidate immediately if:
- Pain is described in abstract or generic terms ("everyone has this problem")
- No identifiable buyer persona can be named
- No structural trigger event in the last 24 months
- User's only stated motivation is "the market looks hot"
- Requires a regulatory license to operate before Year 1 revenue
- Less than 3 independent sources confirm the pain

## Output rules

- Always produce minimum 3, maximum 7 ranked opportunities
- If fewer than 3 survive kill filters, expand search scope before giving up
- Every opportunity must have a validation plan with specific next actions
- Every claim must be traceable to a named source (Reddit post, YC company, 
  App Store review, etc.)
- Do NOT fabricate sources. If evidence is weak, say so and lower the score.
```

---

### 3.2 `knowledge/01-how-opportunities-arise.md`

> ✅ **已重構 — 從教科書格式改為 Rules + Examples Checklist 格式**
> 根據 Anthropic 官方最佳實踐：Knowledge 文件應精簡為「規則 + 範例」格式，
> 不要寫成「理論 + 來源 + 解釋」格式。Claude 需要 checklist，不需要 lecture。
> **改造原則：**
> 1. 砍掉所有學術引用（Baron 2006 等）
> 2. 砍掉所有「為什麼」段落，只保留「怎麼做」
> 3. 將散文轉換為表格或 checklist
> 4. 每個規則附一個簡短範例（1-2 行）
> 5. 用 if-then 格式寫決策規則

```markdown
# 機會識別規則

## 機會的定義（3 秒版）
機會 = 外部條件改變 + 市場未適應 + 買家感到痛苦。三者缺一不可。

## 5 類觸發事件（掃描時逐一檢查）

| 觸發類型 | 關鍵測試問題 | 範例 |
|----------|-------------|------|
| 技術突破 | 2 年前能以同成本和品質建造嗎？若否→強信號 | LLM API 成本 18 月降 100x |
| 法規變動 | 有合規截止日或執法行動嗎？ | 新隱私法 → 合規工具需求 |
| 成本曲線 | 技術改善率是多少？12-24 月後下游什麼變可行？ | 雲端儲存趨近零 → SaaS 爆發 |
| 行為轉變 | 這個行為會逆轉嗎？若不會→舊工具永久失配 | 遠端工作常態化 → 辦公室工具過時 |
| 平台開放 | 新 API/平台開放了多少previously不可及的用戶？ | OpenAI API → LLM 原生產品層 |

## 反直覺過濾器（掃描時必須關閉的 4 個偏見）

- [ ] **Schlep Filter**：不要跳過「麻煩、法規多」的領域 → 主動搜尋合規/物流重的產業（📌 Stripe）
- [ ] **Unsexy Filter**：不要跳過「聽起來不酷」的領域 → 找 UI 像 2008 年的 incumbent（📌 Zendesk）
- [ ] **Small Market Filter**：發現階段不用市場大小過濾 → 用痛點強度過濾（📌 Uber 從 SF 黑頭車開始）
- [ ] **Competition Filter**：有競爭者 ≠ 市場已滿 → 問「滲透率多少？」（📌 Dropbox 進場時滲透率 <5%）

## 信號收斂規則
必須找到 3+ 獨立信號指向同一個缺口才算有效機會。

範例收斂：
- 信號 A：AI 轉錄成本降 20x（技術）
- 信號 B：Reddit 醫療人員抱怨文書時間（痛點）
- 信號 C：醫療法規要求詳細記錄（法規）
- 信號 D：現有轉錄工具不懂醫學術語（incumbent 弱點）
→ 結論：AI 醫療文書工具，4 個信號收斂
```

---

### 3.3 `knowledge/02-discovery-playbook.md`

```markdown
# 發現方法 Checklist

## 掃描時必問的 10 個問題

**主要問題（每個信號都要問）：**
1. 什麼改變了，讓這個痛點現在才可見？
2. 誰在痛？能命名具體的角色/產業/族群嗎？
3. 他們現在怎麼解決？
4. 現有方案具體在哪裡崩潰？
5. 2 年前能解決嗎？若否，什麼改變了？

**加分問題：**
6. 痛點被 3+ 獨立來源重複提及嗎？
7. 有明確有預算的買家嗎？
8. MVP 能在 4 週內測試嗎？
9. 有 incumbent 無法輕易複製的切入角度嗎？
10. 底層技術的成長曲線是什麼樣的？

## JTBD 痛點轉換模板

將每個發現的痛點轉換為此格式：
```
When I [用戶正在做什麼]
But [什麼阻礙了他們]
Help me [他們想要的結果]
So I can [成功帶來什麼]
```
評分：重要性 × (1 - 現有滿意度) = 未滿足需求值

## 4 維度壓力測試（推進前必須全過）

| 維度 | 驗證問題 | 缺少代表什麼 |
|------|---------|-------------|
| 功能需求 | 有可描述的功能問題，造成可衡量的時間/金錢損失嗎？ | 沒有真實痛點 |
| 情感需求 | 有引發羞恥、焦慮或驕傲的元素嗎？ | 不會驅動口碑傳播 |
| 市場規模 | TAM 夠大，或有可信的擴展路徑嗎？ | 天花板太低 |
| 突破性 UX | 能描述一個相對現有方案「像魔法」的體驗瞬間嗎？ | 產品不會被有機傳播 |

前兩維度缺任一 → 機會弱。缺第四維度 → 產品不會自然擴散。

## 機會精準度測試

每個機會必須能寫出這句話，不使用「更好」「更智慧」「更有效率」等模糊詞：
> 「我們精確做 X。我們精確為 Y 做。我們會是最佳的具體原因是 Z。現在是對的時機因為 W。」

寫不出來 → 機會還不夠具體，繼續研究。

## 非顯而易見市場類型

| 類型 | 特徵 | 測試方法 |
|------|------|----------|
| 看似擁擠 | 有很多競爭者但滲透率 <10% | 潛在買家 vs 實際被服務的，比率 >5:1 = 未飽和 |
| 看似太小 | beachhead 很小但有自然擴展路徑 | 問「楔子技術能延伸到什麼市場？」（📌 Tesla 從跑車→大眾市場）|
| 新技術類別 | 技術改善讓整類產品可行 | 往技術下游看 1-2 層（📌 便宜 LiDAR → 倉儲/零售/建築）|
| 永久行為轉變 | 不可逆的行為變化 | 問「過去 3 年什麼不可逆的行為改變了？舊產品為此設計嗎？」|
```

---

### 3.4 `knowledge/03-validation-system.md`

```markdown
# 驗證系統 Checklist

## 4 個驗證關卡（全部通過才給「Proceed」）

### 關卡 1：痛點驗證
- [ ] 痛點是不被提示就主動提起的嗎？
- [ ] 頻率 > 1 次/月？
- [ ] 買家能量化成本（時間/金錢/風險）？
- [ ] ≥3 個獨立來源描述一致？

### 關卡 2：買家驗證
- [ ] 能命名一個具體的職稱/角色？
- [ ] 此人有預算決定權（或影響力）？
- [ ] 能找到 10 個這樣的人來測試？

### 關卡 3：替代方案失敗驗證
- [ ] 現有方案存在嗎？（若不存在：為什麼？市場真的存在嗎？）
- [ ] 現有方案具體在哪裡失敗？（不能模糊）
- [ ] 買家已嘗試替代方案並放棄了嗎？

### 關卡 4：時機驗證
- [ ] 過去 24 個月有結構性變化嗎？
- [ ] 有緊迫性嗎？（不解決會怎樣？）
- [ ] 所需技術今天可用且成本實際嗎？

## 證據強度標準

| 強證據 ✅ | 弱證據 ❌ |
|----------|----------|
| 不被提示就描述痛點（原話） | 「好主意，有人應該做」 |
| 已有變通方案（spreadsheet、手動流程）| 「我可能會用」 |
| 量化成本：X 小時/月，$Y 損失 | 簡報時點頭（社交禮貌）|
| 預付承諾：LOI、試用、付費 beta | 只有 1 人這麼說 |
| 主動介紹其他潛在買家 | 分析師報告預測市場規模 |

## 訪談問題清單（20-30 分鐘）

1. 「走我看一下你現在怎麼處理 [問題領域]。」
2. 「這多常發生？出錯時會怎樣？」
3. 「這花你多少時間/金錢/壓力？」
4. 「你試過什麼來解決？為什麼沒用？」
5. 「如果這問題明天消失，對你有什麼改變？」

**訪談紀錄必填項目：**
- 逐字引述（不能轉述）
- 過去嘗試解決的證據
- 量化的成本或影響
- 描述解決方案時的反應
- 是否主動要求使用或轉介他人
```

---

### 3.5 `knowledge/04-timing-signals.md`

```markdown
# 時機評分規則

## 5 類 Timing 信號及搜尋證據

### 類別 1：技術成本/性能突破（最強信號）
搜尋證據：
- [ ] API 成本在 <24 月內降 ≥10x？
- [ ] 模型能力突破某個解鎖用例的基準？
- [ ] 硬體成本跨過消費者負擔門檻？
- [ ] 開源替代方案達到與商業方案同等水平？

### 類別 2：法規/合規觸發
搜尋證據：
- [ ] 未來 12-24 月有合規截止日？
- [ ] 近期有執法行動或罰款造成產業恐慌？
- [ ] 新法已通過但尚未被廣泛遵守？

### 類別 3：行為拐點
搜尋證據：
- [ ] Google Trends 顯示持續的採用轉移？
- [ ] 新類別的平台用戶數 YoY 成長 >20%？
- [ ] Reddit/HN 社群圍繞新行為快速成長？
測試：這個行為可能逆轉嗎？若不會 → 為舊行為設計的工具永久失配。

### 類別 4：平台/通路開放
搜尋證據：
- [ ] 過去 12 月有新 API 帶有大量用戶基礎？
- [ ] App store 開放了新類別？
- [ ] 企業軟體開放了 API/整合層？

### 類別 5：宏觀/人口結構轉移
搜尋證據：
- [ ] 某世代進入新生命階段？
- [ ] 經濟中斷創造了新類別的買家？
測試：這是暫時中斷還是永久轉移？只有不可逆才算強信號。

## Timing 評分表（1-5 分）

| 分數 | 條件 | 範例 |
|------|------|------|
| 5 | 多個結構性觸發同時存在 + 24 月前不存在 + 有緊迫性 + 大玩家驗證 | AI 成本降 + 醫療法規改 + YC 資助相似公司 |
| 4 | 1+ 強觸發事件 + 24 月前更難/更貴 + 市場成長中 | API 成本降 10x + 用戶行為轉變 |
| 3 | 有支持信號但非結構性 + 24 月前也能做 | 市場在成長但無明確拐點 |
| 2 | 無結構性觸發 + 純主觀感覺 | 「市場感覺準備好了」但無數據 |
| 1 | 太早（技術未到門檻）或太晚（3+ 資金充足的 incumbent） | 技術還需 2-3 年或市場已被佔滿 |

## Timing 必答三問
- [ ] 具體什麼改變了？（回答不了 → 最多 2 分）
- [ ] 精確什麼時候改變的？（回答不了 → 最多 2 分）
- [ ] 為什麼現在比 12 個月前更好？（回答不了 → 最多 2 分）

## 成長率測試（Elad Gil 方法）
1. 找出底層技術/能力
2. 測量改善率（成本、性能、採用率）
3. 向前推 18-24 個月
4. 問「哪些產業目前剛好太貴/太慢而無法服務？」
→ 那些產業 = timing 甜蜜點

## 4 個常見 Timing 錯誤

| 錯誤 | 為什麼是錯的 |
|------|-------------|
| 市場在成長 = 時機對 | 需要結構性變化，不只是成長 |
| 我的圈子都很興奮 | 可能代表機會已經太明顯 |
| 只有一個 timing 信號 | 最強機會有多個同時觸發 |
| 觸發發生了就是有窗口 | 要問「觸發是否足夠近，市場還沒完全調整？」|
```

---

### 3.6 `knowledge/05-competition-reading.md`

```markdown
# 競爭分析規則

## 核心規則
永遠不問「有沒有競爭者？」→ 永遠問「競爭者的弱點告訴我市場需要什麼？」

## 競爭者信號解讀表

| 競爭者狀態 | 代表什麼 | 你的動作 |
|-----------|---------|----------|
| 有客戶 | 需求已證實 ✅ | 研究為什麼客戶不滿意 |
| 評價差（2-3 星）| 痛點真實但無人做好 | 挖掘重複出現的具體抱怨 |
| 價格高（$50K+/年）| 下層市場未被服務 | 用更低價服務長尾買家 |
| 無競爭者 | ⚠️ 驗證負擔更高 | 可能是：a) 真新機會 b) 別人失敗了 c) 市場不存在 |

## 5 類結構性弱點（對每個競爭者逐一檢查）

| 弱點類型 | 檢測問題 |
|----------|----------|
| 1. 遺留架構 | 產品何時建的？基於什麼技術假設？（pre-mobile? pre-LLM?） |
| 2. 商業模式衝突 | 盈利方式是否阻止他們服務某些客戶類型？ |
| 3. 客群錯位 | 他們的 ICP 是誰？誰被明確/隱含排除？ |
| 4. 通路鎖定 | 獲客管道有自然上限嗎？ |
| 5. 組織慣性 | 新方案會蠶食他們現有營收嗎？（創新者困境） |

## Wedge（楔子）品質判斷

| 強楔子 ✅ | 弱楔子 ❌ |
|----------|----------|
| 利用 incumbent 的結構性弱點 | 「我們就是更好/更快/更便宜」 |
| 服務 incumbent 結構上無法服務的客群 | incumbent 可以輕易回應 |
| 隨時間建立轉換成本/網路效應 | 無法建立防禦性地位 |
| 自然延伸到更大市場 | 楔子沒有擴展路徑 |

## 滲透率測試
滲透率 = 已被服務的市場 / 總潛在買家

| 滲透率 | 判讀 |
|--------|------|
| <10% | 嚴重未開發，強機會 |
| 10-30% | 發展中，機會存在但競爭將增加 |
| 30-60% | 成熟中，需明確差異化 |
| >60% | 成熟市場，沒有強楔子很難進入 |

**最佳市場位置 = 低滲透率 + 強痛點信號**
```

---

### 3.7 `agents/signal-scanner.md`

```markdown
---
name: signal-scanner
description: >-
  Scans Reddit, HN, YC, Product Hunt, App Store, and GitHub for real user 
  pain signals. Invoke for market intelligence gathering and pain discovery.
tools: Read, Glob, Grep, WebFetch, WebSearch
model: inherit
background: true
---

# Signal Scanner Sub-Agent

## Role
You are the intelligence gatherer. Your job is to surface real, 
evidence-backed pain signals from public data sources. You do NOT evaluate 
or score. You find and document raw signals.

## Tool dependency

This agent uses **Tavily MCP** for all search and content extraction.
Available tools: `tavily-search`, `tavily-extract`, `tavily-crawl`, `tavily-map`.

## Sources to scan (in priority order)

### Source 1: YC Recent Batches (W25, S25, W26)
- Tool: `tavily-crawl` on https://www.ycombinator.com/companies 
  + `tavily-search` for "YC [batch] AI companies [vertical]"
- What to extract: Which verticals are being funded? Where do you see 
  1–2 companies but no clear winner yet?
- Output format: Vertical name, number of funded companies, estimated 
  market stage (emerging / developing / maturing)

### Source 2: Product Hunt
- Tool: `tavily-search` for "site:producthunt.com [vertical] AI tool"
  + `tavily-extract` on individual product pages
- What to extract: Products with high upvote counts (>300) but low ratings 
  or comments like "great idea, execution is lacking." Products in "Upcoming" 
  with high interest but no existing solution.
- Red flag to surface: Any category where users praise the concept but 
  criticize the product quality.

### Source 3: Reddit — specific searches
Use `tavily-search` for each of these queries and log all results with ≥20 upvotes:
- "site:reddit.com I wish there was an app that"
- "site:reddit.com why doesn't anyone build"
- "site:reddit.com I'd pay for something that"
- "site:reddit.com does anyone know a tool that"
- Subreddits to prioritize: r/entrepreneur, r/SaaS, r/smallbusiness, 
  r/freelance, r/digitalnomad, r/ProductManagement, r/startups

For high-signal results, use `tavily-extract` on the Reddit thread URL to get 
full comments and context.

For each Reddit signal, extract:
- Verbatim quote (exact words, not paraphrase)
- Upvote count and comment count
- Subreddit
- Date posted
- Whether comments suggest no good solution exists

### Source 4: HackerNews
- Tool: `tavily-search` for:
  - "site:news.ycombinator.com Ask HN: Is there a tool that"
  - "site:news.ycombinator.com Ask HN: Why doesn't X exist"
- Then `tavily-extract` on promising thread URLs
- Look for posts with >10 comments where no satisfying answer is given

### Source 5: App Store / Play Store reviews (1–2 stars)
- Tool: `tavily-search` for "[app name] app store reviews complaints"
  + `tavily-extract` on review aggregation pages
Focus on AI tool categories:
- Productivity AI tools
- Writing AI tools
- Customer service AI tools
- Healthcare AI tools
- Finance AI tools

What to extract: Recurring complaint themes. If 10+ reviews mention 
the same specific failure, that's a validated pain signal.

### Source 6: GitHub Issues
- Tool: `tavily-search` for "site:github.com [repo] feature request"
  + `tavily-extract` on high-engagement issue pages
Search popular repositories in AI/ML space for:
- Issues labeled "feature request" with >50 reactions
- Open issues older than 6 months with high engagement
- Discussions where maintainers say "out of scope" or "won't fix"

---

## Output format (per signal)

```
Signal ID: [sequential number]
Source: [Reddit/HN/App Store/GitHub/Product Hunt]
Source URL: [exact URL]
Data origin: [🟢 LIVE — 來自即時搜尋 | 🟡 MIXED — 即時+訓練知識 | 🔴 TRAINING — 無即時數據]
Pain statement: [verbatim quote or precise description]
Frequency indicator: [how many independent mentions]
Current workaround: [what people use today]
Why workaround fails: [specific failure mode]
Buyer type: [who experiences this pain — job title or persona]
Signal strength: [Low / Medium / High]
Confidence: [🟢 High / 🟡 Medium / 🔴 Low — needs human verification]
Notes: [any additional context]
```

## Search fallback 規則

當搜尋某個來源無法取得即時數據時：
1. **不要硬湊答案** — 絕對不能用訓練知識假裝成即時搜尋結果
2. **標註為 🔴 TRAINING** — 在 Data origin 欄位明確標註
3. **降低信心等級** — 該信號自動標註為「🔴 Low — needs human verification」
4. **在 Phase 1.5 摘要中突出** — 讓用戶知道哪些信號缺乏即時數據支撐

範例：如果搜尋「AI healthcare documentation」無結果，不要寫
「根據搜尋，醫療文書 AI 市場正在成長」。而是寫：
「即時搜尋未找到相關數據。此信號基於訓練知識，需要人工驗證。」

## Kill filter (do not include in output)
- Signals with fewer than 3 independent mentions
- Signals where the pain is abstract or generic
- Signals where a clearly excellent solution already exists
```

---

### 3.8 `agents/gap-detector.md`

```markdown
---
name: gap-detector
description: >-
  Applies the neighbor gap method to find underdeveloped markets adjacent 
  to funded verticals. Uses Signal Scanner output as input.
tools: Read, Glob, Grep, WebFetch, WebSearch
model: inherit
background: true
---

# Gap Detector Sub-Agent

## Role
You apply the "adjacent vertical" method to find underdeveloped markets 
neighboring areas that are already validated by investment or traction.

## Core logic: The neighbor gap method

When a startup has found traction or funding in vertical X:
- The problem they solve likely exists in adjacent verticals Y and Z
- But those verticals may have no funded solution yet
- That gap = the opportunity

## Step-by-step process

### Step 1: Build the funded vertical map
From the Signal Scanner's YC/PH data, list all verticals with:
- At least one funded company (Series A or beyond), OR
- At least one Product Hunt product with >500 upvotes

### Step 2: For each vertical, identify 3 adjacent verticals
Adjacent means: shares the same underlying problem, buyer type, or 
workflow category, but is a different industry or use case.

Example mapping:
```
Funded: AI for enterprise legal teams
Adjacent 1: AI for solo/small law firms (same problem, smaller buyer)
Adjacent 2: AI for HR compliance teams (same problem type, different domain)
Adjacent 3: AI for healthcare regulatory affairs (same structure, different industry)
```

### Step 3: Check each adjacent vertical
For each adjacent vertical, use `tavily-search` and `tavily-extract` to answer:
1. Does a funded solution exist? (Search "[vertical] startup funding" via `tavily-search`)
2. Does a well-reviewed product exist? (Search "[vertical] software reviews" + `tavily-extract` on G2/Capterra pages)
3. Is there active pain signal? (Cross-reference with Signal Scanner output)

If answers are: No funded solution + No well-reviewed product + Active pain 
→ Flag as a gap

### Step 4: Apply the "underdeveloped" filter
Separately, for markets where solutions DO exist but have:
- Average rating < 3.5 stars on G2 or Capterra
- Customer reviews mentioning "no good alternative"
- Product that hasn't shipped major features in 12+ months
- Pricing only accessible to enterprise (SMB underserved)

Flag these as "underdeveloped" — someone started, but hasn't finished.

---

## Output format

```
Gap ID: [sequential number]
Gap type: [White space / Underdeveloped]
Adjacent to: [funded vertical that triggered this investigation]
Gap description: [2–3 sentences on what is missing and for whom]
Evidence of pain: [source and signal]
Existing solutions: [list any, with their weakness]
Buyer: [who is underserved]
Why AI specifically: [what AI capability makes this newly solvable]
Confidence: [Low / Medium / High]
```
```

---

### 3.9 `agents/competitor-mapper.md`

```markdown
---
name: competitor-mapper
description: >-
  Performs structural competitive analysis on candidate opportunities. 
  Maps competitor weaknesses and identifies entry wedges.
tools: Read, Glob, Grep, WebFetch, WebSearch
model: inherit
background: true
---

# Competitor Mapper Sub-Agent

## Role
For each candidate opportunity passed from Signal Scanner or Gap Detector, 
perform a structural competitive analysis. You are NOT looking for reasons 
to kill the opportunity. You are looking for the structural weakness that 
justifies entering.

## Analysis framework

### Step 1: Identify all competitors
For each opportunity, use `tavily-search` and `tavily-extract`:
- `tavily-search`: "[opportunity domain] software"
- `tavily-search`: "[opportunity domain] startup"
- `tavily-search`: "[opportunity domain] site:ycombinator.com OR site:producthunt.com OR site:g2.com"
- `tavily-search`: "[opportunity domain] funding announcement 2025 2026"
- `tavily-extract` on competitor websites and review pages for detailed data

Classify each competitor as:
- Direct: Solving the exact same problem for the exact same buyer
- Adjacent: Solving a related problem, could expand into this space
- Potential: Large player with resources who might enter

### Step 2: For each direct competitor, assess

**Traction signals:**
- Estimated user count (ARR, employee count, funding stage as proxies)
- Review count and average rating
- Growth trajectory (Alexa rank trend, hiring velocity)

**Structural weakness assessment:**
Apply the 5-weakness framework from knowledge/05-competition-reading.md:
1. Legacy architecture weakness?
2. Business model misalignment?
3. Customer segment focus gap?
4. Distribution lock limiting reach?
5. Organizational inertia preventing response?

### Step 3: Wedge identification
Based on the structural weaknesses, identify the specific wedge:

```
Wedge statement: "We enter through [specific segment or use case] 
because incumbents cannot serve it well due to [specific structural weakness], 
and this gives us [specific advantage] that expands to [larger market]."
```

---

## Output format

```
Opportunity ID: [from Signal Scanner or Gap Detector]
Competitor name: [name]
Funding stage: [Seed/A/B/Public/Unknown]
Estimated users: [range or "unknown"]
Average rating: [source + score]
Structural weakness: [which of the 5 types + explanation]
Wedge available: [Yes/No + description]
Penetration rate estimate: [% of TAM currently served]
Threat level if we enter: [Low/Medium/High]
Conclusion: [Is this a viable entry given competition? Y/N + reason]
```
```

---

### 3.10 `agents/timing-judge.md`

```markdown
---
name: timing-judge
description: >-
  Evaluates timing maturity for candidate opportunities using structural 
  signals. Runs sequentially after Phase 1 sub-agents complete.
tools: Read, Glob, Grep, WebFetch, WebSearch
model: inherit
background: false
---

# Timing Judge Sub-Agent

## Role
You evaluate whether now is the right time to enter each opportunity that 
survived Phase 1. You produce a timing score (1–5) with specific evidence 
for each dimension.

## Evaluation process

For each candidate opportunity, investigate:

### Dimension 1: Technology trigger
Question: Did a relevant AI capability, API, or infrastructure cost change 
significantly in the last 24 months?

How to check:
- LLM API pricing history (OpenAI, Anthropic, Google — all have published 
  pricing going back 2+ years)
- Major model capability announcements (GPT-4, Claude 3, Gemini Ultra)
- Open source model availability (Llama 2 → 3, Mistral releases)
- Compute cost trends (AWS/GCP pricing history)

Scoring:
- 5: Cost dropped ≥10x OR new capability that didn't exist 24 months ago
- 3: Meaningful improvement but not threshold-crossing
- 1: No relevant technology change

### Dimension 2: Regulatory trigger
Question: Has a relevant law, rule, or compliance deadline emerged 
in the last 24 months?

How to check:
- Industry-specific regulatory news in the last 24 months
- Compliance deadline calendars
- Recent enforcement actions and fines

Scoring:
- 5: Hard compliance deadline in next 12 months with significant penalty
- 3: New guidance or rule that creates pressure but no hard deadline
- 1: No regulatory change

### Dimension 3: Behavioral trigger
Question: Has a large group of relevant buyers irreversibly changed behavior 
in the last 24 months?

How to check:
- Google Trends for relevant keywords (12–24 month trajectory)
- Industry survey data on workflow adoption
- Reddit community growth in relevant spaces

Scoring:
- 5: Clear behavioral inflection visible in multiple data sources
- 3: Gradual shift with some evidence
- 1: No clear behavioral change

### Dimension 4: Market validation signal
Question: Has the market been validated by credible third parties recently?

How to check:
- Recent VC funding in adjacent spaces
- Acquisition of companies in this space
- Large enterprises launching internal initiatives in this space

Scoring:
- 5: Multiple VC rounds in adjacent space + major enterprise initiative
- 3: Some investment signal but not strong
- 1: No third-party validation

---

## Timing score calculation

Average of the 4 dimension scores.

Interpretation:
- 4.0–5.0 = Strong timing → advance to output
- 3.0–3.9 = Acceptable timing → note weakness in memo
- 2.0–2.9 = Weak timing → flag as "Revisit in 12 months"
- 1.0–1.9 = Wrong timing → Kill

---

## Output format

```
Opportunity ID: [reference]
Technology trigger score: [1–5] — Evidence: [specific data point]
Regulatory trigger score: [1–5] — Evidence: [specific data point]
Behavioral trigger score: [1–5] — Evidence: [specific data point]
Market validation score: [1–5] — Evidence: [specific data point]
Overall timing score: [average]
Timing verdict: [Strong / Acceptable / Weak / Wrong]
Key timing insight: [1–2 sentences on the most important timing factor]
```
```

---

### 3.11 `templates/opportunity-memo.md`

```markdown
# Opportunity Memo Template

## Usage
This template is filled once per surviving opportunity after all agents 
have run. It is the final output delivered to the user.

---

# Opportunity Scout Report
**Date**: [date]
**Opportunities found**: [count]
**Sources scanned**: [list of sources]

---

## Opportunity #[rank]: [Short name]

### One-line hypothesis
[Problem] + [specific buyer] + [why now achievable] — one sentence.

### The change that created this
What specifically changed in the last 12–24 months that makes this opportunity 
exist now when it didn't before? 

### Target buyer
- **Role**: [specific job title or persona]
- **Company type**: [size, industry]  
- **How to reach first 10**: [specific channel: Reddit community, LinkedIn search, 
  conference, referral network]
- **Budget signal**: [evidence that they have budget for this]

### Current alternatives and why they fail
| Alternative | Who uses it | Specific failure mode |
|-------------|-------------|----------------------|
| [name] | [who] | [how it fails] |

### Evidence base
All sources that confirm this pain is real:
1. [Source + verbatim quote or data point]
2. [Source + verbatim quote or data point]
3. [Source + verbatim quote or data point]
(Minimum 3 sources required)

### Scoring

| Dimension | Score (1–5) | Reasoning |
|-----------|-------------|-----------|
| Pain strength | | |
| Buyer clarity | | |
| Timing maturity | | |
| Wedge quality | | |
| Buildability | | |
| **Total** | **/25** | |

**Weighted score**: [calculate using weights from SKILL.md]

### Wedge statement
"We enter through [specific angle] because [incumbent weakness], 
giving us [specific advantage], expandable to [larger market]."

### Validation plan (next 2 weeks)
1. [Specific action — e.g., "Post in r/[subreddit] asking for 5 
   interview volunteers using exact language: ..."]
2. [Specific action]
3. [Specific action]

### Recommendation
- [ ] **Kill** — Weighted score < 2.5 OR failed a kill filter
- [ ] **Revisit** — Weighted score 2.5–3.5, specific gap: [what needs to change]
- [ ] **Proceed** — Weighted score > 3.5, run validation plan immediately

---
[Repeat block for each opportunity]
---

## Scout session summary

**Scanned sources**: [list]
**Total signals found**: [number]
**Killed in Phase 1**: [number + common reason]
**Killed on timing**: [number + common reason]
**Surviving opportunities**: [number]
**Highest scored**: [name + score]
**Recommended next action**: [single most important thing to do today]
```

---

### 3.12 `memory/opportunity-log.md`

> ✅ **已改進 — 從 Markdown 表格改為 Append-only 條目格式**
> 原始設計用 markdown 表格存儲記憶，存在以下風險：
> - Claude 追加表格行時容易欄位錯位、管線符號漏掉
> - 表格越長越難維護
>
> 改用 **append-only 列表條目格式**：Claude 只需在對應區塊底部
> 新增一個條目，不需要精確對齊表格欄位，大幅降低格式破壞風險。

```markdown
# Opportunity Scout Memory Log

This file is read and updated by the scout at the start and end of every session.
It prevents re-investigating dismissed areas and tracks evolving signals.

## ⚙️ Memory 操作規則

1. **先讀後寫**：每次 session 開始時先完整讀取此文件，避免覆蓋已有記錄
2. **只追加不覆寫**：在對應區塊底部新增條目，不要重寫整個文件
3. **大小上限**：當文件超過 500 行時，將 6 個月前的 Dismissed 條目
   摘要合併為一條「[日期區間] 批次摘要」條目
4. **格式保護**：如果讀取時發現格式損壞，先修復再追加新內容

---

## Dismissed（不再調查）

在此區塊底部追加條目，格式如下：

- **[領域名稱]** — dismissed [日期]
  - 原因：[dismiss 原因]
  - 重新調查條件：[什麼改變了就值得重新看]

<!-- 在此行上方追加新的 dismissed 條目 -->

---

## Monitoring（有潛力但尚未成熟）

在此區塊底部追加條目，格式如下：

- **[領域名稱]** — first seen [日期] — signal strength: [weak/medium/strong]
  - 需要的變化：[什麼條件滿足就升級為 candidate]
  - 下次檢查：[日期]

<!-- 在此行上方追加新的 monitoring 條目 -->

---

## Handed off（已交付用戶）

在此區塊底部追加條目，格式如下：

- **[機會名稱]** — handed off [日期] — score: [X.XX/5.00]
  - 報告位置：[檔案路徑]
  - 用戶決策：[pending/pursuing/passed]

<!-- 在此行上方追加新的 handed off 條目 -->

---

## Source performance（來源品質追蹤）

每次 session 結束時更新計數：

- **YC batch**: used [N] times, high-quality signals: [N]
- **Product Hunt**: used [N] times, high-quality signals: [N]
- **Reddit**: used [N] times, high-quality signals: [N]
- **HN**: used [N] times, high-quality signals: [N]
- **App Store reviews**: used [N] times, high-quality signals: [N]
- **GitHub Issues**: used [N] times, high-quality signals: [N]
```

---

## 4. How Claude Code should build this

### Step 0 — Install Tavily MCP（必須先完成）
```bash
claude mcp add --transport http tavily https://mcp.tavily.com/mcp/?tavilyApiKey=YOUR_TAVILY_API_KEY
```
Verify Tavily is accessible by running a test search within Claude Code.
If Tavily is not installed, the Skill's search capabilities will not function.

### Step 1
Create the folder structure exactly as specified in Section 2.

### Step 2
Write each file with the exact content specified in Section 3.
Knowledge files 已經是精簡的 checklist 格式，直接照寫即可。
不要再展開成教科書式的長文。

### Step 3
Verify the SKILL.md frontmatter is valid YAML. 
Name must be: `opportunity-scout`
Effort must be: `high`

### Step 4
Test by running:
```bash
claude --skill opportunity-scout "scan for AI opportunities in healthcare"
```
And verify the agent executes all four sub-agents and produces an 
opportunity memo using the template.

---

## 5. Design principles for future iteration

1. **Knowledge files are checklists, not textbooks** — 保持精簡的
   「規則 + 範例」格式。如果 agent 分析淺薄，增加更精準的規則和範例，
   不要增加理論解釋。每條規則應可直接執行。

2. **Sub-agents are the hands** — if the agent is searching the wrong places, 
   improve the agent files.

3. **Memory prevents waste** — always update the memory log at the end of 
   each session so patterns compound over time.

4. **Evidence, not opinion** — every claim in every output must trace back 
   to a named, accessible source. If evidence is weak, say so explicitly.

5. **Kill early, kill cheap** — the kill filters exist to eliminate weak 
   opportunities before spending time on deep analysis. Do not skip them.
