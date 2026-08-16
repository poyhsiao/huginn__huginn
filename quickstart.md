---
title: Huginn Quickstart
tags:
  - AI_collection
  - huginn__huginn
created: 2026-08-16
source: https://github.com/huginn/huginn
---

# Huginn Quickstart

## Heroku 快速部署（推薦）

### 一鍵部署

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy)

### 步驟

1. 點擊上方的 Deploy to Heroku 按鈕
2. 填寫 **App name**（可選）
3. 填寫 **INVITATION_CODE**（設定註冊邀請碼）
4. 填寫 **APP_SECRET_TOKEN**（Rails session 加密密鑰）
5. 啟用 **SendGrid Addon**（免費層，發送 Email）
6. 點擊 **Deploy app**
7. 等待建置完成（約 5-10 分鐘）
8. 訪問 `https://<app-name>.herokuapp.com/`
9. 輸入邀請碼登入

### Heroku 部署腳本

已存在本地 Huginn repo 的情況：

```bash
# 1. 安裝 Heroku Toolbelt
brew install heroku  # macOS

# 2. 登入
heroku login

# 3. 複製並配置環境變數
cp .env.example .env
# 編輯 .env，至少設定 APP_SECRET_TOKEN

# 4. 執行部署向導
bin/setup_heroku

# 5. 推送代碼
git push heroku master
```

### Heroku 環境變數

```bash
heroku config:set APP_SECRET_TOKEN=your-secret-token
heroku config:set SMTP_DOMAIN=your-domain.com
heroku config:set SMTP_USER_NAME=your-smtp-username
heroku config:set SMTP_PASSWORD=your-smtp-password
heroku config:set EMAIL_FROM_ADDRESS=you@example.com
```

### Heroku 限制提醒

> ⚠️ **免費 plan 限制多**：每日 runtime 18 小時、Postgres 限 10,000 行。建議使用付費 plan。

---

## Docker Compose 本機安裝

### 前置需求

- Docker Desktop 已安裝
- Docker Compose v2+

### 快速啟動

```bash
# 1. 啟動 Huginn 容器（附带內嵌 PostgreSQL + Redis）
docker run -it -p 3000:3000 ghcr.io/huginn/huginn

# 2. 開啟瀏覽器
open http://localhost:3000

# 3. 登入
#  username: admin
#  password: password
```

### Docker Compose 生產配置

```yaml
# docker-compose.yml
version: '3.8'
services:
  huginn:
    image: ghcr.io/huginn/huginn
    ports:
      - "3000:3000"
    environment:
      - APP_SECRET_TOKEN=your-secret-token
      - DATABASE_ADAPTER=postgresql
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DATABASE=huginn
      - POSTGRES_USERNAME=huginn
      - POSTGRES_PASSWORD=huginn_password
      - REDIS_URL=redis://redis:6379
      - SMTP_DOMAIN=smtp.example.com
      - SMTP_USER_NAME=user
      - SMTP_PASSWORD=password
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=huginn
      - POSTGRES_USER=huginn
      - POSTGRES_PASSWORD=huginn_password

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

```bash
# 啟動
docker-compose up -d

# 查看日誌
docker-compose logs -f huginn
```

---

## 本機 Development 環境

### 前置需求

| 軟體 | 版本要求 |
|------|---------|
| **Ruby** | 3.x |
| **Node.js** | 18+ |
| **PostgreSQL** 或 **MySQL** | 最新穩定版 |
| **Redis** | 7.x |
| **ChromeDriver** | （用於 feature specs，可選） |

### macOS 安裝步驟

```bash
# 1. 安裝 Homebrew（如果還沒有）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. 安裝依賴
brew install ruby postgres redis nodejs

# 3. 啟動服務
brew services start postgresql
brew services start redis

# 4. Clone 並進入專案
git clone https://github.com/huginn/huginn.git
cd huginn

# 5. 安裝 Ruby 依賴
bundle install

# 6. 配置環境變數
cp .env.example .env
# 編輯 .env，至少更新 APP_SECRET_TOKEN

# 7. 建立資料庫
bundle exec rake db:create db:migrate db:seed

