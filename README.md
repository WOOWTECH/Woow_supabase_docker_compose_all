# Woow Supabase Docker Compose All

**Supabase Self-Hosted Deployment with Docker Compose (Podman Compatible)**

**使用 Docker Compose 自架 Supabase（相容 Podman）**

---

## Table of Contents / 目錄

- [Overview / 概述](#overview--概述)
- [Architecture / 架構](#architecture--架構)
- [Services Explained / 服務說明](#services-explained--服務說明)
- [Prerequisites / 前置需求](#prerequisites--前置需求)
- [Quick Start / 快速開始](#quick-start--快速開始)
- [Configuration / 配置說明](#configuration--配置說明)
- [Common Commands / 常用指令](#common-commands--常用指令)
- [File Structure / 檔案結構](#file-structure--檔案結構)
- [Troubleshooting / 故障排除](#troubleshooting--故障排除)
- [Backup & Restore / 備份與還原](#backup--restore--備份與還原)
- [Podman Notes / Podman 注意事項](#podman-notes--podman-注意事項)
- [License / 授權](#license--授權)

---

## Overview / 概述

**English:**
This repository contains all configuration files needed to self-host [Supabase](https://supabase.com/) — the open-source Firebase alternative — using Docker Compose. The setup includes 12 services: PostgreSQL database, authentication, RESTful API, real-time subscriptions, file storage, edge functions, API gateway, connection pooling, log analytics, image processing, database metadata management, and a web dashboard.

This configuration has been verified to work with **Podman** (rootless mode) as a Docker-compatible container runtime.

**中文：**
本專案包含使用 Docker Compose 自架 [Supabase](https://supabase.com/)（開源 Firebase 替代方案）所需的全部配置文件。此設定包含 12 個服務：PostgreSQL 資料庫、身份認證、RESTful API、即時訂閱、檔案儲存、邊緣函數、API 閘道、連線池、日誌分析、圖片處理、資料庫元資料管理及 Web 儀表板。

此配置已在 **Podman**（無根模式）下驗證可正常運行。

---

## Architecture / 架構

```
                              ┌─────────────┐
                              │   Browser    │
                              │   瀏覽器     │
                              └──────┬───────┘
                                     │
                              ┌──────▼───────┐
                              │  Kong (API   │ Port 8100 (HTTP)
                              │  Gateway)    │ Port 8543 (HTTPS)
                              │  API 閘道    │
                              └──┬───┬───┬───┘
           ┌───────────┬────────┬┘   │   └┬────────┬───────────┐
           │           │        │    │    │        │           │
    ┌──────▼──┐ ┌──────▼──┐ ┌──▼────▼┐ ┌─▼──────┐ │    ┌──────▼──────┐
    │ Studio  │ │  Auth   │ │  REST  │ │Realtime│ │    │  Functions  │
    │ 儀表板  │ │  認證   │ │  API   │ │ 即時   │ │    │  邊緣函數   │
    │ :3000   │ │ :9999   │ │ :3000  │ │ :4000  │ │    │  :9000      │
    └────┬────┘ └────┬────┘ └───┬────┘ └───┬────┘ │    └─────────────┘
         │           │         │           │      │
         │    ┌──────▼─────────▼───────────▼──┐   │
         │    │      PostgreSQL (DB)           │   │
         │    │      資料庫 :5432              │   │
         │    └──────▲─────────▲───────────▲──┘   │
         │           │         │           │      │
    ┌────▼────┐ ┌────▼────┐ ┌──▼──────┐ ┌─▼──────▼──┐
    │  Meta   │ │Analytics│ │ Storage │ │ Supavisor  │
    │ 元資料  │ │  分析   │ │  儲存   │ │  連線池    │
    │ :8080   │ │  :4000  │ │  :5000  │ │ :5432/6543 │
    └─────────┘ └────▲────┘ └────┬────┘ └────────────┘
                     │           │
              ┌──────▼────┐ ┌───▼──────┐
              │  Vector   │ │ Imgproxy │
              │ 日誌收集  │ │ 圖片處理 │
              │  :9001    │ │  :5001   │
              └───────────┘ └──────────┘
```

---

## Services Explained / 服務說明

| Service / 服務 | Image / 映像 | Port / 埠 | Description (EN) | 說明 (中文) |
|---|---|---|---|---|
| **Studio** | `supabase/studio` | 3000 (internal) | Web dashboard for managing your Supabase project. Provides SQL editor, table viewer, auth management, and storage browser. | Web 管理儀表板，提供 SQL 編輯器、資料表瀏覽器、認證管理及儲存瀏覽功能。 |
| **Kong** | `kong:2.8.1` | 8100 (HTTP), 8543 (HTTPS) | API gateway / reverse proxy. Routes all API requests to the correct service. Handles authentication via API keys (anon/service_role). | API 閘道 / 反向代理。將所有 API 請求路由到正確的服務。通過 API 金鑰（匿名/服務角色）處理認證。 |
| **Auth (GoTrue)** | `supabase/gotrue` | 9999 (internal) | Authentication server. Handles user signup, login, JWT token issuance, email/phone verification, OAuth providers, and MFA. | 認證伺服器。處理使用者註冊、登入、JWT 令牌發行、電子郵件/手機驗證、OAuth 供應商及多因素認證。 |
| **REST (PostgREST)** | `postgrest/postgrest` | 3000 (internal) | Automatically generates a RESTful API from your PostgreSQL schema. Supports filtering, pagination, and embedded resources. | 自動從 PostgreSQL schema 生成 RESTful API。支援篩選、分頁及嵌入式資源。 |
| **Realtime** | `supabase/realtime` | 4000 (internal) | WebSocket server for real-time database change notifications. Enables live subscriptions to INSERT, UPDATE, DELETE events. | WebSocket 伺服器，提供即時資料庫變更通知。支援 INSERT、UPDATE、DELETE 事件的即時訂閱。 |
| **Storage** | `supabase/storage-api` | 5000 (internal) | File storage service with S3-compatible API. Manages file uploads, downloads, and access policies via Row Level Security. | 檔案儲存服務，具有 S3 相容 API。通過行級安全策略（RLS）管理檔案上傳、下載及存取權限。 |
| **Imgproxy** | `darthsim/imgproxy` | 5001 (internal) | On-the-fly image transformation. Resizes, crops, and converts images (WebP detection supported). | 即時圖片轉換。支援調整大小、裁剪及格式轉換（支援 WebP 偵測）。 |
| **Meta (pg-meta)** | `supabase/postgres-meta` | 8080 (internal) | PostgreSQL metadata API. Provides table/column/role/policy information for the Studio dashboard. | PostgreSQL 元資料 API。為 Studio 儀表板提供資料表/欄位/角色/策略資訊。 |
| **Functions (Edge Runtime)** | `supabase/edge-runtime` | 9000 (internal) | Deno-based edge function runtime. Executes serverless TypeScript/JavaScript functions. | 基於 Deno 的邊緣函數執行環境。執行無伺服器 TypeScript/JavaScript 函數。 |
| **Analytics (Logflare)** | `supabase/logflare` | 4000 | Log ingestion and analytics engine. Collects logs from all services and stores them in PostgreSQL. | 日誌收集及分析引擎。從所有服務收集日誌並儲存在 PostgreSQL 中。 |
| **Vector** | `timberio/vector` | 9001 (internal) | Log collection agent. Reads Docker/Podman container logs and routes them to the Analytics service. | 日誌收集代理。讀取 Docker/Podman 容器日誌並路由到分析服務。 |
| **Supavisor** | `supabase/supavisor` | 5432, 6543 | Database connection pooler. Manages PostgreSQL connections efficiently using transaction-level pooling. | 資料庫連線池。使用交易級連線池高效管理 PostgreSQL 連線。 |
| **DB (PostgreSQL)** | `supabase/postgres:15.8.1` | 5432 (internal) | PostgreSQL 15 database with Supabase extensions (pgvector, pg_net, pgsodium, etc.). The core data store. | PostgreSQL 15 資料庫，包含 Supabase 擴充套件（pgvector、pg_net、pgsodium 等）。核心資料儲存。 |

---

## Prerequisites / 前置需求

**English:**
- **Docker Engine** 20.10+ with Docker Compose v2, **or Podman** 4.0+ with `podman-compose` or `docker compose` compatibility
- **RAM:** 4 GB minimum (8 GB recommended)
- **Disk:** 10 GB minimum
- **openssl** (for key generation utility)
- **Ports available:** 8100, 8543, 4000, 5432, 6543

**中文：**
- **Docker Engine** 20.10+ 搭配 Docker Compose v2，**或 Podman** 4.0+ 搭配 `podman-compose` 或 `docker compose` 相容模式
- **記憶體：** 最少 4 GB（建議 8 GB）
- **磁碟空間：** 最少 10 GB
- **openssl**（用於金鑰生成工具）
- **可用埠：** 8100、8543、4000、5432、6543

---

## Quick Start / 快速開始

### 1. Clone the repository / 複製儲存庫

```bash
git clone https://github.com/WOOWTECH/Woow_supabase_docker_compose_all.git
cd Woow_supabase_docker_compose_all
```

### 2. Generate secrets / 產生密鑰

```bash
# Generate new JWT secrets, API keys, and passwords
# 產生新的 JWT 密鑰、API 金鑰和密碼
sh utils/generate-keys.sh
```

This will output all required secrets. When prompted, type `y` to automatically update the `.env` file.

這將輸出所有必要的密鑰。當提示時，輸入 `y` 自動更新 `.env` 檔案。

### 3. Create .env from template / 從模板建立 .env

```bash
# If you haven't used generate-keys.sh to auto-update:
# 如果你沒有使用 generate-keys.sh 自動更新：
cp .env.example .env
# Then edit .env with your own values
# 然後編輯 .env 填入你自己的值
```

### 4. Configure your domain (optional) / 設定你的網域（可選）

Edit `.env` and update these values if not using localhost:

編輯 `.env`，如果不使用 localhost 請更新以下值：

```bash
SITE_URL=https://your-domain.com
API_EXTERNAL_URL=https://your-domain.com
SUPABASE_PUBLIC_URL=https://your-domain.com
```

### 5. Pull images and start / 拉取映像並啟動

```bash
# Pull latest images / 拉取最新映像
docker compose pull

# Start all services / 啟動所有服務
docker compose up -d

# With dev tools (includes mail server) / 使用開發工具（包含郵件伺服器）
docker compose -f docker-compose.yml -f ./dev/docker-compose.dev.yml up -d
```

### 6. Access the dashboard / 存取儀表板

Open your browser and navigate to:

打開瀏覽器並前往：

```
http://localhost:8100
```

Login with the credentials set in `.env`:

使用 `.env` 中設定的帳號密碼登入：

- **Username:** `DASHBOARD_USERNAME` (default: `supabase`)
- **Password:** `DASHBOARD_PASSWORD`

---

## Configuration / 配置說明

### Key Environment Variables / 重要環境變數

| Variable / 變數 | Description (EN) | 說明 (中文) | Default / 預設值 |
|---|---|---|---|
| `POSTGRES_PASSWORD` | PostgreSQL superuser password | PostgreSQL 超級使用者密碼 | *(must set / 必須設定)* |
| `JWT_SECRET` | Secret for signing JWT tokens (min 32 chars) | JWT 令牌簽名密鑰（至少 32 字元） | *(must set / 必須設定)* |
| `ANON_KEY` | Public API key for anonymous access | 匿名存取的公開 API 金鑰 | *(generated / 產生)* |
| `SERVICE_ROLE_KEY` | Admin API key that bypasses RLS | 繞過 RLS 的管理員 API 金鑰 | *(generated / 產生)* |
| `DASHBOARD_USERNAME` | Studio login username | Studio 登入使用者名稱 | `supabase` |
| `DASHBOARD_PASSWORD` | Studio login password | Studio 登入密碼 | *(must set / 必須設定)* |
| `KONG_HTTP_PORT` | External HTTP port for API gateway | API 閘道外部 HTTP 埠 | `8100` |
| `KONG_HTTPS_PORT` | External HTTPS port for API gateway | API 閘道外部 HTTPS 埠 | `8543` |
| `SITE_URL` | Your application URL | 你的應用程式 URL | `http://localhost:3000` |
| `API_EXTERNAL_URL` | External URL for API access | API 外部存取 URL | `http://localhost:8100` |
| `SUPABASE_PUBLIC_URL` | Public URL for Studio dashboard | Studio 儀表板公開 URL | `http://localhost:8100` |
| `POOLER_PROXY_PORT_TRANSACTION` | Supavisor transaction pooling port | Supavisor 交易連線池埠 | `6543` |
| `POOLER_DEFAULT_POOL_SIZE` | Max PostgreSQL connections per pool | 每個連線池最大 PostgreSQL 連線數 | `20` |
| `POOLER_MAX_CLIENT_CONN` | Max client connections per pool | 每個連線池最大客戶端連線數 | `100` |
| `DOCKER_SOCKET_LOCATION` | Path to Docker/Podman socket | Docker/Podman socket 路徑 | `/var/run/docker.sock` |
| `FUNCTIONS_VERIFY_JWT` | Enable JWT verification for Edge Functions | Edge Functions 是否啟用 JWT 驗證 | `false` |
| `ENABLE_EMAIL_SIGNUP` | Allow email-based registration | 允許電子郵件註冊 | `true` |
| `ENABLE_PHONE_SIGNUP` | Allow phone-based registration | 允許手機號碼註冊 | `true` |
| `IMGPROXY_ENABLE_WEBP_DETECTION` | Auto-detect WebP browser support | 自動偵測瀏覽器 WebP 支援 | `true` |
| `OPENAI_API_KEY` | OpenAI key for SQL Editor AI assistant | SQL 編輯器 AI 助手的 OpenAI 金鑰 | *(optional / 可選)* |

### SMTP Configuration / SMTP 郵件設定

For email verification and password reset to work, configure a real SMTP server:

若要使電子郵件驗證和密碼重設正常運作，請設定真實的 SMTP 伺服器：

```bash
SMTP_ADMIN_EMAIL=noreply@your-domain.com
SMTP_HOST=smtp.your-provider.com
SMTP_PORT=587
SMTP_USER=your-smtp-username
SMTP_PASS=your-smtp-password
SMTP_SENDER_NAME=YourApp
```

> **Tip / 提示:** For development, use `docker-compose.dev.yml` which includes [Inbucket](https://inbucket.org/) — a test mail server with web UI at port 9000.
>
> 開發時，使用 `docker-compose.dev.yml`，它包含 [Inbucket](https://inbucket.org/) — 一個測試郵件伺服器，Web 介面在埠 9000。

---

## Common Commands / 常用指令

| Command / 指令 | Description (EN) | 說明 (中文) |
|---|---|---|
| `docker compose up -d` | Start all services in background | 在背景啟動所有服務 |
| `docker compose down` | Stop all services | 停止所有服務 |
| `docker compose ps` | List running containers | 列出執行中的容器 |
| `docker compose logs -f` | Follow all service logs | 跟蹤所有服務日誌 |
| `docker compose logs -f db` | Follow database logs only | 僅跟蹤資料庫日誌 |
| `docker compose logs -f auth` | Follow auth service logs | 跟蹤認證服務日誌 |
| `docker compose pull` | Pull latest images | 拉取最新映像 |
| `docker compose restart <service>` | Restart a specific service | 重啟特定服務 |
| `docker compose up -d --force-recreate` | Recreate all containers (after .env change) | 重建所有容器（修改 .env 後） |
| `docker compose -f docker-compose.yml -f ./dev/docker-compose.dev.yml up -d` | Start with dev tools | 使用開發工具啟動 |
| `./reset.sh` | Full reset: remove containers, volumes, and .env | 完全重設：移除容器、卷及 .env |
| `sh utils/generate-keys.sh` | Generate new secrets and API keys | 產生新的密鑰和 API 金鑰 |
| `sh utils/db-passwd.sh` | Change database password for all roles | 更改所有角色的資料庫密碼 |

### S3 Storage Backend / S3 儲存後端

To use MinIO (S3-compatible) storage instead of local file storage:

使用 MinIO（S3 相容）儲存取代本地檔案儲存：

```bash
docker compose -f docker-compose.yml -f docker-compose.s3.yml up -d
```

---

## File Structure / 檔案結構

```
.
├── docker-compose.yml              # Main service orchestration
│                                   # 主要服務編排檔案
├── docker-compose.s3.yml           # S3 storage backend extension (MinIO)
│                                   # S3 儲存後端擴展（MinIO）
├── .env.example                    # Environment variable template (safe to commit)
│                                   # 環境變數模板（可安全提交）
├── .gitignore                      # Git ignore rules
│                                   # Git 忽略規則
├── reset.sh                        # Full reset script (containers + data + .env)
│                                   # 完全重設腳本（容器 + 資料 + .env）
├── README.md                       # This documentation
│                                   # 本文件
├── utils/
│   ├── generate-keys.sh            # Generate JWT secrets, API keys, passwords
│   │                               # 產生 JWT 密鑰、API 金鑰、密碼
│   └── db-passwd.sh                # Update database passwords for all roles
│                                   # 更新所有角色的資料庫密碼
├── dev/
│   ├── docker-compose.dev.yml      # Dev overlay (mail server, fresh DB)
│   │                               # 開發疊加設定（郵件伺服器、全新資料庫）
│   └── data.sql                    # Seed data for development
│                                   # 開發用種子資料
└── volumes/
    ├── api/
    │   └── kong.yml                # Kong API gateway routing config
    │                               # Kong API 閘道路由設定
    ├── db/
    │   ├── _supabase.sql           # Create _supabase internal database
    │   │                           # 建立 _supabase 內部資料庫
    │   ├── jwt.sql                 # Set JWT secret in PostgreSQL settings
    │   │                           # 在 PostgreSQL 設定中設定 JWT 密鑰
    │   ├── logs.sql                # Create _analytics schema for Logflare
    │   │                           # 為 Logflare 建立 _analytics schema
    │   ├── pooler.sql              # Create _supavisor schema for pooler
    │   │                           # 為連線池建立 _supavisor schema
    │   ├── realtime.sql            # Create _realtime schema
    │   │                           # 建立 _realtime schema
    │   ├── roles.sql               # Set passwords for all database roles
    │   │                           # 設定所有資料庫角色的密碼
    │   ├── webhooks.sql            # Create webhook functions (pg_net)
    │   │                           # 建立 Webhook 函數（pg_net）
    │   └── init/
    │       └── data.sql            # Optional initial data / seed
    │                               # 可選的初始資料 / 種子資料
    ├── functions/
    │   ├── main/
    │   │   └── index.ts            # Edge Functions entry point (JWT router)
    │   │                           # Edge Functions 進入點（JWT 路由器）
    │   └── hello/
    │       └── index.ts            # Example Edge Function
    │                               # 範例 Edge Function
    ├── logs/
    │   └── vector.yml              # Vector log collection & routing config
    │                               # Vector 日誌收集與路由設定
    └── pooler/
        └── pooler.exs              # Supavisor tenant initialization
                                    # Supavisor 租戶初始化腳本
```

---

## Troubleshooting / 故障排除

### Container won't start / 容器無法啟動

```bash
# Check container status / 檢查容器狀態
docker compose ps

# Check specific service logs / 檢查特定服務日誌
docker compose logs <service-name>

# Common: DB not ready yet - other services depend on it
# 常見問題：資料庫尚未就緒 - 其他服務依賴它
docker compose logs db
```

### Database connection refused / 資料庫連線被拒

```bash
# Verify DB is healthy / 確認資料庫健康
docker compose exec db pg_isready -U postgres -h localhost

# Check if port is already in use / 檢查埠是否已被佔用
ss -tlnp | grep 5432
```

### Dashboard returns 502 / 儀表板回傳 502

The Studio service depends on Analytics (Logflare). Ensure analytics is healthy:

Studio 服務依賴 Analytics (Logflare)。確保 analytics 服務健康：

```bash
docker compose logs analytics
# Wait for "Logflare is ready" message
# 等待 "Logflare is ready" 訊息
```

### Reset everything / 完全重設

```bash
# Interactive reset with confirmations
# 互動式重設（有確認提示）
./reset.sh

# Auto-confirm all steps / 自動確認所有步驟
./reset.sh -y
```

### Regenerate all secrets / 重新產生所有密鑰

```bash
sh utils/generate-keys.sh
# Then restart: / 然後重啟：
docker compose up -d --force-recreate
```

### Update database password / 更新資料庫密碼

```bash
# Updates password for all 12 database roles at once
# 一次更新全部 12 個資料庫角色的密碼
sh utils/db-passwd.sh
```

---

## Backup & Restore / 備份與還原

### Database Backup / 資料庫備份

```bash
# Backup using pg_dump / 使用 pg_dump 備份
docker compose exec db pg_dump -U postgres -d postgres > backup_$(date +%Y%m%d_%H%M%S).sql

# Backup specific schema only / 僅備份特定 schema
docker compose exec db pg_dump -U postgres -d postgres -n public > public_schema_backup.sql
```

### Database Restore / 資料庫還原

```bash
# Restore from backup file / 從備份檔案還原
docker compose exec -T db psql -U postgres -d postgres < backup_20250101_120000.sql
```

### Full Backup (DB + Storage) / 完整備份（資料庫 + 儲存）

```bash
# Stop services first / 先停止服務
docker compose down

# Backup database volume / 備份資料庫卷
tar -czf db_data_backup.tar.gz ./volumes/db/data/

# Backup storage files / 備份儲存檔案
tar -czf storage_backup.tar.gz ./volumes/storage/

# Restart / 重啟
docker compose up -d
```

---

## Podman Notes / Podman 注意事項

**English:**
This setup is fully compatible with Podman in rootless mode. Key differences from Docker:

1. **Socket path:** Set `DOCKER_SOCKET_LOCATION` in `.env` to your Podman socket:
   ```bash
   DOCKER_SOCKET_LOCATION=/run/user/1000/podman/podman.sock
   ```

2. **SELinux labels:** All volume mounts use `:z` or `:Z` flags for SELinux compatibility. `:z` (lowercase) means shared label, `:Z` (uppercase) means private label.

3. **Vector log collection:** Vector reads container logs from the Podman socket. The `security_opt: label=disable` setting on the vector service is required for this to work.

4. **Podman socket activation:** Ensure the Podman socket is running:
   ```bash
   systemctl --user enable --now podman.socket
   ```

5. **Compose command:** Use `podman compose` or install `docker-compose` as a Podman plugin.

**中文：**
此設定完全相容 Podman 無根模式。與 Docker 的主要差異：

1. **Socket 路徑：** 在 `.env` 中設定 `DOCKER_SOCKET_LOCATION` 為你的 Podman socket：
   ```bash
   DOCKER_SOCKET_LOCATION=/run/user/1000/podman/podman.sock
   ```

2. **SELinux 標籤：** 所有卷掛載使用 `:z` 或 `:Z` 旗標以相容 SELinux。`:z`（小寫）表示共享標籤，`:Z`（大寫）表示私有標籤。

3. **Vector 日誌收集：** Vector 從 Podman socket 讀取容器日誌。vector 服務上的 `security_opt: label=disable` 設定是必要的。

4. **Podman socket 啟用：** 確保 Podman socket 正在運行：
   ```bash
   systemctl --user enable --now podman.socket
   ```

5. **Compose 指令：** 使用 `podman compose` 或安裝 `docker-compose` 作為 Podman 插件。

---

## API Endpoints / API 端點

Once running, the following endpoints are available through the Kong gateway:

啟動後，以下端點可通過 Kong 閘道存取：

| Endpoint / 端點 | Service / 服務 | Auth Required / 需要認證 |
|---|---|---|
| `/auth/v1/*` | GoTrue (Authentication) | API Key |
| `/rest/v1/*` | PostgREST (REST API) | API Key |
| `/graphql/v1` | PostgREST (GraphQL) | API Key |
| `/realtime/v1/*` | Realtime (WebSocket) | API Key |
| `/storage/v1/*` | Storage API | No (manages own auth) |
| `/functions/v1/*` | Edge Functions | No (configurable) |
| `/analytics/v1/*` | Logflare Analytics | No |
| `/pg/*` | pg-meta (DB management) | Service Role Key only |
| `/` | Studio Dashboard | Basic Auth (username/password) |

### Example API calls / API 呼叫範例

```bash
# Health check / 健康檢查
curl http://localhost:8100/auth/v1/health

# Query data via REST API / 通過 REST API 查詢資料
curl 'http://localhost:8100/rest/v1/your_table' \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Authorization: Bearer YOUR_ANON_KEY"

# Call an Edge Function / 呼叫 Edge Function
curl 'http://localhost:8100/functions/v1/hello' \
  -H "Authorization: Bearer YOUR_ANON_KEY"
```

---

## License / 授權

This project configuration is provided for reference and deployment use.

Supabase is licensed under the [Apache License 2.0](https://github.com/supabase/supabase/blob/master/LICENSE).

此專案配置提供參考及部署使用。

Supabase 採用 [Apache License 2.0](https://github.com/supabase/supabase/blob/master/LICENSE) 授權。

---

**Maintained by / 維護者：** [WOOWTECH](https://github.com/WOOWTECH)
