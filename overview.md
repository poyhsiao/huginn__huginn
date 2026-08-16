---
title: Huginn Overview
tags:
  - AI_collection
  - huginn__huginn
created: 2026-08-16
source: https://github.com/huginn/huginn
---

# Huginn Overview

## 基本資訊

| 欄位 | 數值 |
|------|------|
| **代號** | huginn__huginn |
| **GitHub** | https://github.com/huginn/huginn |
| **描述** | Create agents that monitor and act on your behalf. Your agents are standing by! |
| **License** | MIT |
| **Stars** | 49,798 |
| **Forks** | 4,285 |
| **Watchers** | 49,798 |
| **Open Issues** | 696 |
| **建立時間** | 2013-03-10 |
| **更新時間** | 2026-08-16 |
| **Latest Release** | v2022.08.18 (2022-08-18) |
| **Contributor Count** | 20 (paginated) |

## Topics

`agent` `automation` `feed` `feedgenerator` `huginn` `monitoring` `notifications` `rss` `scraper` `twitter` `twitter-streaming` `webscraping`

## 簡述

Huginn 是一個用於在自有伺服器上建立自動化代理（Agent）的系統，可視為 **IFTTT 或 Zapier 的開源可破解版本**。使用者在自己的基礎設施上運行，永遠掌控自己的資料。

Huginn 的核心概念是 **Agent** — 代理程式會讀取網頁、監控事件、代表用戶執行操作。Agents 創建和消費 Events，並沿有向圖傳播。

## 主要功能摘要

- 追蹤天氣、Twitter 趨勢、網站變更
- 連接到 HipChat、FTP、IMAP、Jabber、JIRA、MQTT、Pushbullet、Pushover、RSS、Bash、Slack、StubHub、翻譯 API、Twilio、Twitter、Weibo 等
- 發送摘要郵件、定時通知
- 發送和接收 Webhooks
- 執行自訂 JavaScript 函數
- 追蹤位置
- 建立 Amazon Mechanical Turk 工作流
- RSS/Atom 閱讀
- Web 爬蟲

## 部署方式

- **Docker**（官方鏡像：`ghcr.io/huginn/huginn`）
- **Heroku**（一鍵部署按鈕，需付費 plan）
- **OpenShift Online**
- **手動安裝**（任何 Linux 伺服器）

## 快速開始

```bash
# Docker 快速啟動
docker run -it -p 3000:3000 ghcr.io/huginn/huginn

# 本機開發
cp .env.example .env
bundle exec rake db:create db:migrate db:seed
bundle exec foreman start
# 登入：admin / password
```

## 技術棧

- **後端**：Ruby on Rails + JavaScript Workers
- **資料庫**：PostgreSQL / MySQL
- **部署**：Heroku、Docker、手動

## License

MIT License
