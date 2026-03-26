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

- **YC batch**: used 0 times, high-quality signals: 0
- **Product Hunt**: used 0 times, high-quality signals: 0
- **Reddit**: used 0 times, high-quality signals: 0
- **HN**: used 0 times, high-quality signals: 0
- **App Store reviews**: used 0 times, high-quality signals: 0
- **GitHub Issues**: used 0 times, high-quality signals: 0
