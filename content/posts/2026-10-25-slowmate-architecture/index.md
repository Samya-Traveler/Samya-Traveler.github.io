---
title: "SlowMate 技術架構：用 GitHub Pages＋Azure Functions＋Azure OpenAI 打造低成本 AI 行程規劃服務"
slug: "slowmate-architecture"
date: 2026-10-25T10:00:00+08:00
lastmod: 2026-10-26T10:00:00+08:00
draft: false
description: "SlowMate AI 慢旅行程規劃助手的技術架構拆解：GitHub Pages 靜態前端、Azure Functions v4（TypeScript）無伺服器 API、Azure Table Storage、Azure OpenAI、Gmail SMTP，以及 Portaly＋Logic App 的自動發碼流程。說明雙水池每日名額、Email OTP、多次數序號與 ETag 樂觀鎖的設計。"
summary: "一個人維護的 AI 小工具，怎麼做到免伺服器、低成本又不怕被濫用？拆解 SlowMate 的架構：靜態前端、無狀態 API、Table Storage、雙水池名額與多次數序號。"
categories: ["旅行工具"]
tags: ["SlowMate", "Azure", "Azure Functions", "Azure OpenAI", "GitHub Pages", "Serverless", "AI 應用", "系統架構"]
author: "Samya"
ShowToc: true
TocOpen: true
comments: true

cover:
  image: "cover.jpg"
  alt: "SlowMate 系統架構圖"
  relative: true
  hiddenInSingle: true
---

{{< hero src="cover.jpg" title="SlowMate 技術架構" subtitle="一個人維護、盡量不花錢、也不怕被濫用的 AI 小工具。" position="center 30%" >}}

這篇是 [SlowMate 是什麼？]({{< relref "posts/2026-10-25-slowmate-ai-slow-travel-planner" >}}) 的技術篇，
給對「一個人怎麼做出 AI 服務」有興趣的朋友。不需要很懂程式也能看懂大概，細節的部分可以跳過。

<!--more-->

> [!NOTE] 設計目標
> 1. **一個人維護得起**：不要有要顧的伺服器。
> 2. **成本可控**：AI 呼叫要錢，要能設上限。
> 3. **對使用者友善**：免下載、免註冊；失敗不扣額度。
> 4. **防止濫用**：免費額度不能被無限刷，序號不能被重複核銷。

---

## 架構總覽

{{< figure src="gallery/01-architecture.jpg" caption="SlowMate 架構圖：前端、無狀態 API、資料儲存、外部服務四層" align="center" >}}

| 層 | 用什麼／負責什麼 |
|---|---|
| 🖥️ 前端 | **GitHub Pages**（單一靜態 HTML）<br>表單、行程畫面、分享圖、列印 PDF |
| ⚙️ 後端 | **Azure Functions v4（TypeScript）**<br>規劃、OTP、寄信、產碼、管理統計 5 支 API |
| 🗄️ 儲存 | **Azure Table Storage**<br>每日用量、免費資格、序號、驗證碼 |
| 🌐 外部 | **Azure OpenAI（gpt-4.1）**、**Gmail SMTP**<br>**Portaly＋Azure Logic App** |
{.trip-facts}

---

## 1. 前端：一個靜態 HTML 檔

SlowMate 的前端就是部落格裡的一個檔案 `static/slowmate/index.html`，跟著這個 Hugo 部落格一起部署到 **GitHub Pages**。

- **沒有框架**：原生 HTML／CSS／JavaScript，插圖是內嵌 SVG，載入快、沒有建置步驟。
- **行程存在瀏覽器**：產生的行程存在 `localStorage`，重新整理不會不見，也不需要帳號。
- **範例行程是純前端資料**：京都、清邁兩份範例直接寫在頁面裡，點了就渲染，**不呼叫 API、不扣額度**。
- **分享圖用 Canvas 畫**：用原生 HTML5 Canvas 把行程畫成 **9:16（IG 限動）**和 **4:5（Threads / FB）**的雜誌風圖片，再用 **Web Share API** 直接叫出手機的分享選單。
- **列印就是 PDF**：用 `@media print` 把所有天數展開，瀏覽器「列印 → 另存 PDF」就是一本乾淨的手冊。
- **導航連結**：每個地點組成 Google Maps 搜尋網址，不需要串地圖 API。

