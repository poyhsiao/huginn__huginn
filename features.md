---
title: Huginn Features
tags:
  - AI_collection
  - huginn__huginn
created: 2026-08-16
source: https://github.com/huginn/huginn
---

# Huginn Features

## Agent System

Huginn 的核心是 **Agent** — 每個 Agent 是獨立的工作單元，能創建、轉發或消費事件。

### Agent 的四大概念

| 概念 | 說明 |
|------|------|
| **Instruction** | 定義 Agent 的行為邏輯（Ruby 程式碼或 JSON 配置） |
| **Trigger** | Agent 接收事件的方式（排程、Webhook、條件觸發） |
| **Action** | Agent 對事件的處理動作（發送郵件、發 Twitter、寫入 DB） |
| **Credential** | Agent 使用的敏感憑據（API Key、密碼、OAuth Token） |

### Agent 內部工作流

```
Trigger → Check Conditions → Emit Event → (optional) Action
```

Agent 透過 **Events** 相互通訊，形成有向圖結構。

---

## Webhook 系統

### Inbound Webhook（接收外部觸發）

- `WebhookAgent` 監聽外部 HTTP 請求
- 接收 POST/GET 請求作為事件觸發
- 支援 JSON、XML、Form-encoded payload
- 可與任何支援 Webhook 的服務整合

### Outbound Webhook（發送事件到外部）

- `WebhookAgent` 可主動發送 HTTP 請求
- `TriggerAgent` 可配置 HTTP callback
- 支援自訂 Headers、Body 模板

---

## Scheduler（排程系統）

Huginn 支援兩種排程模式：

### Cron 排程

- 透過 `SchedulerAgent` 或 Agent 內建的 `schedule` 選項
- 支援標準 cron 表達式：`*/5 * * * *`（每 5 分鐘）
- 適合定時任務（每日報告、每小時檢查）

### Interval 排程

- 相對時間間隔：`every: "1h"`, `every: "30m"`
- 適合需要固定頻率執行的任務
- `SchedulerAgent` 可同時觸發多個 Agent

### Long-Running Agent

- `HumanTaskAgent` 支援需要長時間運行的任務（如 Mechanical Turk）
- 不受限於標準排程，事件驅動執行

---

## Credentials 系統

Huginn 的 Credential 系統安全地儲存敏感資訊：

| 類型 | 用途 |
|------|------|
| **API Key** | 第三方服務 API Key（OpenAI、Pirate Weather 等） |
| **Password** | SMTP、IMAP、JIRA 等密碼認證 |
| **Text** | 任意文字（Webhook URL、私有筆記） |
| **OAuth** | OAuth 1.0/2.0 Token（Twitter、Google Calendar 等） |

Credentials 由 `CredentialsController` 管理，Agent 在執行時透過 `credential()` 方法引用。

---

## Events（事件系統）

### Agent Events

每個 Agent 可維護自己的事件隊列：
- 事件包含 `payload`（JSON 結構）和 `created_at` 時間戳
- `AgentEvent` 模型記錄所有事件（可配置保留數量）

### Event Propagation（事件傳播）

```
TriggerAgent → 產生 Event → Agent 消費 Event → 產生新 Event → ...
```

- 事件沿有向圖傳播
- `DeDuplicationAgent` 可過濾重複事件
- `DelayAgent` 可延遲事件傳播
- `GapDetectorAgent` 可偵測事件間隙

---

## Agent Types（內建 Agent 類型）

基於 gh api 列出 `app/models/agents/` 目錄，共約 70+ 個內建 Agent：

### Trigger Agents（觸發型）

| Agent | 說明 |
|-------|------|
| `SchedulerAgent` | 定時觸發其他 Agent |
| `WebhookAgent` | HTTP Webhook 觸發 |
| `TwitterSearchAgent` | Twitter 關鍵字搜尋觸發 |
| `TwitterStreamAgent` | Twitter 即時串流監聽 |
| `TwitterUserAgent` | 監控特定 Twitter 用戶 |
| `RSSAgent` | RSS/Atom Feed 監控 |
| `WebsiteAgent` | 網站變更偵測 |
| `HTTPStatusAgent` | HTTP 狀態碼監控 |
| `ChangeDetectorAgent` | 資料變更偵測 |
| `WeatherAgent` | 天氣變更觸發 |

### Action Agents（動作型）

| Agent | 說明 |
|-------|------|
| `Agent` (Base) | 一般事件處理 |
| `EmailAgent` | 發送 Email |
| `EmailDigestAgent` | 發送摘要郵件 |
| `SlackAgent` | 發送 Slack 訊息 |
| `TwitterPublishAgent` | 發布 Twitter |
| `LinkedInPublishAgent` | 發布 LinkedIn |
| `TelegramAgent` | 發送 Telegram 訊息 |
| `TwilioAgent` | 發送 SMS |
| `PushbulletAgent` | 發送 Pushbullet 通知 |
| `PushoverAgent` | 發送 Pushover 通知 |
| `WebhookAgent` | 發送 HTTP 請求 |
| `EvernoteAgent` | 寫入 Evernote |
| `GoogleCalendarPublishAgent` | 新增 Google Calendar 事件 |
| `DropboxFileUrlAgent` | 上傳 Dropbox |
| `S3Agent` | 上傳 AWS S3 |

### Processing Agents（處理型）

| Agent | 說明 |
|-------|------|
| `TriggerAgent` | 條件判斷/轉發 |
| `DigestAgent` | 收集事件後統一輸出 |
| `EventFormattingAgent` | 格式化事件 payload |
| `DeDuplicationAgent` | 去除重複事件 |
| `DelayAgent` | 延遲事件處理 |
| `CommanderAgent` | 動態觸發其他 Agent |
| `CsvAgent` | CSV 資料處理 |
| `AttributeDifferenceAgent` | 計算屬性差異 |
| `GapDetectorAgent` | 偵測事件缺口 |
| `SentimentAgent` | 情感分析 |
| `GoogleTranslationAgent` | 翻譯 |
| `JiraAgent` | JIRA 整合 |

### Cleanup/Utility Agents

| Agent | 說明 |
|-------|------|
| `CleanupAgent` | 清理舊事件 |
| `DataOutputAgent` | JSON/CSV 資料輸出 |
| `HumanTaskAgent` | Amazon Mechanical Turk 人力任務 |

---

## 第三方 Agent Gems

Huginn 支援將 Agent 封裝為外部 Ruby Gem，透過 `ADDITIONAL_GEMS` 環境變數載入：

```ruby
# See: https://github.com/huginn/huginn_agent
```

官方建議：複雜或特定領域的 Agent 應作為 Gem 開發，通用 Agent 放核心。

---

## Liquid 模板

Huginn 广泛使用 **Liquid 模板語法** 處理文字替換：

```
{{ agent.name }} - {{ event.payload.title }}
{{#if event.payload.price }}Price: {{ event.payload.price }}{{/if}}
```

---

## 自訂 JavaScript 與 Ruby

- `WebsiteAgent` 支援 `javascript` 欄位撰寫自訂抓取邏輯
- `ShellCommandAgent` 執行系統命令
- `Agent` 類別可被子類化擴展

---

## 事件保留策略

- `AgentEvent` 保留數量可透過 `AGENT_LOG_LENGTH` 環境變數控制
- `CleanupAgent` 可定期刪除超時事件
- `DeDuplicationAgent` 減少重複儲存
