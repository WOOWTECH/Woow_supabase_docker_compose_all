# Woow Supabase K3s/Kubernetes Deployment

**Supabase Self-Hosted Deployment with K3s/Kubernetes (Kustomize)**

**使用 K3s/Kubernetes 自架 Supabase（Kustomize 編排）**

---

## Table of Contents / 目錄

- [Overview / 概述](#overview--概述)
- [Architecture / 架構](#architecture--架構)
- [Services Explained / 服務說明](#services-explained--服務說明)
- [Prerequisites / 前置需求](#prerequisites--前置需求)
- [Quick Start / 快速開始](#quick-start--快速開始)
- [Post-Deployment Database Setup / 部署後資料庫設定](#post-deployment-database-setup--部署後資料庫設定)
- [Configuration / 配置說明](#configuration--配置說明)
- [Common Commands / 常用指令](#common-commands--常用指令)
- [File Structure / 檔案結構](#file-structure--檔案結構)
- [Troubleshooting / 故障排除](#troubleshooting--故障排除)
- [Known Issues / 已知問題](#known-issues--已知問題)
- [Backup & Restore / 備份與還原](#backup--restore--備份與還原)
- [K3s Notes / K3s 注意事項](#k3s-notes--k3s-注意事項)
- [API Endpoints / API 端點](#api-endpoints--api-端點)
- [Deployment Verified Status / 部署驗證狀態](#deployment-verified-status--部署驗證狀態)
- [License / 授權](#license--授權)

---

## Overview / 概述

**English:**
This branch contains all Kubernetes manifests needed to self-host [Supabase](https://supabase.com/) — the open-source Firebase alternative — on a K3s/Kubernetes cluster using Kustomize. The setup includes 13 microservices: PostgreSQL database, authentication (GoTrue), RESTful API (PostgREST), real-time subscriptions (Realtime), file storage with image transformations (Storage + imgproxy), edge functions, API gateway (Kong), connection pooling (Supavisor), log analytics (Logflare + Vector), database metadata management (pg-meta), and a web dashboard (Studio).

All 13 services have been **deployed and tested on a live k3s cluster** (4-node: 1 control-plane + 3 workers) and verified running on 2026-03-18.

> **Docker Compose / Podman version:** See the [`main` branch](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all/tree/main) for Podman/Docker Compose deployment.

**中文：**
此分支包含使用 Kustomize 在 K3s/Kubernetes 叢集上自架 [Supabase](https://supabase.com/)（開源 Firebase 替代方案）所需的全部 Kubernetes 部署清單。此設定包含 13 個微服務：PostgreSQL 資料庫、身份認證（GoTrue）、RESTful API（PostgREST）、即時訂閱（Realtime）、檔案儲存及影像轉換（Storage + imgproxy）、邊緣函數（Edge Functions）、API 閘道（Kong）、連線池（Supavisor）、日誌分析（Logflare + Vector）、資料庫中繼資料管理（pg-meta）及 Web 儀表板（Studio）。

全部 13 項服務已在 **k3s 叢集上實際部署測試通過**（4 節點：1 控制平面 + 3 工作節點），於 2026-03-18 驗證全數運行。

> **Docker Compose / Podman 版本：** 請參閱 [`main` 分支](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all/tree/main)。

---

## Architecture / 架構

```
                          ┌─────────────────────────────────────┐
                          │           External Access            │
                          │           外部存取                   │
                          │  HTTP:  http://<node-ip>:30080       │
                          │  HTTPS: https://<node-ip>:30443      │
                          └──────────────┬──────────────────────┘
                                         │
                          ┌──────────────▼──────────────────────┐
                          │      Kong API Gateway (NodePort)     │
                          │      Kong API 閘道（NodePort）       │
                          │      kong:2  |  Ports: 8000 / 8443   │
                          └──┬──────┬──────┬──────┬──────┬──────┘
        ┌───────────┬────────┘      │      │      │      └────────┬───────────┐
        │           │               │      │      │               │           │
 ┌──────▼──┐ ┌──────▼──┐    ┌──────▼──┐ ┌─▼──────┐       ┌──────▼──┐ ┌──────▼──────┐
 │ Studio  │ │  Auth   │    │  REST   │ │Realtime│       │ Storage │ │  Functions  │
 │ 儀表板  │ │ 認證    │    │  API    │ │ 即時   │       │ 儲存    │ │  邊緣函數   │
 │ :3000   │ │ :9999   │    │ :3000   │ │ :4000  │       │ :5000   │ │  :9000      │
 └────┬────┘ └────┬────┘    └───┬─────┘ └───┬────┘       └───┬────┘ └─────────────┘
      │           │             │            │                │
      │    ┌──────▼─────────────▼────────────▼──┐             │
      │    │       PostgreSQL (StatefulSet)      │◄────────────┘
      │    │       資料庫 :5432                   │
      │    │       PVC: db-data (8Gi)            │
      │    └──────▲─────────▲───────────▲───────┘
      │           │         │           │
 ┌────▼────┐ ┌────▼────┐ ┌──▼──────┐ ┌─▼──────────┐
 │  Meta   │ │Analytics│ │ Storage │ │ Supavisor  │
 │ 中繼資料│ │  分析   │ │  儲存   │ │  連線池    │
 │ :8080   │ │  :4000  │ │  :5000  │ │ :5432/6543 │
 └─────────┘ └────▲────┘ └────┬────┘ └────────────┘
                  │           │
           ┌──────▼────┐ ┌───▼──────┐
           │  Vector   │ │ Imgproxy │
           │ 日誌收集  │ │ 圖片處理 │
           │  :9001    │ │  :5001   │
           └───────────┘ └──────────┘
```

**Component Dependency Map / 元件相依關係圖：**

```
PostgreSQL ◄─── Auth (GoTrue, 身份驗證)
     ▲  ◄───── REST (PostgREST, 自動 API)
     │  ◄───── Realtime (即時訂閱)
     │  ◄───── Storage (檔案儲存) ──────► imgproxy (影像轉換)
     │  ◄───── Meta (中繼資料 API)
     │  ◄───── Functions (邊緣函數執行環境)
     │  ◄───── Analytics (Logflare 分析)
     │  ◄───── Supavisor (連線池)
     │
     └──────── Vector (日誌收集器)

Kong ◄───────── Studio (儀表板 UI)
  ▲  ◄───────── All API traffic / 所有 API 流量
  │
External clients / SDKs / 外部用戶端
```

---

## Services Explained / 服務說明

| Service / 服務 | Image / 映像 | Port / 埠 | Description (EN) | 說明 (中文) |
|---|---|---|---|---|
| **PostgreSQL (DB)** | `supabase/postgres:15.8.1.085` | 5432 | PostgreSQL 15 database with Supabase extensions (pgvector, pg_net, pgsodium, etc.). The core data store, deployed as a StatefulSet with PVC. | PostgreSQL 15 資料庫，包含 Supabase 擴充套件（pgvector、pg_net、pgsodium 等）。核心資料儲存，以 StatefulSet 搭配 PVC 部署。 |
| **Kong** | `kong:2` | 8000 (HTTP), 8443 (HTTPS) | API gateway / reverse proxy. Routes all external API requests to internal services. Exposed via NodePort 30080/30443. | API 閘道 / 反向代理。將所有外部 API 請求路由到內部服務。通過 NodePort 30080/30443 對外暴露。 |
| **Auth (GoTrue)** | `supabase/gotrue:v2.185.0` | 9999 | Authentication server. Handles user signup, login, JWT token issuance, email/phone verification, OAuth providers, and MFA. | 認證伺服器。處理使用者註冊、登入、JWT 令牌發行、電子郵件/手機驗證、OAuth 供應商及多因素認證。 |
| **REST (PostgREST)** | `postgrest/postgrest:v12.2.8` | 3000 | Automatically generates a RESTful API from your PostgreSQL schema. Supports filtering, pagination, and embedded resources. | 自動從 PostgreSQL schema 生成 RESTful API。支援篩選、分頁及嵌入式資源。 |
| **Realtime** | `supabase/realtime:v2.72.0` | 4000 | Elixir-based WebSocket server for real-time database change notifications. Enables live subscriptions to INSERT, UPDATE, DELETE events. | 基於 Elixir 的 WebSocket 伺服器，提供即時資料庫變更通知。支援 INSERT、UPDATE、DELETE 事件的即時訂閱。 |
| **Storage** | `supabase/storage-api:v1.22.12` | 5000 | File storage service with S3-compatible API. Manages file uploads, downloads, and access policies via Row Level Security. | 檔案儲存服務，具有 S3 相容 API。通過行級安全策略（RLS）管理檔案上傳、下載及存取權限。 |
| **Imgproxy** | `darthsim/imgproxy:v3.8.0` | 5001 | On-the-fly image transformation. Resizes, crops, and converts images (WebP detection supported). Used by Storage. | 即時圖片轉換。支援調整大小、裁剪及格式轉換（支援 WebP 偵測）。由 Storage 服務呼叫。 |
| **Meta (pg-meta)** | `supabase/postgres-meta:v0.86.1` | 8080 | PostgreSQL metadata API. Provides table/column/role/policy information for the Studio dashboard. | PostgreSQL 中繼資料 API。為 Studio 儀表板提供資料表/欄位/角色/策略資訊。 |
| **Functions (Edge Runtime)** | `supabase/edge-runtime:v1.67.4` | 9000 | Deno-based edge function runtime. Executes serverless TypeScript/JavaScript functions. | 基於 Deno 的邊緣函數執行環境。執行無伺服器 TypeScript/JavaScript 函數。 |
| **Studio** | `supabase/studio:2026.01.27-sha-6aa59ff` | 3000 | Web dashboard for managing your Supabase project. Provides SQL editor, table viewer, auth management, and storage browser. | Web 管理儀表板，提供 SQL 編輯器、資料表瀏覽器、認證管理及儲存瀏覽功能。 |
| **Analytics (Logflare)** | `supabase/logflare:1.12.0` | 4000 | Log ingestion and analytics engine. Collects logs from all services and stores them in PostgreSQL (`_supabase` database). | 日誌收集及分析引擎。從所有服務收集日誌並儲存在 PostgreSQL（`_supabase` 資料庫）中。 |
| **Vector** | `timberio/vector:0.34.0-alpine` | 9001 | Log collection agent. Reads container logs and routes them to the Analytics service. | 日誌收集代理。讀取容器日誌並路由到分析服務。 |
| **Supavisor** | `supabase/supavisor:2.5.1` | 5432, 6543 | Database connection pooler. Manages PostgreSQL connections efficiently using transaction-level (6543) and session-level (5432) pooling. | 資料庫連線池。使用交易級（6543）和會話級（5432）連線池高效管理 PostgreSQL 連線。 |

---

## Prerequisites / 前置需求

**English:**
- **K3s** installed and running (`curl -sfL https://get.k3s.io | sh -`), or any Kubernetes 1.25+ cluster
- **kubectl** configured to access the cluster
- **RAM:** 4 GB minimum (8 GB recommended)
- **Disk:** 20 GB minimum (for PVCs: db-data 8Gi + db-config 1Gi + storage-data 10Gi)
- **openssl** (for generating secrets)
- Default **local-path** storage provisioner (k3s built-in) or equivalent

**中文：**
- 已安裝並運行 **K3s**（`curl -sfL https://get.k3s.io | sh -`），或任何 Kubernetes 1.25+ 叢集
- **kubectl** 已設定可存取叢集
- **記憶體：** 最少 4 GB（建議 8 GB）
- **磁碟空間：** 最少 20 GB（PVC 需求：db-data 8Gi + db-config 1Gi + storage-data 10Gi）
- **openssl**（用於生成密鑰）
- 預設 **local-path** 儲存供應器（k3s 內建）或同等儲存方案

---

## Quick Start / 快速開始

### 1. Clone the k3s branch / 複製 k3s 分支

```bash
git clone -b k3s https://github.com/WOOWTECH/Woow_supabase_docker_compose_all.git
cd Woow_supabase_docker_compose_all
```

### 2. Update secrets (CRITICAL) / 更新密鑰（關鍵步驟）

```bash
# Edit secret.yaml and replace ALL placeholder values
# 編輯 secret.yaml 並替換所有預設值
nano k8s-manifests/supabase/secret.yaml

# Encode your values in base64 / 將值進行 base64 編碼
echo -n 'your-secret-value' | base64

# Generate a strong random password / 產生強隨機密碼
openssl rand -base64 32
```

### 3. Update configuration URLs / 更新配置 URL

```bash
# Edit configmap.yaml and update domain URLs
# 編輯 configmap.yaml 並更新網域 URL
nano k8s-manifests/supabase/configmap.yaml

# Key variables to update / 需要更新的重要變數：
# SITE_URL, API_EXTERNAL_URL, SUPABASE_PUBLIC_URL
```

### 4. Deploy all components / 部署所有元件

```bash
# Deploy with Kustomize / 使用 Kustomize 部署
kubectl apply -k k8s-manifests/supabase/

# Wait for database to be ready (other services depend on it)
# 等待資料庫就緒（其他服務依賴它）
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s
```

### 5. Run post-deployment database setup (CRITICAL) / 執行部署後資料庫設定（關鍵步驟）

> **See the next section for detailed instructions.**
>
> **詳細步驟請見下一節。**

### 6. Verify all pods / 確認所有 Pod 運行中

```bash
kubectl -n supabase get pods
# All 13 pods should show 1/1 Running
# 所有 13 個 Pod 應顯示 1/1 Running
```

### 7. Access the dashboard / 存取儀表板

Open your browser and navigate to:

打開瀏覽器並前往：

```
http://<node-ip>:30080
```

Login with the credentials set in `secret.yaml`:

使用 `secret.yaml` 中設定的帳號密碼登入：

- **Username:** `DASHBOARD_USERNAME` (default: `supabase`)
- **Password:** `DASHBOARD_PASSWORD`

---

## Post-Deployment Database Setup / 部署後資料庫設定

> **Why is this needed? / 為什麼需要這個步驟？**
>
> The `supabase/postgres` image ships with a pre-built PostgreSQL data directory baked into the container image. This means `docker-entrypoint-initdb.d` scripts are **always skipped** (the entrypoint detects an existing database and prints "Skipping initialization"). You must run these setup steps manually after the database pod is running.
>
> `supabase/postgres` 映像檔在容器映像中預先建立了 PostgreSQL 資料目錄。這表示 `docker-entrypoint-initdb.d` 腳本**永遠會被跳過**（入口點偵測到現有資料庫後會顯示「Skipping initialization」）。您必須在資料庫 Pod 運行後手動執行這些設定步驟。

### Step 1: Create required roles / 步驟一：建立必要角色

```bash
kubectl -n supabase exec -it sts/db -- psql -U supabase_admin postgres
```

```sql
-- Create the 'postgres' role (required by GoTrue and Storage migrations)
-- 建立 'postgres' 角色（GoTrue 和 Storage 的 migration SQL 需要）
CREATE ROLE postgres WITH LOGIN SUPERUSER CREATEDB CREATEROLE
  REPLICATION BYPASSRLS PASSWORD 'YOUR_POSTGRES_PASSWORD';

-- Set passwords for all service roles (replace YOUR_POSTGRES_PASSWORD)
-- 設定所有服務角色的密碼（替換 YOUR_POSTGRES_PASSWORD）
ALTER ROLE supabase_auth_admin WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
ALTER ROLE supabase_storage_admin WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
ALTER ROLE supabase_functions_admin WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
ALTER ROLE supabase_realtime_admin WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
ALTER ROLE authenticator WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
ALTER ROLE pgbouncer WITH LOGIN PASSWORD 'YOUR_POSTGRES_PASSWORD';
```

### Step 2: Grant schema permissions / 步驟二：授予 Schema 權限

```sql
-- PostgreSQL 15 changed public schema ownership - grant access to service roles
-- PostgreSQL 15 更改了 public schema 擁有權 - 授予服務角色存取權
GRANT ALL ON SCHEMA public TO supabase_auth_admin;
GRANT ALL ON SCHEMA public TO supabase_storage_admin;
GRANT ALL ON SCHEMA public TO supabase_functions_admin;
GRANT CREATE ON DATABASE postgres TO supabase_auth_admin;
GRANT CREATE ON DATABASE postgres TO supabase_storage_admin;

-- Default privileges for dashboard users / 儀表板使用者預設權限
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON TABLES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON SEQUENCES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON ROUTINES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON TABLES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON SEQUENCES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON ROUTINES TO postgres, dashboard_user;
```

### Step 3: Create `_supabase` database / 步驟三：建立 `_supabase` 資料庫

```sql
-- Analytics (Logflare) and Supavisor need the _supabase database
-- Analytics (Logflare) 和 Supavisor 需要 _supabase 資料庫
CREATE DATABASE _supabase OWNER supabase_admin;
\c _supabase
CREATE SCHEMA IF NOT EXISTS _analytics;
\c postgres
```

### Step 4: Restart and fix GoTrue enums / 步驟四：重啟並修正 GoTrue 列舉類型

```bash
# Restart all services to pick up role/permission changes
# 重啟所有服務以套用角色/權限變更
kubectl -n supabase rollout restart deploy --all

# Wait ~30 seconds for GoTrue to run initial migrations
# 等待約 30 秒讓 GoTrue 執行初始 migration
sleep 30
```

```bash
# Connect to DB and move enums from public to auth schema
# 連線到資料庫，將 enum 從 public 移至 auth schema
kubectl -n supabase exec -it sts/db -- psql -U supabase_admin postgres
```

```sql
-- GoTrue creates enum types in 'public' schema, but later migrations
-- reference them as 'auth.factor_type' - move them to auth schema
-- GoTrue 在 'public' schema 建立 enum 類型，但後續 migration
-- 以 'auth.factor_type' 引用 - 將它們移至 auth schema
ALTER TYPE public.factor_type SET SCHEMA auth;
ALTER TYPE public.factor_status SET SCHEMA auth;
ALTER TYPE public.aal_level SET SCHEMA auth;
ALTER TYPE public.code_challenge_method SET SCHEMA auth;
ALTER TYPE public.one_time_token_type SET SCHEMA auth;
```

### Step 5: Final restart / 步驟五：最終重啟

```bash
# Restart auth to complete remaining migrations
# 重啟 auth 以完成剩餘的 migration
kubectl -n supabase rollout restart deploy/auth

# Verify all pods are running / 確認所有 Pod 運行中
kubectl -n supabase get pods
```

### One-Shot Script / 一鍵腳本

For convenience, here is the complete post-deploy setup as a single script:

為方便使用，以下是完整的部署後設定腳本：

```bash
#!/bin/bash
# Run after: kubectl apply -k k8s-manifests/supabase/
# Wait for DB first: kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

NS="supabase"
PG_PASS="YOUR_POSTGRES_PASSWORD"  # Must match secret.yaml / 必須與 secret.yaml 一致

# Steps 1-3: Roles, grants, _supabase database
kubectl -n $NS exec -i sts/db -- psql -U supabase_admin -d postgres <<EOSQL
DO \$\$ BEGIN
  IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='postgres') THEN
    EXECUTE format('CREATE ROLE postgres WITH LOGIN SUPERUSER CREATEDB CREATEROLE REPLICATION BYPASSRLS PASSWORD %L', '$PG_PASS');
  END IF;
END \$\$;

ALTER ROLE supabase_auth_admin WITH LOGIN PASSWORD '$PG_PASS';
ALTER ROLE supabase_storage_admin WITH LOGIN PASSWORD '$PG_PASS';
ALTER ROLE supabase_functions_admin WITH LOGIN PASSWORD '$PG_PASS';
ALTER ROLE supabase_realtime_admin WITH LOGIN PASSWORD '$PG_PASS';
ALTER ROLE authenticator WITH LOGIN PASSWORD '$PG_PASS';
ALTER ROLE pgbouncer WITH LOGIN PASSWORD '$PG_PASS';

GRANT ALL ON SCHEMA public TO supabase_auth_admin;
GRANT ALL ON SCHEMA public TO supabase_storage_admin;
GRANT ALL ON SCHEMA public TO supabase_functions_admin;
GRANT CREATE ON DATABASE postgres TO supabase_auth_admin;
GRANT CREATE ON DATABASE postgres TO supabase_storage_admin;

ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON TABLES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON SEQUENCES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_auth_admin IN SCHEMA auth
  GRANT ALL ON ROUTINES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON TABLES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON SEQUENCES TO postgres, dashboard_user;
ALTER DEFAULT PRIVILEGES FOR ROLE supabase_storage_admin IN SCHEMA storage
  GRANT ALL ON ROUTINES TO postgres, dashboard_user;
EOSQL

# Create _supabase database
kubectl -n $NS exec -i sts/db -- psql -U supabase_admin -d postgres -c \
  "SELECT 1 FROM pg_database WHERE datname='_supabase'" | grep -q 1 || \
  kubectl -n $NS exec -i sts/db -- psql -U supabase_admin -d postgres -c \
  "CREATE DATABASE _supabase OWNER supabase_admin;"

kubectl -n $NS exec -i sts/db -- psql -U supabase_admin -d _supabase -c \
  "CREATE SCHEMA IF NOT EXISTS _analytics;"

echo "=== Steps 1-3 done. Restarting... ==="
kubectl -n $NS rollout restart deploy --all
echo "=== Waiting 30s for GoTrue initial migrations... ==="
sleep 30

# Step 4: Move enums from public to auth schema
kubectl -n $NS exec -i sts/db -- psql -U supabase_admin -d postgres <<EOSQL
DO \$\$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid
             WHERE t.typname='factor_type' AND n.nspname='public') THEN
    ALTER TYPE public.factor_type SET SCHEMA auth;
  END IF;
  IF EXISTS (SELECT 1 FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid
             WHERE t.typname='factor_status' AND n.nspname='public') THEN
    ALTER TYPE public.factor_status SET SCHEMA auth;
  END IF;
  IF EXISTS (SELECT 1 FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid
             WHERE t.typname='aal_level' AND n.nspname='public') THEN
    ALTER TYPE public.aal_level SET SCHEMA auth;
  END IF;
  IF EXISTS (SELECT 1 FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid
             WHERE t.typname='code_challenge_method' AND n.nspname='public') THEN
    ALTER TYPE public.code_challenge_method SET SCHEMA auth;
  END IF;
  IF EXISTS (SELECT 1 FROM pg_type t JOIN pg_namespace n ON t.typnamespace=n.oid
             WHERE t.typname='one_time_token_type' AND n.nspname='public') THEN
    ALTER TYPE public.one_time_token_type SET SCHEMA auth;
  END IF;
END \$\$;
EOSQL

echo "=== Step 4 done. Restarting auth... ==="
kubectl -n $NS rollout restart deploy/auth
echo "=== Post-deploy setup complete! Check: kubectl -n $NS get pods ==="
```

---

## Configuration / 配置說明

### Secrets (MUST change before deploying) / 密鑰（部署前必須更改）

Edit `k8s-manifests/supabase/secret.yaml` and replace ALL placeholder values. Values are base64-encoded.

編輯 `k8s-manifests/supabase/secret.yaml` 並替換所有預設值。值為 base64 編碼。

```bash
# Encode a value / 編碼一個值
echo -n 'your-secret-value' | base64

# Decode a value / 解碼一個值
echo 'base64string' | base64 -d

# Generate JWT keys (recommended) / 產生 JWT 金鑰（建議）
supabase gen keys --experimental

# Generate a strong random password / 產生強隨機密碼
openssl rand -base64 32
```

| Variable / 變數 | Description (EN) | 說明 (中文) | Notes / 備註 |
|---|---|---|---|
| `POSTGRES_PASSWORD` | PostgreSQL superuser password | PostgreSQL 超級使用者密碼 | Use a strong random password / 使用強隨機密碼 |
| `JWT_SECRET` | Secret for signing JWT tokens | JWT 令牌簽名密鑰 | Minimum 32 characters / 至少 32 字元 |
| `ANON_KEY` | Public API key for anonymous access | 匿名存取的公開 API 金鑰 | Generate with supabase CLI / 使用 supabase CLI 產生 |
| `SERVICE_ROLE_KEY` | Admin API key that bypasses RLS | 繞過 RLS 的管理員 API 金鑰 | Server-side only / 僅限伺服器端 |
| `DASHBOARD_USERNAME` | Studio login username | Studio 登入使用者名稱 | Default: `supabase` / 預設：`supabase` |
| `DASHBOARD_PASSWORD` | Studio login password | Studio 登入密碼 | Must change / 必須更改 |
| `LOGFLARE_PUBLIC_ACCESS_TOKEN` | Logflare public API token | Logflare 公開 API 權杖 | For log ingestion / 用於日誌擷取 |
| `LOGFLARE_PRIVATE_ACCESS_TOKEN` | Logflare private API token | Logflare 私有 API 權杖 | For admin operations / 用於管理操作 |
| `SECRET_KEY_BASE` | Erlang/Phoenix secret key | Erlang/Phoenix 密鑰 | **Minimum 64 characters** / **至少 64 字元** |
| `VAULT_ENC_KEY` | Database vault encryption key | 資料庫 Vault 加密金鑰 | For encrypted columns / 用於加密欄位 |
| `PG_META_CRYPTO_KEY` | Postgres Meta encryption key | Postgres Meta 加密金鑰 | For metadata encryption / 用於中繼資料加密 |

### ConfigMap Environment Variables / ConfigMap 環境變數

Key variables to adjust in `k8s-manifests/supabase/configmap.yaml`:

在 `k8s-manifests/supabase/configmap.yaml` 中需調整的關鍵變數：

| Variable / 變數 | Description (EN) | 說明 (中文) | Default / 預設值 |
|---|---|---|---|
| `SITE_URL` | Your application URL | 你的應用程式 URL | `http://localhost` |
| `API_EXTERNAL_URL` | External URL for API access | API 外部存取 URL | `http://localhost:8000` |
| `SUPABASE_PUBLIC_URL` | Public URL for Studio dashboard | Studio 儀表板公開 URL | `http://localhost:8000` |
| `DISABLE_SIGNUP` | Disable new user registration | 停用新使用者註冊 | `false` |
| `ENABLE_EMAIL_AUTOCONFIRM` | Auto-confirm email signups | 自動確認電子郵件註冊 | `true` |
| `GOTRUE_SMTP_HOST` | SMTP server for email delivery | 電子郵件傳送 SMTP 伺服器 | *(empty - configure for production)* |
| `GOTRUE_SMTP_PORT` | SMTP server port | SMTP 伺服器連接埠 | `587` |
| `GOTRUE_SMTP_ADMIN_EMAIL` | From address for emails | 電子郵件寄件地址 | `admin@example.com` |
| `FILE_SIZE_LIMIT` | Maximum upload file size (bytes) | 最大上傳檔案大小（位元組） | `52428800` (50MB) |
| `POOLER_DEFAULT_POOL_SIZE` | Supavisor pool size per tenant | Supavisor 每租戶連線池大小 | `20` |
| `POOLER_MAX_CLIENT_CONN` | Maximum client connections | 最大用戶端連線數 | `200` |

### SMTP Configuration / SMTP 郵件設定

For email verification and password reset to work, configure a real SMTP server in `configmap.yaml`:

若要使電子郵件驗證和密碼重設正常運作，請在 `configmap.yaml` 中設定真實的 SMTP 伺服器：

```yaml
GOTRUE_SMTP_HOST: "smtp.your-provider.com"
GOTRUE_SMTP_PORT: "587"
GOTRUE_SMTP_ADMIN_EMAIL: "noreply@your-domain.com"
GOTRUE_SMTP_USER: "your-smtp-username"
GOTRUE_SMTP_PASS: "your-smtp-password"
GOTRUE_SMTP_SENDER_NAME: "YourApp"
```

---

## Common Commands / 常用指令

| Command / 指令 | Description (EN) | 說明 (中文) |
|---|---|---|
| `kubectl apply -k k8s-manifests/supabase/` | Deploy all Supabase components | 部署所有 Supabase 元件 |
| `kubectl -n supabase get pods` | List all running pods | 列出所有運行中的 Pod |
| `kubectl -n supabase get pods -w` | Watch pod status in real-time | 即時觀察 Pod 狀態 |
| `kubectl -n supabase logs deploy/<service> -f` | Follow logs for a specific service | 跟蹤特定服務日誌 |
| `kubectl -n supabase logs sts/db -f` | Follow database logs | 跟蹤資料庫日誌 |
| `kubectl -n supabase rollout restart deploy --all` | Restart all deployments | 重啟所有 Deployment |
| `kubectl -n supabase rollout restart deploy/<service>` | Restart a specific service | 重啟特定服務 |
| `kubectl -n supabase exec -it sts/db -- psql -U supabase_admin postgres` | Connect to PostgreSQL | 連線到 PostgreSQL |
| `kubectl -n supabase get pvc` | Check PVC status | 檢查 PVC 狀態 |
| `kubectl -n supabase get secret supabase-secret -o jsonpath='{.data.JWT_SECRET}' \| base64 -d` | Verify JWT secret | 驗證 JWT 密鑰 |
| `kubectl delete -k k8s-manifests/supabase/` | Delete all Supabase resources | 刪除所有 Supabase 資源 |
| `kubectl -n supabase scale deploy/<service> --replicas=2` | Scale a service | 擴展服務副本數 |
| `kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s` | Wait for DB readiness | 等待資料庫就緒 |

---

## File Structure / 檔案結構

```
.
├── README.md                                  # This documentation (K3s deployment)
│                                              # 本文件（K3s 部署）
└── k8s-manifests/
    ├── README.md                              # General K3s overview (all WOOWTECH services)
    │                                          # 通用 K3s 概述（所有 WOOWTECH 服務）
    └── supabase/
        ├── README.md                          # Detailed Supabase-specific guide
        │                                      # 詳細 Supabase 專屬指南
        ├── kustomization.yaml                 # Kustomize orchestration (all resources)
        │                                      # Kustomize 編排檔（所有資源）
        ├── namespace.yaml                     # Namespace: supabase
        │                                      # 命名空間：supabase
        ├── secret.yaml                        # All sensitive credentials (base64)
        │                                      # 所有敏感憑證（base64 編碼）
        ├── configmap.yaml                     # Shared non-sensitive configuration
        │                                      # 共用非敏感設定
        ├── pvc.yaml                           # PVCs: db-data (8Gi), db-config (1Gi), storage-data (10Gi)
        │                                      # PVC：db-data (8Gi)、db-config (1Gi)、storage-data (10Gi)
        ├── db-statefulset.yaml                # PostgreSQL StatefulSet (with Supabase extensions)
        │                                      # PostgreSQL StatefulSet（含 Supabase 擴充套件）
        ├── db-service.yaml                    # PostgreSQL ClusterIP service
        │                                      # PostgreSQL ClusterIP 服務
        ├── kong-deployment.yaml               # Kong API gateway Deployment
        │                                      # Kong API 閘道 Deployment
        ├── kong-service.yaml                  # Kong NodePort service (30080/30443)
        │                                      # Kong NodePort 服務（30080/30443）
        ├── auth-deployment.yaml               # GoTrue authentication Deployment
        │                                      # GoTrue 身份認證 Deployment
        ├── auth-service.yaml                  # GoTrue ClusterIP service
        │                                      # GoTrue ClusterIP 服務
        ├── rest-deployment.yaml               # PostgREST API Deployment
        │                                      # PostgREST API Deployment
        ├── rest-service.yaml                  # PostgREST ClusterIP service
        │                                      # PostgREST ClusterIP 服務
        ├── realtime-deployment.yaml           # Realtime WebSocket Deployment
        │                                      # Realtime WebSocket Deployment
        ├── realtime-service.yaml              # Realtime ClusterIP service
        │                                      # Realtime ClusterIP 服務
        ├── storage-deployment.yaml            # Storage API Deployment
        │                                      # Storage API Deployment
        ├── storage-service.yaml               # Storage ClusterIP service
        │                                      # Storage ClusterIP 服務
        ├── imgproxy-deployment.yaml           # imgproxy image transformer Deployment
        │                                      # imgproxy 圖片轉換 Deployment
        ├── imgproxy-service.yaml              # imgproxy ClusterIP service
        │                                      # imgproxy ClusterIP 服務
        ├── meta-deployment.yaml               # Postgres Meta API Deployment
        │                                      # Postgres Meta API Deployment
        ├── meta-service.yaml                  # Postgres Meta ClusterIP service
        │                                      # Postgres Meta ClusterIP 服務
        ├── functions-deployment.yaml          # Edge Functions runtime Deployment
        │                                      # 邊緣函數執行環境 Deployment
        ├── functions-service.yaml             # Edge Functions ClusterIP service
        │                                      # 邊緣函數 ClusterIP 服務
        ├── studio-deployment.yaml             # Studio dashboard Deployment
        │                                      # Studio 儀表板 Deployment
        ├── studio-service.yaml                # Studio ClusterIP service
        │                                      # Studio ClusterIP 服務
        ├── supavisor-deployment.yaml          # Connection pooler Deployment
        │                                      # 連線池 Deployment
        ├── supavisor-service.yaml             # Connection pooler ClusterIP service
        │                                      # 連線池 ClusterIP 服務
        ├── analytics-deployment.yaml          # Logflare analytics Deployment
        │                                      # Logflare 分析 Deployment
        ├── analytics-service.yaml             # Logflare analytics ClusterIP service
        │                                      # Logflare 分析 ClusterIP 服務
        ├── vector-deployment.yaml             # Vector log pipeline Deployment
        │                                      # Vector 日誌管線 Deployment
        └── vector-service.yaml                # Vector ClusterIP service
                                               # Vector ClusterIP 服務
```

---

## Troubleshooting / 故障排除

### Container won't start / 容器無法啟動

```bash
# Check pod status and events / 檢查 Pod 狀態和事件
kubectl -n supabase get pods
kubectl -n supabase describe pod <pod-name>

# Check specific service logs / 檢查特定服務日誌
kubectl -n supabase logs deploy/<service-name>

# Most common: DB not ready yet - other services depend on it
# 最常見：資料庫尚未就緒 - 其他服務依賴它
kubectl -n supabase logs sts/db
```

### Database connection refused / 資料庫連線被拒

```bash
# Verify DB is healthy / 確認資料庫健康
kubectl -n supabase exec sts/db -- pg_isready -U supabase_admin -h localhost

# Check PVC is bound / 檢查 PVC 是否已綁定
kubectl -n supabase get pvc db-data
```

### Auth: `type "auth.factor_type" does not exist (SQLSTATE 42704)`

**Root Cause / 根本原因：** GoTrue's early migrations create enum types (`factor_type`, `factor_status`, `aal_level`, `code_challenge_method`, `one_time_token_type`) in the `public` schema without schema qualification. Later migrations reference them as `auth.factor_type`, which fails.

GoTrue 的早期 migration 在 `public` schema 建立 enum 類型，沒有指定 schema。後續 migration 以 `auth.factor_type` 引用而失敗。

**Fix / 修復：**
```sql
ALTER TYPE public.factor_type SET SCHEMA auth;
ALTER TYPE public.factor_status SET SCHEMA auth;
ALTER TYPE public.aal_level SET SCHEMA auth;
ALTER TYPE public.code_challenge_method SET SCHEMA auth;
ALTER TYPE public.one_time_token_type SET SCHEMA auth;
```

Then restart: `kubectl -n supabase rollout restart deploy/auth`

### Auth: `role "postgres" does not exist`

**Root Cause / 根本原因：** GoTrue and Storage migration SQL hardcodes `GRANT ... TO postgres`. The supabase image uses `supabase_admin` as superuser.

GoTrue 和 Storage 的 migration SQL 硬編碼了 `GRANT ... TO postgres`，但映像檔使用 `supabase_admin`。

**Fix / 修復：**
```sql
CREATE ROLE postgres WITH LOGIN SUPERUSER CREATEDB CREATEROLE
  REPLICATION BYPASSRLS PASSWORD 'YOUR_POSTGRES_PASSWORD';
```

### Auth: `permission denied for schema public (SQLSTATE 42501)`

**Root Cause / 根本原因：** PostgreSQL 15 changed `public` schema ownership to `pg_database_owner`. Service roles lack CREATE privilege by default.

PostgreSQL 15 將 `public` schema 擁有權改為 `pg_database_owner`。服務角色預設沒有 CREATE 權限。

**Fix / 修復：**
```sql
GRANT ALL ON SCHEMA public TO supabase_auth_admin;
GRANT CREATE ON DATABASE postgres TO supabase_auth_admin;
GRANT CREATE ON DATABASE postgres TO supabase_storage_admin;
```

### Analytics/Supavisor: database `_supabase` does not exist

**Root Cause / 根本原因：** The supabase/postgres image does not create the `_supabase` database that Analytics and Supavisor need.

supabase/postgres 映像檔不會建立 Analytics 和 Supavisor 需要的 `_supabase` 資料庫。

**Fix / 修復：**
```sql
CREATE DATABASE _supabase OWNER supabase_admin;
\c _supabase
CREATE SCHEMA IF NOT EXISTS _analytics;
```

### Realtime: health check returning 404

**Root Cause / 根本原因：** `supabase/realtime` v2.x (Elixir-based) does not expose an HTTP `/api/health` endpoint.

`supabase/realtime` v2.x（基於 Elixir）不提供 HTTP `/api/health` 端點。

**Fix / 修復：** Use TCP socket probe (already applied in manifests):
```yaml
livenessProbe:
  tcpSocket:
    port: 4000
readinessProbe:
  tcpSocket:
    port: 4000
```

### Realtime: `cookie store expects conn.secret_key_base to be at least 64 bytes`

**Root Cause / 根本原因：** The `SECRET_KEY_BASE` value in `secret.yaml` is shorter than 64 bytes.

`secret.yaml` 中的 `SECRET_KEY_BASE` 值短於 64 位元組。

**Fix / 修復：** Ensure `SECRET_KEY_BASE` is at least 64 characters:
```bash
openssl rand -base64 48
echo -n 'your-64-char-or-longer-secret' | base64
```

### Studio: readiness probe failing on `/api/profile`

**Root Cause / 根本原因：** The `/api/profile` endpoint requires authentication.

`/api/profile` 端點需要認證。

**Fix / 修復：** Use TCP socket probe (already applied in manifests):
```yaml
readinessProbe:
  tcpSocket:
    port: 3000
```

### Dashboard returns 502 / 儀表板回傳 502

The Studio service depends on Analytics (Logflare). Ensure both Kong and upstream services are running:

Studio 服務依賴 Analytics (Logflare)。確保 Kong 和上游服務都在運行：

```bash
kubectl -n supabase get pods
kubectl -n supabase logs deploy/kong
```

### Reset everything / 完全重設

```bash
# Delete all resources / 刪除所有資源
kubectl delete -k k8s-manifests/supabase/

# Delete PVCs for fresh start (WARNING: deletes all data!)
# 刪除 PVC 以重新開始（警告：會刪除所有資料！）
kubectl -n supabase delete pvc --all

# Redeploy / 重新部署
kubectl apply -k k8s-manifests/supabase/
```

---

## Known Issues / 已知問題

| Issue / 問題 | Cause / 原因 | Status / 狀態 |
|---|---|---|
| `docker-entrypoint-initdb.d` scripts never run / 初始化腳本不會執行 | `supabase/postgres` image has pre-built data directory / 映像檔預先建立了資料目錄 | By design - use manual post-deploy setup / 設計如此 - 使用手動部署後設定 |
| GoTrue creates enums in `public` schema / GoTrue 在 public schema 建立 enum | GoTrue SQL lacks schema qualifier / GoTrue SQL 沒有指定 schema | Workaround: `ALTER TYPE ... SET SCHEMA auth` / 解決方案：移動 enum |
| GoTrue/Storage require `postgres` role / 需要 postgres 角色 | Migration SQL hardcodes `GRANT ... TO postgres` / Migration 硬編碼 postgres | Workaround: create `postgres` role manually / 解決方案：手動建立角色 |
| PostgreSQL 15 `public` schema permissions / PG15 公共 schema 權限 | PG15 changed ownership to `pg_database_owner` / PG15 更改了擁有權 | Workaround: explicit GRANT statements / 解決方案：明確 GRANT 語句 |
| Realtime no HTTP health endpoint / Realtime 無 HTTP 健康檢查端點 | Elixir app uses tenant-specific checks / Elixir 應用使用租戶特定檢查 | Fixed in manifests: TCP socket probe / 已在清單中修復 |
| Studio `/api/profile` requires auth / Studio 端點需要認證 | Next.js API route needs session / Next.js 路由需要 session | Fixed in manifests: TCP socket probe / 已在清單中修復 |
| `SECRET_KEY_BASE` minimum 64 bytes / 密鑰最少 64 位元組 | Elixir/Phoenix cookie store requirement / Elixir/Phoenix 要求 | Ensure key length in secret.yaml / 確保密鑰長度 |

---

## Backup & Restore / 備份與還原

### Database Backup / 資料庫備份

```bash
# Full database dump / 完整資料庫轉儲
kubectl -n supabase exec sts/db -- pg_dumpall -U supabase_admin > supabase-full-backup.sql

# Application database only / 僅應用程式資料庫
kubectl -n supabase exec sts/db -- pg_dump -U supabase_admin postgres > supabase-db-backup.sql

# Backup specific schema / 僅備份特定 schema
kubectl -n supabase exec sts/db -- pg_dump -U supabase_admin -d postgres -n public > public_schema_backup.sql
```

### Storage Backup / 儲存備份

```bash
# Backup uploaded files / 備份上傳的檔案
kubectl -n supabase exec deploy/storage -- tar czf /tmp/storage-backup.tar.gz /var/lib/storage
kubectl -n supabase cp supabase/<storage-pod>:/tmp/storage-backup.tar.gz ./storage-backup.tar.gz

# Backup secrets (store securely!) / 備份密鑰（請安全保存！）
kubectl -n supabase get secret supabase-secret -o yaml > supabase-secret-backup.yaml
```

### Database Restore / 資料庫還原

```bash
# Restore from backup / 從備份還原
kubectl -n supabase exec -i sts/db -- psql -U supabase_admin postgres < supabase-db-backup.sql

# Restore storage files / 還原儲存檔案
kubectl -n supabase cp ./storage-backup.tar.gz supabase/<storage-pod>:/tmp/storage-backup.tar.gz
kubectl -n supabase exec deploy/storage -- tar xzf /tmp/storage-backup.tar.gz -C /

# Restart services / 重啟服務
kubectl -n supabase rollout restart deploy --all
```

---

## K3s Notes / K3s 注意事項

**English:**
This setup is designed for and tested on K3s. Key considerations:

1. **Storage:** All PVCs use `local-path` storage class (k3s default). Data is stored on the node's local disk. For production, consider:
   - **Longhorn** - k3s distributed block storage
   - **NFS** - Network file system for shared storage
   - **Rook-Ceph** - Enterprise-grade distributed storage

2. **Service exposure:** Kong is exposed via NodePort (30080/30443). For production, consider:
   - Traefik IngressRoute (k3s built-in Traefik)
   - Nginx Ingress Controller
   - Cloudflare Tunnel for secure external access

3. **Resource limits:** All deployments have `resources.requests` and `resources.limits` configured. Adjust based on your cluster capacity.

4. **Namespace isolation:** All Supabase services run in the `supabase` namespace for clean isolation.

5. **Service discovery:** Services communicate via Kubernetes DNS (`<service>.supabase.svc.cluster.local`).

6. **Differences from Docker Compose:**
   - Secrets are managed via Kubernetes Secrets (base64-encoded) instead of `.env` files
   - Configuration via ConfigMap instead of environment files
   - Storage via PersistentVolumeClaims instead of Docker volumes
   - Health checks via liveness/readiness probes instead of Docker healthchecks
   - Rolling updates via `kubectl rollout restart` instead of `docker compose up -d`

**中文：**
此設定專為 K3s 設計並已在其上測試通過。主要注意事項：

1. **儲存：** 所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。資料儲存在節點的本機磁碟上。正式環境建議考慮：
   - **Longhorn** - k3s 分散式區塊儲存
   - **NFS** - 網路檔案系統（共享儲存）
   - **Rook-Ceph** - 企業級分散式儲存

2. **服務暴露：** Kong 通過 NodePort（30080/30443）對外暴露。正式環境建議：
   - Traefik IngressRoute（k3s 內建 Traefik）
   - Nginx Ingress Controller
   - Cloudflare Tunnel 安全外部存取

3. **資源限制：** 所有 Deployment 已設定 `resources.requests` 和 `resources.limits`。請根據叢集容量調整。

4. **命名空間隔離：** 所有 Supabase 服務在 `supabase` 命名空間中運行，實現乾淨隔離。

5. **服務發現：** 服務透過 Kubernetes DNS（`<service>.supabase.svc.cluster.local`）通信。

6. **與 Docker Compose 的差異：**
   - 密鑰透過 Kubernetes Secrets（base64 編碼）管理，而非 `.env` 檔案
   - 設定透過 ConfigMap，而非環境檔案
   - 儲存透過 PersistentVolumeClaim，而非 Docker volumes
   - 健康檢查透過 liveness/readiness 探針，而非 Docker healthcheck
   - 滾動更新透過 `kubectl rollout restart`，而非 `docker compose up -d`

---

## API Endpoints / API 端點

Once running, the following endpoints are available through the Kong gateway:

啟動後，以下端點可通過 Kong 閘道存取：

| Endpoint / 端點 | Service / 服務 | Auth Required / 需要認證 |
|---|---|---|
| `/auth/v1/*` | GoTrue (Authentication / 身份認證) | API Key |
| `/rest/v1/*` | PostgREST (REST API) | API Key |
| `/graphql/v1` | PostgREST (GraphQL) | API Key |
| `/realtime/v1/*` | Realtime (WebSocket / 即時) | API Key |
| `/storage/v1/*` | Storage API (儲存) | No (manages own auth / 自行管理認證) |
| `/functions/v1/*` | Edge Functions (邊緣函數) | No (configurable / 可設定) |
| `/analytics/v1/*` | Logflare Analytics (分析) | No |
| `/pg/*` | pg-meta (DB management / 資料庫管理) | Service Role Key only |
| `/` | Studio Dashboard (儀表板) | Basic Auth (username/password) |

### Example API calls / API 呼叫範例

```bash
# Health check / 健康檢查
curl http://<node-ip>:30080/auth/v1/health

# Query data via REST API / 通過 REST API 查詢資料
curl 'http://<node-ip>:30080/rest/v1/your_table' \
  -H "apikey: YOUR_ANON_KEY" \
  -H "Authorization: Bearer YOUR_ANON_KEY"

# Call an Edge Function / 呼叫 Edge Function
curl 'http://<node-ip>:30080/functions/v1/hello' \
  -H "Authorization: Bearer YOUR_ANON_KEY"
```

### Client SDK Configuration / 用戶端 SDK 設定

```javascript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'http://<node-ip>:30080',  // API_EXTERNAL_URL
  '<your-anon-key>'           // ANON_KEY from secret.yaml
)
```

---

## Deployment Verified Status / 部署驗證狀態

All 13 services tested and confirmed running on k3s (2026-03-18):

全部 13 個服務已在 k3s 上測試並確認運行（2026-03-18）：

| Service / 服務 | Image / 映像 | Status / 狀態 | Health Check / 健康檢查 |
|---|---|---|---|
| PostgreSQL (db) | `supabase/postgres:15.8.1.085` | Running | TCP :5432 |
| Auth (GoTrue) | `supabase/gotrue:v2.185.0` | Running | HTTP `/health` |
| REST (PostgREST) | `postgrest/postgrest:v12.2.8` | Running | HTTP `/` |
| Realtime | `supabase/realtime:v2.72.0` | Running | TCP :4000 |
| Storage | `supabase/storage-api:v1.22.12` | Running | HTTP `/status` |
| Studio | `supabase/studio:2026.01.27-sha-6aa59ff` | Running | TCP :3000 |
| Kong | `kong:2` | Running | HTTP routes |
| Meta | `supabase/postgres-meta:v0.86.1` | Running | HTTP `/health` |
| Functions | `supabase/edge-runtime:v1.67.4` | Running | HTTP `/` |
| Analytics | `supabase/logflare:1.12.0` | Running | HTTP `/` |
| Supavisor | `supabase/supavisor:2.5.1` | Running | TCP :5432/:6543 |
| imgproxy | `darthsim/imgproxy:v3.8.0` | Running | HTTP `/` |
| Vector | `timberio/vector:0.34.0-alpine` | Running | TCP :9001 |

### Production Checklist / 正式環境檢查清單

- [ ] Replace ALL placeholder values in `secret.yaml` / 替換 secret.yaml 中所有預設值
- [ ] Generate proper JWT keys (`ANON_KEY`, `SERVICE_ROLE_KEY`) / 產生正確的 JWT 金鑰
- [ ] Update `SITE_URL`, `API_EXTERNAL_URL`, `SUPABASE_PUBLIC_URL` / 更新 URL 設定
- [ ] Configure SMTP settings for email delivery / 設定 SMTP 郵件傳送
- [ ] Set `ENABLE_EMAIL_AUTOCONFIRM: "false"` for production / 正式環境停用自動確認
- [ ] Increase PVC sizes for `db-data` and `storage-data` / 增加 PVC 大小
- [ ] Switch to network-attached storage (Longhorn, Ceph, NFS) / 切換到網路儲存
- [ ] Place behind TLS-terminating reverse proxy / 放置在 TLS 反向代理後方
- [ ] Set proper `FILE_SIZE_LIMIT` / 設定適當的檔案大小限制
- [ ] Review connection pooler settings (`POOLER_*`) / 檢視連線池設定
- [ ] Set up automated database backups / 設定自動化備份
- [ ] Monitor resource usage and adjust limits / 監控資源並調整限制

### Deployment Methods Comparison / 部署方式比較

| Feature / 功能 | Podman/Docker Compose | K3s/Kubernetes |
|---|---|---|
| Branch / 分支 | `main` | `k3s` |
| Orchestrator / 編排工具 | Podman / Docker | K3s / Kubernetes |
| Config format / 設定格式 | `.env` + `docker-compose.yml` | ConfigMap + Secret + YAML manifests |
| Scaling / 擴展 | Manual / 手動 | `kubectl scale` |
| Health checks / 健康檢查 | Docker healthcheck | liveness/readiness/startup probes |
| Service discovery / 服務發現 | Docker DNS | Kubernetes DNS (`svc.cluster.local`) |
| Storage / 儲存 | Docker volumes | PersistentVolumeClaims |
| Rolling updates / 滾動更新 | `docker compose pull && up -d` | `kubectl rollout restart` |
| Resource limits / 資源限制 | Not enforced | Enforced by kubelet |
| Namespace isolation / 命名空間隔離 | Not available | `supabase` namespace |

---

## License / 授權

This project configuration is provided for reference and deployment use.

Supabase is licensed under the [Apache License 2.0](https://github.com/supabase/supabase/blob/master/LICENSE).

此專案配置提供參考及部署使用。

Supabase 採用 [Apache License 2.0](https://github.com/supabase/supabase/blob/master/LICENSE) 授權。

---

**Maintained by / 維護者：** [WOOWTECH](https://github.com/WOOWTECH)

> For Podman/Docker Compose deployment, see the [`main` branch](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all/tree/main).
>
> Podman/Docker Compose 部署請參閱 [`main` 分支](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all/tree/main)。