---

## 2. 後端：5 支無狀態 API（Azure Functions）

- **`/api/planTrip`**：查詢今日名額（GET）、提交行程（POST）。負責雙水池名額控管、序號核銷、呼叫 AI。
  <br><small>讀寫：UserProfiles、DailyUsage、RedeemCodes</small>
- **`/api/sendOtp`**：攔截短時間內重複寄送、寄出 6 位數驗證碼。
  <br><small>讀寫：EmailOtps</small>
- **`/api/sendEmail`**：把行程排成響應式 HTML 信件寄出（前端另有 60 秒冷卻）。
- **`/api/generateCode`**：接收 Logic App 的請求，依贊助金額產生對應次數的序號並寄出。
  <br><small>讀寫：RedeemCodes</small>
- **`/api/adminStats`**：管理用的統計與批次產碼，需要後端驗證的 `x-admin-secret` 標頭。
  <br><small>讀寫：DailyUsage、RedeemCodes</small>

用 Functions 的好處是**沒人用就幾乎不花錢**，也不用自己管機器、打補丁。
每支 API 都是**無狀態**的，所有狀態都放在 Table Storage，方便除錯。

---

## 3. 主要流程

### 流程 ①：第一次免費使用（Email OTP）

{{< timeline title="免費規劃一次" >}}
{{< stop title="輸入 Email，請求驗證碼" icon="✉️" place="/api/sendOtp" >}}
檢查是否頻繁請求，產生 6 位數驗證碼寫入 **EmailOtps**（10 分鐘過期），用 Gmail SMTP 寄出。
{{< /stop >}}
{{< stop title="輸入驗證碼，送出規劃" icon="🔐" place="/api/planTrip" >}}
驗證 OTP → 檢查 **UserProfiles** 是否已用過免費額度 → 檢查 **DailyUsage** 今日免費席是否還有名額。
{{< /stop >}}
{{< stop title="呼叫 Azure OpenAI 產生行程" icon="🤖" >}}
成功才把「已使用免費額度」與「今日用量 +1」寫回資料表；**AI 失敗就不扣任何額度**。
{{< /stop >}}
{{< /timeline >}}

### 流程 ②：贊助自動發碼（Portaly → Logic App）

{{< timeline title="贊助自動發碼" >}}
{{< stop title="使用者在 Portaly 完成贊助" icon="☕" >}}
NT$ 88／300／488 三種方案。
{{< /stop >}}
{{< stop title="Azure Logic App 收到結帳通知" icon="🔁" >}}
觸發 `/api/generateCode`，帶上方案資訊。
{{< /stop >}}
{{< stop title="依方案產生序號並寄信" icon="🔑" place="/api/generateCode" >}}
NT$ 88 → **1 次**、NT$ 300 → **5 次**、NT$ 488 → **10 次**。序號寫入 **RedeemCodes**，信中附「一鍵帶入」連結，1～3 分鐘內送達。
{{< /stop >}}
{{< /timeline >}}

### 流程 ③：用序號規劃

持有序號的人**不需要 OTP**，`/api/planTrip` 會先確認序號還有剩餘次數，**AI 成功產生行程後才扣 1 次**。
如果中途失敗或斷線，次數不會被扣，前端也會明確告訴使用者「序號未被核銷」。
如果這個信箱**還有首次免費額度**，會優先使用免費額度，保留序號。

---

## 4. 幾個設計考量

### 雙水池每日名額：免費 50 席＋序號 30 席

AI 呼叫是按用量計費，所以用 **DailyUsage** 表記錄每天的次數（以台灣時間 UTC+8 每天 00:00 重置）：

- 🆓 **免費席 50 席**：首次免費使用者共用。
- ✨ **序號保留席 30 席**：只給持有序號的人用。免費席滿了，贊助者仍然排得到。