# 8. 啟動開發伺服器
bundle exec foreman start

# 9. 開啟瀏覽器
open http://localhost:3000

# 預設登入資訊
# username: admin
# password: password
```

### 使用 PostgreSQL

```bash
# 設定環境變數
export DATABASE_ADAPTER=postgresql
export POSTGRES_HOST=localhost
export POSTGRES_USERNAME=your_user
export POSTGRES_PASSWORD=your_password
```

---

## 第一個 Agent 教學

### 目標：建立天氣提醒

**流程**：每天早上 7 點檢查天氣，若有雨則發送 Email 提醒。

### 步驟 1：建立天氣 Agent

1. 點擊 **+ New Agent**
2. 選擇類型：**Weather Agent**
3. 填寫名稱：`Daily Weather Check`
4. 配置：
   ```json
   {
     "api_key": "your-pirate-weather-key",
     "location": "San Francisco, CA",
     "lat": 37.7749,
     "lng": -122.4194
   }
   ```
5. Schedule 設定：**Daily at 7am**
6. 點擊 **Save**

### 步驟 2：設定天氣 API Key Credential

1. 前往 **Credentials** 頁面
2. 點擊 **+ New Credential**
3. 選擇類型：**Text**
4. 名稱：`pirate_weather_api_key`
5. 值：粘貼你的 Pirate Weather API Key
6. 在 Weather Agent 中引用：`{{credential pirate_weather_api_key}}`

### 步驟 3：建立 Trigger Agent

1. 點擊 **+ New Agent**
2. 選擇類型：**Trigger Agent**
3. 名稱：`Rain Alert Trigger`
4. Source Agent 選擇：`Daily Weather Check`
5. Trigger 條件：
   ```json
   {
     "reason": "{{event.payload.daily.data.0.precipProbability}} > 0.5"
   }
   ```
6. 點擊 **Save**

### 步驟 4：建立 Email Agent

1. 點擊 **+ New Agent**
2. 選擇類型：**Email Agent**
3. 名稱：`Rain Alert Email`
4. Source Agent 選擇：`Rain Alert Trigger`
5. 設置 SMTP Credential（credentials 頁面）
6. 郵件內容：
   ```
   Subject: Don't forget your umbrella! ☔

   Tomorrow's weather in San Francisco:
   - Precipitation Probability: {{event.payload.daily.data.0.precipProbability}}%
   - Temperature: {{event.payload.daily.data.0.temperatureHigh}}°F
   ```
7. 點擊 **Save**

### 步驟 5：測試 Agent

1. 選中 `Daily Weather Check` Agent
2. 點擊 **Run Agent**（手動觸發一次）
3. 查看 **Events** 標籤，確認收到天氣事件
4. 檢查 `Rain Alert Trigger` 是否正確觸發
5. 檢查 Email 是否收到

### 測試 Webhook 觸發

```
# 本地環境
curl -X POST http://localhost:3000/users/web_requests/1/your-webhook-secret

# Heroku
curl -X POST https://your-app.herokuapp.com/users/web_requests/1/your-webhook-secret
```

---

## 基本概念：資料流向

```
┌─────────────────────────────────────────────────────────────┐
│  Trigger  →  Options  →  Source  →  Targets                │
│                                                             │
│  Scheduler │Webhook │ Event ──▶ Agent ──▶ Emit Event ──▶下游Agent │
│   (觸發)    (外部)   (事件)   (處理)    (新事件)    (繼續傳播)   │
└─────────────────────────────────────────────────────────────┘
```

### 核心 Flow

| 階段 | 說明 |
|------|------|
| **Trigger** | Agent 何時被執行（Schedule、Webhook、Event） |
| **Options** | Agent 的配置參數（JSON 格式） |
| **Source** | 事件來源（上一個 Agent 輸出的事件） |
| **Targets** | 事件去向（發送到哪個 Agent 或外部系統） |

### Liquid 模板引用

```
{{agent.name}}          # 當前 Agent 名稱
{{event.payload.key}}  # 事件中的某個欄位
{{credential name}}     # 引用已儲存的 Credential
{{#if condition}}...{{/if}}  # 條件判斷
```
