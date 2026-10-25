---
title: "SlowMate 技術架構：用 GitHub Pages＋Azure Functions＋Azure OpenAI 打造低成本 AI 行程規劃服務"
slug: "slowmate-architecture"
date: 2026-10-25T10:00:00+08:00
lastmod: 2026-10-25T10:00:00+08:00
draft: false
description: "SlowMate AI 慢旅行程規劃助手的技術架構拆解：GitHub Pages 靜態前端、Azure Functions 無伺服器 API、Azure Table Storage、Azure OpenAI、Gmail SMTP 寄信，以及 Portaly＋Logic App 的贊助序號流程。說明每日名額、Email OTP、一次性序號的設計考量。"
summary: "一個人維護的 AI 小工具，怎麼做到免伺服器、低成本又不怕被濫用？拆解 SlowMate 的架構：靜態前端、無狀態 API、Table Storage 和 AI 呼叫的設計。"
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
> 4. **防止濫用**：免費額度不能被無限刷。

---

## 架構總覽

{{< figure src="gallery/01-architecture.jpg" caption="SlowMate 架構圖：前端、無狀態 API、資料儲存、外部服務四層" align="center" >}}

整個系統分成四層：

| 層 | 用什麼／負責什麼 |
|---|---|
| 🖥️ 前端 | **GitHub Pages**（單一靜態 HTML）<br>表單、行程畫面、列印 PDF |
| ⚙️ 後端 | **Azure Functions**（HTTP 觸發、無狀態）<br>規劃、OTP、寄信、產生序號 4 支 API |
| 🗄️ 儲存 | **Azure Table Storage**<br>每日用量、免費資格、序號、驗證碼 |
| 🌐 外部 | **Azure OpenAI**、**Gmail SMTP**<br>**Portaly＋Azure Logic App** |
{.trip-facts}

---

## 1. 前端：一個靜態 HTML 檔

SlowMate 的前端就是部落格裡的一個檔案 `static/slowmate/index.html`，跟著這個 Hugo 部落格一起部署到 **GitHub Pages**。

- **沒有框架**：原生 HTML／CSS／JavaScript，插圖是內嵌 SVG，載入快、沒有建置步驟。
- **行程存在瀏覽器**：產生的行程存在 `localStorage`，重新整理不會不見，也不需要帳號。
- **列印就是 PDF**：用 `@media print` 樣式把所有天數展開，瀏覽器「列印 → 另存 PDF」就是一本乾淨的手冊，不用另外做 PDF 產生器。
- **導航連結**：每個地點組成 Google Maps 搜尋網址，不需要串地圖 API。

---

## 2. 後端：4 支無狀態 API（Azure Functions）

- **`/api/planTrip`**：白名單、每日 50 次限額、序號核銷、呼叫 AI
  <br><small>讀寫：DailyUsage、UserProfiles、RedeemCodes</small>
- **`/api/sendOtp`**：防洗信箱、寄送 6 位數驗證碼
  <br><small>讀寫：EmailOtps</small>
- **`/api/sendEmail`**：把行程排成 HTML 信件寄出
- **`/api/generateCode`**：接收 Logic App 請求、產生序號並寄出
  <br><small>讀寫：RedeemCodes</small>

用 Functions 的好處是**沒人用就幾乎不花錢**，也不用自己管機器、打補丁。
每支 API 都是**無狀態**的，所有狀態都放在 Table Storage，方便水平擴展，也方便除錯。

---

## 3. 三個主要流程

### 流程 ①：第一次免費使用（Email OTP）

{{< timeline title="免費規劃一次" >}}
{{< stop title="輸入 Email，請求驗證碼" icon="✉️" place="/api/sendOtp" >}}
檢查是否頻繁請求（防洗信箱），產生 6 位數驗證碼寫入 **EmailOtps**（10 分鐘過期），用 Gmail SMTP 寄出。
{{< /stop >}}
{{< stop title="輸入驗證碼，送出規劃" icon="🔐" place="/api/planTrip" >}}
驗證 OTP → 檢查 **UserProfiles** 是否已用過免費額度 → 檢查 **DailyUsage** 今日是否未滿 50 次。
{{< /stop >}}
{{< stop title="呼叫 Azure OpenAI 產生行程" icon="🤖" >}}
成功才把「已使用免費額度」與「今日用量 +1」寫回資料表；**AI 失敗就不扣任何額度**。
{{< /stop >}}
{{< /timeline >}}