超過就回 HTTP 429，前端依情況顯示「免費名額已滿，可開啟 VIP 保留通道」或「今日全部額滿，明日請早」，並說明**不會扣使用者的額度**。
這是最簡單、也最可靠的**成本保險絲**，同時保障付費使用者的體驗。

### 多次數序號＋ETag 樂觀鎖

序號不再只是「一次性」，而是有幾個屬性：

- `max_uses`：可以使用的總次數（1／5／10，或活動碼自訂）
- `allow_same_email`：同一個信箱能不能重複使用（買家本人可以；分享給旅伴時每個信箱限 1 次）
- `campaign_tag`：活動標籤，方便統計不同活動的使用狀況

多人同時用同一組序號時，可能會「同時讀到剩 1 次、同時扣掉」。
所以更新序號時用 Azure Table Storage 的 **ETag 樂觀鎖**（`If-Match`）：寫入時帶上讀到的版本，版本不同就重試，**確保次數不會被多扣或少扣**。

### 「成功才扣」的交易順序

使用者最在意的是「我付了錢／用了免費額度，結果什麼都沒拿到」。
所以所有扣額度、核銷序號的動作，都放在 **AI 回傳成功之後**；失敗時回滾，前端也依錯誤碼（402、429、5xx）顯示不同的說明。

### 用結構化 JSON 讓 AI 輸出穩定

行程由 **Azure OpenAI（gpt-4.1）**產生，要求輸出固定欄位的 JSON，例如：

- `transport_passes`：交通票券與購買時機
- `advance_reservations`：需提前預約的項目與期限
- `traveler_persona`：慢旅人格標籤
- 每日的 `time_slot`、`title`、`transport_tag`、`is_must_visit`

前端只負責排版，**AI 負責內容、前端負責樣式**，兩邊可以分開調整。
提示詞裡也寫好了幾條規則：

- **節奏**：5 天以內每天三個時段、6～14 天每天兩個時段。
- **季節感知**：依旅行日期調整景點和穿搭建議（例如賞楓季、雨季）。
- **首尾日航班**：使用者填了抵達／離開時間，第一天從抵達後開始排，**最後一天在離開時間前結束**。

### 寄信用 Gmail SMTP

驗證碼、序號、行程信都用 **Nodemailer＋Gmail SMTP** 寄出，不需要購買網域或付費寄信服務。
缺點是偶爾會被歸到垃圾郵件，所以畫面上都有提醒使用者檢查垃圾郵件匣。

### 管理用的戰情室

在網址加上管理參數就能開啟管理面板，查看每天兩個水池的用量、各活動序號的使用排行，也能**批次產生活動碼**（例如抽獎碼、合作夥伴碼）。
權限**完全由後端驗證**：前端只是把金鑰放進 `x-admin-secret` 標頭送出，讀到後立刻從網址移除，不會寫死在頁面原始碼裡。

---

## 5. 為什麼不用「更完整」的做法？

- **會員系統、OAuth 登入** → 對一般旅人門檻太高；Email OTP＋序號已足夠防濫用。
- **月費訂閱、金流串接** → 一年出國幾次的人不需要訂閱；Portaly 處理收款更單純。
- **關聯式資料庫** → 資料結構簡單，Table Storage 便宜又免維護，ETag 也足以處理併發。
- **前端框架、App** → 一個 HTML 檔就夠用，免安裝也最友善。

這些選擇都是「**夠用就好**」。等使用的人變多、需求變複雜，再一步步調整。

---

## 歡迎回饋

如果你發現 bug、有架構上的建議，或想要什麼新功能，歡迎到 GitHub 開 issue：
👉 [github.com/Samya-Traveler/Samya-Traveler.github.io/issues](https://github.com/Samya-Traveler/Samya-Traveler.github.io/issues/new)

還沒用過的話，可以先去試試看 → [SlowMate](/slowmate/)，或看給一般旅人的介紹：
[SlowMate 是什麼？免下載、免註冊的 AI 慢旅行程規劃助手]({{< relref "posts/2026-10-25-slowmate-ai-slow-travel-planner" >}})。下一站見 🧸
