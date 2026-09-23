---
# Created with:  hugo new content posts/YYYY-MM-DD-your-slug/index.md
# Put photos next to this file (cover.jpg + gallery/*.jpg) — Hugo resizes them at build time.
title: "{{ replaceRE `^\d{4}-\d{2}-\d{2}-` "" .File.ContentBaseName | humanize | title }}"
slug: "{{ replaceRE `^\d{4}-\d{2}-\d{2}-` "" .File.ContentBaseName }}"
date: {{ .Date }}
lastmod: {{ .Date }}
draft: true
description: "一句話描述這趟旅程（顯示在標題下方與搜尋結果）"
summary: "文章列表卡片上的摘要"
categories: ["台灣"]
tags: []
destinations: []
author: "Samya"
ShowToc: true
comments: true

cover:
  image: "cover.jpg"
  alt: "封面圖片描述"
  relative: true
  hiddenInSingle: true

# resources:
#   - src: gallery/01-example.jpg
#     title: "照片說明"
---

{{< hero src="cover.jpg" title="旅程標題" subtitle="一句副標題" >}}

開場段落：這趟旅程的心情與亮點（這段會成為列表摘要）。

<!--more-->

## 旅程速覽

| 項目 | 內容 |
|---|---|
| 📍 地點 | |
| 🗓️ 天數 | |
| 🍁 最佳季節 | |
| 🚗 交通 | |
| 💰 預算 | |
{.trip-facts}

> [!TIP]
> 最重要的一個不踩雷小提醒。

## 行程時間軸

{{< timeline title="Day 1" >}}
{{< stop time="09:00" title="出發" icon="🚗" place="地點" >}}
這一站的細節（支援 **Markdown**）。
{{< /stop >}}
{{< stop time="12:00" title="午餐" icon="🍜" >}}
{{< /stop >}}
{{< /timeline >}}

## 照片集

{{< gallery match="gallery/*" cols="3" >}}

## 住宿與交通小筆記

> [!NOTE]
> 補充資訊。

> [!WARNING]
> 需要特別注意的事項。
