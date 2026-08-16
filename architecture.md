---
title: Huginn Architecture
tags:
  - AI_collection
  - huginn__huginn
created: 2026-08-16
source: https://github.com/huginn/huginn
---

# Huginn Architecture

## 整體架構

Huginn 採用經典的 **Rails + Background Worker** 架構：

```
┌─────────────────────────────────────────┐
│              Rails Server               │
│  ┌─────────────────────────────────┐   │
│  │   Web UI (ERB + JavaScript)     │   │
│  │   Agent Management / Events     │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │   ActionCable (WebSocket)       │   │
│  │   Real-time Agent Status        │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
          │              │
          ▼              ▼
┌──────────────┐  ┌──────────────────┐
│  Job Workers  │  │  Scheduler       │
│  (Sidekiq)   │  │  (Sidekiq-scheduler) │
└──────────────┘  └──────────────────┘
          │              │
          ▼              ▼
┌─────────────────────────────────────────┐
│         PostgreSQL / MySQL              │
│    (Agents, Events, Credentials)       │
└─────────────────────────────────────────┘
```

---

## Rails + JavaScript Workers

### 後端：Ruby on Rails

- **框架**：Ruby on Rails（主應用程式）
- **職責**：Web UI、API endpoints、Agent 管理介面、Credential 管理
- **視圖**：ERB 模板 + Vanilla JavaScript（非 SPA）
- **WebSocket**：ActionCable 實現即時 agent 狀態更新

### 前端：JavaScript

- 使用 jQuery 管理 DOM 操作
- 無現代化前端框架（React/Vue），保持簡單
- Agent 事件即時顯示透過 ActionCable

### Background Jobs：Sidekiq

- 基於 Redis 的 Ruby 後台任務庫
- **HuginnAgentRunnerJob**：執行 Agent 任務
- **Scheduler**：定時觸發 Agent（sidekiq-scheduler）
- **Procfile** 定義多進程：
  - `web`：Rails server
  - `worker`：Sidekiq worker
  - `scheduler`：sidekiq-scheduler 行程

---

## Database（PostgreSQL / MySQL）

### 支援的資料庫

- **PostgreSQL**（推薦用於生產環境）
- **MySQL**（支援但非首選）

### 主要資料表

| Model | 說明 |
|-------|------|
| `Agent` | Agent 配置、選項、调度設定 |
| `AgentEvent` | Agent 產生的事件 payload |
| `Credential` | 加密儲存的敏感資訊 |
| `User` | 使用者帳號、邀請碼 |
| `AgentType` | 內建 Agent 類型元數據 |

### Schema 特點

- `Agent#options` 為 JSON 欄位（Agent 特定配置）
- `AgentEvent#payload` 為 JSON 欄位（事件資料）
- 使用 `attr_encrypted` 加密 Credential 值

---

## Heroku One-Click Deploy

### 支援的 Heroku Plan

Huginn 官方支援付費 Heroku plan，不建議使用免費 plan：

| Plan 限制 | 說明 |
|-----------|------|
| 免費 plan 限制 | 每日 runtime 18 小時，強制睡眠 |
| 免費 Postgres | 限 10,000 行資料 |
| 免費 plan 建議 | 只用每 1/2/5 分鐘调度选项 |
| 推薦 Plan | 1GB RAM 付費 plan |

### One-Click 部署流程

1. 點擊 Heroku Deploy Button
2. 填寫必填環境變數（`APP_SECRET_TOKEN`、`INVITATION_CODE`）
3. 啟用 SendGrid addon（免費層）
4. 執行 `bin/setup_heroku` 向導腳本
5. 自動設定 `BUILDPACK_URL`、`PROCFILE_PATH`、`FORCE_SSL`

### Heroku 專用 Procfile

```
deployment/heroku/Procfile.heroku
```

設計為在單一 Heroku web worker 上運行。如需多 worker，需設定 `PROCFILE_PATH` 切換到標準 Procfile。

---

## Docker Compose 支援

### 官方 Docker 鏡像

- **Multi-process 鏡像**：`ghcr.io/huginn/huginn`
- **Single-process 鏡像**：`huginn/huginn-single-process`

### Docker Compose 架構

```yaml
services:
  huginn:
    image: ghcr.io/huginn/huginn
    ports:
      - "3000:3000"
    environment:
      - DATABASE_ADAPTER=postgresql
      - POSTGRES_HOST=db
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
```

### Single-Process vs Multi-Process

| 模式 | 優點 | 缺點 |
|------|------|------|
| **Single-process** | 簡單部署 | 擴展性有限 |
| **Multi-process** | 可獨立擴展 web/worker | 配置複雜 |

---

## 平行化任務執行

### Sidekiq 的並行模型

- Sidekiq 使用執行緒而非行程（低記憶體開銷）
- 預設並發度：25 個執行緒（可配置 `sidekiq['concurrency']`）
- 每個 Agent 任務作為一個 Sidekiq job 排入隊列

### Huginn 的任務排程

```
Scheduler fires → Agent becomes due →
HuginnAgentRunnerJob enqueued →
Sidekiq picks up job → Agent#receive called →
Agent emits events → Downstream agents triggered
```

### 限制

- Ruby MRI GIL 限制：CPU-bound 任務無法真正並行
- I/O-bound 任務（如 HTTP 請求）可有效並行
- `human_task` 和 `trigger` 類型的 Agent 適合 I/O 操作

---

## Long-Running Agent 支援

### HumanTaskAgent

Amazon Mechanical Turk 人力任務 Agent，支援長任務工作流：

- 任務創建 → 人力處理 → 結果收集 → 後續處理
- 不受限於標準 cron/interval 调度
- 使用 ` RutherfordService` 與 MTurk API 互動

### Event-Driven 而非 Time-Driven

需要長時間運行的 Agent 可選擇：

- `event_mode: true`（事件驅動，不依賴排程）
- 手動觸發（透過 UI 或 API）
- Webhook 觸發（外部系統發起）

---

## 環境變數配置

| 變數 | 用途 |
|------|------|
| `APP_SECRET_TOKEN` | Rails session 加密 |
| `DATABASE_ADAPTER` | `postgresql` 或 `mysql` |
| `POSTGRES_*` / `MYSQL_*` | 資料庫連接參數 |
| `REDIS_URL` | Sidekiq Redis 連接 |
| `SMTP_*` | 郵件發送配置 |
| `INVITATION_CODE` | 註冊邀請碼 |
| `ADDITIONAL_GEMS` | 額外 Agent Gem |
| `AGENT_LOG_LENGTH` | 每 Agent 保留日誌行數 |

---

## 部署架構總結

| 部署方式 | 適用場景 |
|---------|---------|
| **Heroku** | 快速原型/展示（需付費 plan） |
| **Docker** | 本機評估、簡化部署 |
| **Docker Compose** | 小規模生產 |
| **手動（Linux）** | 大規模生產環境 |
| **OpenShift** | 企業 Kubernetes 環境 |