### 流程 ②：贊助換序號（Portaly → Logic App）

{{< timeline title="贊助自動發碼" >}}
{{< stop title="使用者在 Portaly 完成贊助" icon="☕" >}}
NT$ 88／150／300 三種金額，權益相同。
{{< /stop >}}
{{< stop title="Azure Logic App 收到結帳通知" icon="🔁" >}}
觸發 `/api/generateCode`。
{{< /stop >}}
{{< stop title="產生一次性序號並寄信" icon="🔑" place="/api/generateCode" >}}
序號寫入 **RedeemCodes**，信中附「一鍵帶入」連結，1～3 分鐘內送達。
{{< /stop >}}
{{< /timeline >}}

### 流程 ③：用序號規劃

`/api/planTrip` 收到序號後，先確認狀態是「未使用」，**AI 成功產生行程後才核銷**。
如果中途失敗或斷線，序號仍然有效，前端也會明確告訴使用者「序號未被核銷」。

---

## 4. 幾個設計考量

### 每日全站 50 次上限

AI 呼叫是按用量計費，所以用 **DailyUsage** 表記錄每天的總次數，超過就回 HTTP 429。
前端會顯示「今日名額已滿，明日請早」，並說明**不會扣使用者的額度**。
這是最簡單、也最可靠的**成本保險絲**。

### 「成功才扣」的交易順序

使用者最在意的是「我付了錢／用了免費額度，結果什麼都沒拿到」。
所以所有扣額度、核銷序號的動作，都放在 **AI 回傳成功之後**；失敗時回滾，前端也依錯誤碼（402、429、5xx）顯示不同的說明。

### 用結構化 JSON 讓 AI 輸出穩定

行程由 **Azure OpenAI（gpt-4.1）**產生，要求輸出固定欄位的 JSON，例如：

- `transport_passes`：交通票券與購買時機
- `advance_reservations`：需提前預約的項目與期限
- 每日的 `time_slot`、`title`、`transport_tag`、`is_must_visit`

前端只負責把 JSON 排版，**AI 負責內容、前端負責樣式**，兩邊可以分開調整。
節奏也寫在規則裡：**5 天以內每天三個時段、6～14 天每天兩個時段**。

### 寄信用 Gmail SMTP

驗證碼、序號、行程信都用 **Nodemailer＋Gmail SMTP** 寄出，不需要購買網域或付費寄信服務。
缺點是偶爾會被歸到垃圾郵件，所以畫面上都有提醒使用者檢查垃圾郵件匣。

---

## 5. 為什麼不用「更完整」的做法？

- **會員系統、OAuth 登入** → 對一般旅人門檻太高；Email OTP 已足夠防濫用。
- **月費訂閱、金流串接** → 一年出國幾次的人不需要訂閱；Portaly 處理收款更單純。
- **關聯式資料庫** → 資料結構簡單，Table Storage 便宜又免維護。
- **前端框架、App** → 一個 HTML 檔就夠用，免安裝也最友善。

這些選擇都是「**夠用就好**」。等使用的人變多、需求變複雜，再一步步調整。

---

## 歡迎回饋

如果你發現 bug、有架構上的建議，或想要什麼新功能，歡迎到 GitHub 開 issue：
👉 [github.com/Samya-Traveler/Samya-Traveler.github.io/issues](https://github.com/Samya-Traveler/Samya-Traveler.github.io/issues/new)

還沒用過的話，可以先去試試看 → [SlowMate](/slowmate/)，或看給一般旅人的介紹：
[SlowMate 是什麼？免下載、免註冊的 AI 慢旅行程規劃助手]({{< relref "posts/2026-10-25-slowmate-ai-slow-travel-planner" >}})。下一站見 🧸
