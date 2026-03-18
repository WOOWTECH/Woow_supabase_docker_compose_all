# Supabase K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Open-source Firebase alternative providing a full backend-as-a-service platform. Supabase delivers a PostgreSQL database, authentication (GoTrue), instant RESTful APIs (PostgREST), real-time subscriptions (Realtime), file storage with image transformations (Storage + imgproxy), edge functions, and a management dashboard (Studio) - all self-hosted under your control. This deployment consists of 13 microservices orchestrated through Kustomize.

> **GitHub Repo (Podman/Docker):** [Woow_supabase_docker_compose_all](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all)

### Architecture

```
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                           External Access                                    │
    │              HTTP:  http://<node-ip>:30080                                    │
    │              HTTPS: https://<node-ip>:30443                                   │
    └──────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                    Kong API Gateway (NodePort)                                │
    │                    kong:2  |  Ports: 8000 / 8443                              │
    │                    Routes all external traffic to internal services           │
    └───────┬──────────┬──────────┬──────────┬──────────┬──────────┬───────────────┘
            │          │          │          │          │          │
            ▼          ▼          ▼          ▼          ▼          ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │   Auth   │ │   REST   │ │ Realtime │ │ Storage  │ │   Meta   │ │Functions │
    │ (GoTrue) │ │(PostgRST)│ │          │ │          │ │          │ │  (Edge)  │
    │ :9999    │ │ :3000    │ │ :4000    │ │ :5000    │ │ :8080    │ │ :9000    │
    │          │ │          │ │          │ │          │ │          │ │          │
    │ JWT auth │ │ Auto API │ │ WebSocket│ │ S3-compat│ │ DB meta  │ │ Deno     │
    │ user mgmt│ │ from DB  │ │ pub/sub  │ │ file API │ │ for UI   │ │ runtime  │
    └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
         │            │            │            │            │            │
         └────────────┴────────┬───┴────────────┴────────────┘            │
                               │                                          │
                               ▼                                          │
    ┌──────────────────────────────────────────────────────────────┐       │
    │                    PostgreSQL (StatefulSet)                   │◄──────┘
    │                    supabase/postgres  |  Port: 5432           │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐   │
    │  │  PVC: db-data (8Gi)     /var/lib/postgresql/data       │   │
    │  │  PVC: db-config (1Gi)   /etc/postgresql-custom         │   │
    │  └────────────────────────────────────────────────────────┘   │
    └─────────┬──────────────────────────────┬─────────────────────┘
              │                              │
              ▼                              ▼
    ┌──────────────────┐           ┌──────────────────┐
    │    Supavisor     │           │    Analytics     │
    │    (pooler)      │           │   (Logflare)     │
    │ :5432 transaction│           │    :4000         │
    │ :6543 session    │           └────────┬─────────┘
    └──────────────────┘                    │
                                            ▼
                                  ┌──────────────────┐
                                  │     Vector       │
                                  │  (log pipeline)  │
                                  │    :9001         │
                                  └──────────────────┘

    ┌──────────────────┐           ┌──────────────────┐
    │     Studio       │           │    imgproxy      │
    │  (Dashboard)     │           │ (image transform)│
    │    :3000         │           │    :5001         │
    │  via Kong        │           │  used by Storage │
    └──────────────────┘           └──────────────────┘
```

**Component Dependency Map:**

```
    PostgreSQL ◄─── Auth (GoTrue)
         ▲  ◄───── REST (PostgREST)
         │  ◄───── Realtime
         │  ◄───── Storage ──────► imgproxy
         │  ◄───── Meta
         │  ◄───── Functions (Edge Runtime)
         │  ◄───── Analytics (Logflare)
         │  ◄───── Supavisor (connection pooler)
         │
         └──────── Vector (log collector)

    Kong ◄───────── Studio (Dashboard UI)
      ▲  ◄───────── All API traffic
      │
    External clients / SDKs
```

### Features

- PostgreSQL database with Supabase extensions
- GoTrue authentication with JWT, email/password, OAuth providers
- PostgREST auto-generated RESTful API from database schema
- Real-time WebSocket subscriptions for database changes
- S3-compatible file storage with imgproxy image transformations
- Deno-based edge functions runtime
- Studio web dashboard for database, auth, and storage management
- Supavisor connection pooling (transaction and session modes)
- Logflare analytics with Vector log pipeline
- Kong API gateway as single entry point
- Client SDK compatibility (`@supabase/supabase-js`)

### Quick Start

```bash
# 1. Update secrets before deploying (CRITICAL - change all placeholder values)
nano k8s-manifests/supabase/secret.yaml

# 2. Update configmap URLs for your domain
nano k8s-manifests/supabase/configmap.yaml

# 3. Deploy all Supabase components
kubectl apply -k k8s-manifests/supabase/

# 4. Wait for database to be ready (other services depend on it)
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# 5. Verify all pods are running
kubectl -n supabase get pods

# 6. Watch deployment progress
kubectl -n supabase get pods -w
```

### Configuration

#### Secrets (MUST change before deploying)

Edit `secret.yaml` and replace ALL placeholder values. Values are base64-encoded.

```bash
# Encode a value
echo -n 'your-secret-value' | base64

# Decode a value
echo 'base64string' | base64 -d
```

| Secret Key | Description | Notes |
|------------|-------------|-------|
| `POSTGRES_PASSWORD` | PostgreSQL superuser password | Use a strong random password |
| `JWT_SECRET` | JWT signing secret | Minimum 32 characters |
| `ANON_KEY` | Public anonymous JWT | Generate with supabase CLI or jwt.io |
| `SERVICE_ROLE_KEY` | Service role JWT (elevated privileges) | Keep server-side only, never expose |
| `DASHBOARD_USERNAME` | Studio dashboard login username | Default: `supabase` |
| `DASHBOARD_PASSWORD` | Studio dashboard login password | Change from default |
| `LOGFLARE_PUBLIC_ACCESS_TOKEN` | Logflare public API token | For log ingestion |
| `LOGFLARE_PRIVATE_ACCESS_TOKEN` | Logflare private API token | For admin operations |
| `SECRET_KEY_BASE` | Erlang/Phoenix secret key (Analytics) | Minimum 64 characters |
| `VAULT_ENC_KEY` | Database vault encryption key | For encrypted columns |
| `PG_META_CRYPTO_KEY` | Postgres Meta encryption key | For metadata encryption |

```bash
# Generate JWT keys using supabase CLI (recommended)
supabase gen keys --experimental

# Or generate strong random passwords
openssl rand -base64 32
```

#### Environment Variables (ConfigMap)

Key variables to adjust in `configmap.yaml`:

| Variable | Description | Default |
|----------|-------------|---------|
| `SITE_URL` | Your application URL | `http://localhost` |
| `API_EXTERNAL_URL` | External API gateway URL | `http://localhost:8000` |
| `SUPABASE_PUBLIC_URL` | Public Supabase URL | `http://localhost:8000` |
| `DISABLE_SIGNUP` | Disable new user registration | `false` |
| `ENABLE_EMAIL_AUTOCONFIRM` | Auto-confirm email signups | `true` |
| `GOTRUE_SMTP_HOST` | SMTP server for email delivery | `` (empty - configure for production) |
| `GOTRUE_SMTP_PORT` | SMTP server port | `587` |
| `GOTRUE_SMTP_ADMIN_EMAIL` | From address for emails | `admin@example.com` |
| `FILE_SIZE_LIMIT` | Maximum upload file size (bytes) | `52428800` (50MB) |
| `POOLER_DEFAULT_POOL_SIZE` | Supavisor pool size per tenant | `20` |
| `POOLER_MAX_CLIENT_CONN` | Maximum client connections | `200` |

### Components

| Component | Image | Port | Purpose |
|-----------|-------|------|---------|
| PostgreSQL | `supabase/postgres` | 5432 | Primary database (with Supabase extensions) |
| Kong | `kong:2` | 8000/8443 | API gateway - single entry point for all services |
| GoTrue (Auth) | `supabase/gotrue` | 9999 | Authentication, user management, JWT issuance |
| PostgREST (REST) | `postgrest/postgrest` | 3000 | Auto-generated RESTful API from database schema |
| Realtime | `supabase/realtime` | 4000 | WebSocket-based real-time database change subscriptions |
| Storage | `supabase/storage-api` | 5000 | S3-compatible file storage API |
| imgproxy | `darthsim/imgproxy` | 5001 | On-the-fly image resizing and transformation |
| Meta | `supabase/postgres-meta` | 8080 | Database metadata API (used by Studio) |
| Functions | `supabase/edge-runtime` | 9000 | Deno-based edge functions runtime |
| Studio | `supabase/studio` | 3000 | Web-based management dashboard |
| Supavisor | `supabase/supavisor` | 5432/6543 | Connection pooler (transaction/session modes) |
| Analytics | `supabase/logflare` | 4000 | Log ingestion and query engine |
| Vector | `timberio/vector` | 9001 | Log collection and forwarding pipeline |

### Accessing the Service

| Endpoint | URL | Protocol |
|----------|-----|----------|
| Supabase API (Kong) | `http://<node-ip>:30080` | HTTP (NodePort) |
| Supabase API (HTTPS) | `https://<node-ip>:30443` | HTTPS (NodePort) |
| Studio Dashboard | `http://<node-ip>:30080` (via Kong) | HTTP |
| Internal API | `http://kong.supabase.svc.cluster.local:8000` | HTTP |

#### Client SDK Configuration

```javascript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'http://<node-ip>:30080',  // API_EXTERNAL_URL
  '<your-anon-key>'           // ANON_KEY from secret.yaml
)
```

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `db-data` | `/var/lib/postgresql/data` | 8Gi | PostgreSQL database files |
| `db-config` | `/etc/postgresql-custom` | 1Gi | PostgreSQL custom configuration |
| `storage-data` | `/var/lib/storage` | 10Gi | Uploaded files (Storage API) |

All PVCs use the `local-path` storage class (k3s default). For production, consider network-attached storage (Longhorn, Ceph, NFS).

### Backup & Restore

#### Backup

```bash
# 1. Backup PostgreSQL database (full dump)
kubectl -n supabase exec sts/db -- pg_dumpall -U supabase_admin > supabase-full-backup.sql

# 2. Backup just the application database
kubectl -n supabase exec sts/db -- pg_dump -U supabase_admin postgres > supabase-db-backup.sql

# 3. Backup storage files
kubectl -n supabase exec deploy/storage -- tar czf /tmp/storage-backup.tar.gz /var/lib/storage
kubectl -n supabase cp supabase/<storage-pod>:/tmp/storage-backup.tar.gz ./storage-backup.tar.gz

# 4. Backup secrets (store securely!)
kubectl -n supabase get secret supabase-secret -o yaml > supabase-secret-backup.yaml
```

#### Restore

```bash
# 1. Restore database
kubectl -n supabase exec -i sts/db -- psql -U supabase_admin postgres < supabase-db-backup.sql

# 2. Restore storage files
kubectl -n supabase cp ./storage-backup.tar.gz supabase/<storage-pod>:/tmp/storage-backup.tar.gz
kubectl -n supabase exec deploy/storage -- tar xzf /tmp/storage-backup.tar.gz -C /

# 3. Restart services to pick up restored data
kubectl -n supabase rollout restart deploy --all
```

### Useful Commands

```bash
# Check all pod status
kubectl -n supabase get pods

# View logs for a specific service
kubectl -n supabase logs deploy/kong -f
kubectl -n supabase logs deploy/auth -f
kubectl -n supabase logs deploy/rest -f
kubectl -n supabase logs deploy/realtime -f
kubectl -n supabase logs deploy/storage -f
kubectl -n supabase logs deploy/studio -f
kubectl -n supabase logs sts/db -f

# Wait for database readiness
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# Restart all services
kubectl -n supabase rollout restart deploy --all

# Check PVC status
kubectl -n supabase get pvc

# Connect to PostgreSQL interactively
kubectl -n supabase exec -it sts/db -- psql -U supabase_admin postgres

# Verify JWT secret consistency
kubectl -n supabase get secret supabase-secret -o jsonpath='{.data.JWT_SECRET}' | base64 -d

# Test Kong gateway
curl http://<node-ip>:30080/rest/v1/ -H "apikey: <your-anon-key>"
```

### Troubleshooting

#### Database not starting

```bash
# Check database pod status and logs
kubectl -n supabase get pods -l component=db
kubectl -n supabase logs sts/db

# Check PVC is bound
kubectl -n supabase get pvc db-data
```

#### Services stuck in CrashLoopBackOff

Most services depend on PostgreSQL. Ensure the database is ready first:

```bash
# Wait for database
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# Restart all deployments to retry connections
kubectl -n supabase rollout restart deploy --all
```

#### Kong returning 502 errors

The upstream service is not reachable. Check if the target service pod is running:

```bash
# List all pods to find which service is down
kubectl -n supabase get pods

# Check Kong logs for the specific upstream
kubectl -n supabase logs deploy/kong
```

#### Studio not loading

Studio connects through Kong. Verify both are running:

```bash
kubectl -n supabase get pods -l component=studio
kubectl -n supabase get pods -l component=kong
kubectl -n supabase logs deploy/studio
```

#### Authentication not working

```bash
# Check GoTrue auth service
kubectl -n supabase get pods -l component=auth
kubectl -n supabase logs deploy/auth

# Verify JWT secret matches between auth and other services
kubectl -n supabase get secret supabase-secret -o jsonpath='{.data.JWT_SECRET}' | base64 -d
```

#### Realtime subscriptions failing

```bash
# Check realtime service
kubectl -n supabase logs deploy/realtime

# Verify database connectivity from realtime pod
kubectl -n supabase exec deploy/realtime -- nc -zv db 5432
```

#### Storage upload failures

```bash
# Check storage service
kubectl -n supabase logs deploy/storage

# Verify PVC has available space
kubectl -n supabase exec deploy/storage -- df -h /var/lib/storage

# Check imgproxy for image transformation issues
kubectl -n supabase logs deploy/imgproxy
```

### Production Checklist

- [ ] Replace ALL placeholder values in `secret.yaml` with strong random values
- [ ] Generate proper JWT keys (`ANON_KEY` and `SERVICE_ROLE_KEY`) using `supabase gen keys`
- [ ] Update `SITE_URL`, `API_EXTERNAL_URL`, and `SUPABASE_PUBLIC_URL` in configmap
- [ ] Configure SMTP settings for email delivery (`GOTRUE_SMTP_*`)
- [ ] Set `ENABLE_EMAIL_AUTOCONFIRM: "false"` for production
- [ ] Consider increasing PVC sizes for `db-data` and `storage-data`
- [ ] Switch to network-attached storage class for high availability
- [ ] Place behind a TLS-terminating reverse proxy (Nginx, Traefik, Cloudflare Tunnel)
- [ ] Set proper `FILE_SIZE_LIMIT` based on your requirements
- [ ] Review and adjust connection pooler settings (`POOLER_*`)
- [ ] Set up automated database backups
- [ ] Monitor resource usage and adjust `resources.requests/limits`

### File Structure

```
supabase/
├── kustomization.yaml              # Kustomize orchestration (all resources)
├── namespace.yaml                  # Namespace: supabase
├── secret.yaml                     # All sensitive credentials (base64)
├── configmap.yaml                  # Shared non-sensitive configuration
├── pvc.yaml                        # PVCs: db-data (8Gi), db-config (1Gi), storage-data (10Gi)
├── db-statefulset.yaml             # PostgreSQL StatefulSet (with Supabase extensions)
├── db-service.yaml                 # PostgreSQL ClusterIP service
├── kong-deployment.yaml            # Kong API gateway Deployment
├── kong-service.yaml               # Kong NodePort service (30080/30443)
├── auth-deployment.yaml            # GoTrue authentication Deployment
├── auth-service.yaml               # GoTrue ClusterIP service
├── rest-deployment.yaml            # PostgREST API Deployment
├── rest-service.yaml               # PostgREST ClusterIP service
├── realtime-deployment.yaml        # Realtime WebSocket Deployment
├── realtime-service.yaml           # Realtime ClusterIP service
├── storage-deployment.yaml         # Storage API Deployment
├── storage-service.yaml            # Storage ClusterIP service
├── imgproxy-deployment.yaml        # imgproxy image transformer Deployment
├── imgproxy-service.yaml           # imgproxy ClusterIP service
├── meta-deployment.yaml            # Postgres Meta API Deployment
├── meta-service.yaml               # Postgres Meta ClusterIP service
├── functions-deployment.yaml       # Edge Functions runtime Deployment
├── functions-service.yaml          # Edge Functions ClusterIP service
├── studio-deployment.yaml          # Studio dashboard Deployment
├── studio-service.yaml             # Studio ClusterIP service
├── supavisor-deployment.yaml       # Connection pooler Deployment
├── supavisor-service.yaml          # Connection pooler ClusterIP service
├── analytics-deployment.yaml       # Logflare analytics Deployment
├── analytics-service.yaml          # Logflare ClusterIP service
├── vector-deployment.yaml          # Vector log pipeline Deployment
├── vector-service.yaml             # Vector ClusterIP service
└── README.md                       # This file
```

---

## 中文

### 概述

開源 Firebase 替代方案，提供完整的後端即服務平台。Supabase 提供 PostgreSQL 資料庫、身份驗證（GoTrue）、即時 RESTful API（PostgREST）、即時訂閱（Realtime）、檔案儲存及影像轉換（Storage + imgproxy）、邊緣函式，以及管理儀表板（Studio）- 全部自行託管，完全掌控。本部署由 13 個微服務組成，透過 Kustomize 編排。

> **GitHub 儲存庫 (Podman/Docker)：** [Woow_supabase_docker_compose_all](https://github.com/WOOWTECH/Woow_supabase_docker_compose_all)

### 架構

```
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                           外部存取                                           │
    │              HTTP:  http://<node-ip>:30080                                    │
    │              HTTPS: https://<node-ip>:30443                                   │
    └──────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                                   ▼
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                    Kong API 閘道器 (NodePort)                                 │
    │                    kong:2  |  連接埠: 8000 / 8443                             │
    │                    將所有外部流量路由至內部服務                                  │
    └───────┬──────────┬──────────┬──────────┬──────────┬──────────┬───────────────┘
            │          │          │          │          │          │
            ▼          ▼          ▼          ▼          ▼          ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │  身份驗證 │ │   REST   │ │  即時    │ │   儲存   │ │  中繼資料 │ │  函式    │
    │ (GoTrue) │ │(PostgRST)│ │ Realtime │ │ Storage  │ │   Meta   │ │  (Edge)  │
    │ :9999    │ │ :3000    │ │ :4000    │ │ :5000    │ │ :8080    │ │ :9000    │
    │          │ │          │ │          │ │          │ │          │ │          │
    │ JWT 驗證 │ │ 自動 API │ │ WebSocket│ │S3 相容   │ │ 資料庫   │ │ Deno     │
    │ 使用者   │ │ 從資料庫 │ │ 發布/訂閱│ │ 檔案 API │ │ 供 UI 用 │ │ 執行環境 │
    └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
         │            │            │            │            │            │
         └────────────┴────────┬───┴────────────┴────────────┘            │
                               │                                          │
                               ▼                                          │
    ┌──────────────────────────────────────────────────────────────┐       │
    │                    PostgreSQL (StatefulSet)                   │◄──────┘
    │                    supabase/postgres  |  連接埠: 5432         │
    │                                                              │
    │  ┌────────────────────────────────────────────────────────┐   │
    │  │  PVC: db-data (8Gi)     /var/lib/postgresql/data       │   │
    │  │  PVC: db-config (1Gi)   /etc/postgresql-custom         │   │
    │  └────────────────────────────────────────────────────────┘   │
    └─────────┬──────────────────────────────┬─────────────────────┘
              │                              │
              ▼                              ▼
    ┌──────────────────┐           ┌──────────────────┐
    │    Supavisor     │           │    Analytics     │
    │   (連線池)       │           │   (Logflare)     │
    │ :5432 交易模式   │           │    :4000         │
    │ :6543 會話模式   │           └────────┬─────────┘
    └──────────────────┘                    │
                                            ▼
                                  ┌──────────────────┐
                                  │     Vector       │
                                  │   (日誌管線)     │
                                  │    :9001         │
                                  └──────────────────┘

    ┌──────────────────┐           ┌──────────────────┐
    │     Studio       │           │    imgproxy      │
    │   (儀表板)       │           │  (影像轉換)      │
    │    :3000         │           │    :5001         │
    │  經由 Kong       │           │ 供 Storage 使用  │
    └──────────────────┘           └──────────────────┘
```

**元件相依關係圖：**

```
    PostgreSQL ◄─── Auth (GoTrue，身份驗證)
         ▲  ◄───── REST (PostgREST，自動 API)
         │  ◄───── Realtime（即時訂閱）
         │  ◄───── Storage（檔案儲存）──────► imgproxy（影像轉換）
         │  ◄───── Meta（中繼資料 API）
         │  ◄───── Functions（邊緣函式執行環境）
         │  ◄───── Analytics（Logflare 分析）
         │  ◄───── Supavisor（連線池）
         │
         └──────── Vector（日誌收集器）

    Kong ◄───────── Studio（儀表板 UI）
      ▲  ◄───────── 所有 API 流量
      │
    外部用戶端 / SDK
```

### 功能特色

- 具有 Supabase 擴充套件的 PostgreSQL 資料庫
- GoTrue 身份驗證，支援 JWT、電子郵件/密碼、OAuth 提供者
- PostgREST 從資料庫結構自動產生 RESTful API
- 即時 WebSocket 訂閱資料庫變更
- S3 相容檔案儲存，搭配 imgproxy 影像轉換
- 基於 Deno 的邊緣函式執行環境
- Studio 網頁儀表板，管理資料庫、身份驗證和儲存
- Supavisor 連線池（交易模式和會話模式）
- Logflare 分析搭配 Vector 日誌管線
- Kong API 閘道器作為單一入口點
- 用戶端 SDK 相容（`@supabase/supabase-js`）

### 快速開始

```bash
# 1. 部署前更新 Secret（重要 - 必須更改所有預設值）
nano k8s-manifests/supabase/secret.yaml

# 2. 更新 configmap 中的 URL 為您的網域
nano k8s-manifests/supabase/configmap.yaml

# 3. 部署所有 Supabase 元件
kubectl apply -k k8s-manifests/supabase/

# 4. 等待資料庫就緒（其他服務依賴它）
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# 5. 確認所有 Pod 正在運行
kubectl -n supabase get pods

# 6. 觀察部署進度
kubectl -n supabase get pods -w
```

### 設定

#### Secret（部署前必須更改）

編輯 `secret.yaml` 並替換所有預設值。值為 base64 編碼。

```bash
# 編碼一個值
echo -n 'your-secret-value' | base64

# 解碼一個值
echo 'base64string' | base64 -d
```

| Secret 鍵 | 說明 | 備註 |
|-----------|------|------|
| `POSTGRES_PASSWORD` | PostgreSQL 超級使用者密碼 | 使用強隨機密碼 |
| `JWT_SECRET` | JWT 簽署密鑰 | 最少 32 字元 |
| `ANON_KEY` | 公開匿名 JWT | 使用 supabase CLI 或 jwt.io 產生 |
| `SERVICE_ROLE_KEY` | 服務角色 JWT（提升權限） | 僅限伺服器端使用，切勿暴露 |
| `DASHBOARD_USERNAME` | Studio 儀表板登入使用者名稱 | 預設：`supabase` |
| `DASHBOARD_PASSWORD` | Studio 儀表板登入密碼 | 請更改預設值 |
| `LOGFLARE_PUBLIC_ACCESS_TOKEN` | Logflare 公開 API 權杖 | 用於日誌擷取 |
| `LOGFLARE_PRIVATE_ACCESS_TOKEN` | Logflare 私有 API 權杖 | 用於管理操作 |
| `SECRET_KEY_BASE` | Erlang/Phoenix 密鑰（Analytics） | 最少 64 字元 |
| `VAULT_ENC_KEY` | 資料庫 Vault 加密金鑰 | 用於加密欄位 |
| `PG_META_CRYPTO_KEY` | Postgres Meta 加密金鑰 | 用於中繼資料加密 |

```bash
# 使用 supabase CLI 產生 JWT 金鑰（建議）
supabase gen keys --experimental

# 或產生強隨機密碼
openssl rand -base64 32
```

#### 環境變數（ConfigMap）

在 `configmap.yaml` 中需調整的關鍵變數：

| 變數 | 說明 | 預設值 |
|------|------|--------|
| `SITE_URL` | 您的應用程式 URL | `http://localhost` |
| `API_EXTERNAL_URL` | 外部 API 閘道器 URL | `http://localhost:8000` |
| `SUPABASE_PUBLIC_URL` | 公開 Supabase URL | `http://localhost:8000` |
| `DISABLE_SIGNUP` | 停用新使用者註冊 | `false` |
| `ENABLE_EMAIL_AUTOCONFIRM` | 自動確認電子郵件註冊 | `true` |
| `GOTRUE_SMTP_HOST` | 電子郵件傳送 SMTP 伺服器 | ``（空白 - 正式環境需設定） |
| `GOTRUE_SMTP_PORT` | SMTP 伺服器連接埠 | `587` |
| `GOTRUE_SMTP_ADMIN_EMAIL` | 電子郵件寄件地址 | `admin@example.com` |
| `FILE_SIZE_LIMIT` | 最大上傳檔案大小（位元組） | `52428800`（50MB） |
| `POOLER_DEFAULT_POOL_SIZE` | Supavisor 每租戶連線池大小 | `20` |
| `POOLER_MAX_CLIENT_CONN` | 最大用戶端連線數 | `200` |

### 元件

| 元件 | 映像檔 | 連接埠 | 用途 |
|------|--------|--------|------|
| PostgreSQL | `supabase/postgres` | 5432 | 主資料庫（含 Supabase 擴充套件） |
| Kong | `kong:2` | 8000/8443 | API 閘道器 - 所有服務的單一入口點 |
| GoTrue（Auth） | `supabase/gotrue` | 9999 | 身份驗證、使用者管理、JWT 發放 |
| PostgREST（REST） | `postgrest/postgrest` | 3000 | 從資料庫結構自動產生 RESTful API |
| Realtime | `supabase/realtime` | 4000 | 基於 WebSocket 的即時資料庫變更訂閱 |
| Storage | `supabase/storage-api` | 5000 | S3 相容檔案儲存 API |
| imgproxy | `darthsim/imgproxy` | 5001 | 即時影像縮放與轉換 |
| Meta | `supabase/postgres-meta` | 8080 | 資料庫中繼資料 API（供 Studio 使用） |
| Functions | `supabase/edge-runtime` | 9000 | 基於 Deno 的邊緣函式執行環境 |
| Studio | `supabase/studio` | 3000 | 網頁管理儀表板 |
| Supavisor | `supabase/supavisor` | 5432/6543 | 連線池（交易/會話模式） |
| Analytics | `supabase/logflare` | 4000 | 日誌擷取與查詢引擎 |
| Vector | `timberio/vector` | 9001 | 日誌收集與轉發管線 |

### 存取服務

| 端點 | URL | 協定 |
|------|-----|------|
| Supabase API (Kong) | `http://<node-ip>:30080` | HTTP (NodePort) |
| Supabase API (HTTPS) | `https://<node-ip>:30443` | HTTPS (NodePort) |
| Studio 儀表板 | `http://<node-ip>:30080`（經由 Kong） | HTTP |
| 內部 API | `http://kong.supabase.svc.cluster.local:8000` | HTTP |

#### 用戶端 SDK 設定

```javascript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  'http://<node-ip>:30080',  // API_EXTERNAL_URL
  '<your-anon-key>'           // secret.yaml 中的 ANON_KEY
)
```

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|-----------|----------|------|------|
| `db-data` | `/var/lib/postgresql/data` | 8Gi | PostgreSQL 資料庫檔案 |
| `db-config` | `/etc/postgresql-custom` | 1Gi | PostgreSQL 自訂設定 |
| `storage-data` | `/var/lib/storage` | 10Gi | 上傳的檔案（Storage API） |

所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。正式環境建議考慮網路附加儲存（Longhorn、Ceph、NFS）。

### 備份與還原

#### 備份

```bash
# 1. 備份 PostgreSQL 資料庫（完整轉儲）
kubectl -n supabase exec sts/db -- pg_dumpall -U supabase_admin > supabase-full-backup.sql

# 2. 僅備份應用程式資料庫
kubectl -n supabase exec sts/db -- pg_dump -U supabase_admin postgres > supabase-db-backup.sql

# 3. 備份儲存檔案
kubectl -n supabase exec deploy/storage -- tar czf /tmp/storage-backup.tar.gz /var/lib/storage
kubectl -n supabase cp supabase/<storage-pod>:/tmp/storage-backup.tar.gz ./storage-backup.tar.gz

# 4. 備份 Secret（請安全保存！）
kubectl -n supabase get secret supabase-secret -o yaml > supabase-secret-backup.yaml
```

#### 還原

```bash
# 1. 還原資料庫
kubectl -n supabase exec -i sts/db -- psql -U supabase_admin postgres < supabase-db-backup.sql

# 2. 還原儲存檔案
kubectl -n supabase cp ./storage-backup.tar.gz supabase/<storage-pod>:/tmp/storage-backup.tar.gz
kubectl -n supabase exec deploy/storage -- tar xzf /tmp/storage-backup.tar.gz -C /

# 3. 重新啟動服務以載入還原的資料
kubectl -n supabase rollout restart deploy --all
```

### 實用指令

```bash
# 檢查所有 Pod 狀態
kubectl -n supabase get pods

# 檢視特定服務的日誌
kubectl -n supabase logs deploy/kong -f
kubectl -n supabase logs deploy/auth -f
kubectl -n supabase logs deploy/rest -f
kubectl -n supabase logs deploy/realtime -f
kubectl -n supabase logs deploy/storage -f
kubectl -n supabase logs deploy/studio -f
kubectl -n supabase logs sts/db -f

# 等待資料庫就緒
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# 重新啟動所有服務
kubectl -n supabase rollout restart deploy --all

# 檢查 PVC 狀態
kubectl -n supabase get pvc

# 互動式連線到 PostgreSQL
kubectl -n supabase exec -it sts/db -- psql -U supabase_admin postgres

# 驗證 JWT Secret 一致性
kubectl -n supabase get secret supabase-secret -o jsonpath='{.data.JWT_SECRET}' | base64 -d

# 測試 Kong 閘道器
curl http://<node-ip>:30080/rest/v1/ -H "apikey: <your-anon-key>"
```

### 疑難排解

#### 資料庫無法啟動

```bash
# 檢查資料庫 Pod 狀態和日誌
kubectl -n supabase get pods -l component=db
kubectl -n supabase logs sts/db

# 檢查 PVC 是否已綁定
kubectl -n supabase get pvc db-data
```

#### 服務卡在 CrashLoopBackOff

大多數服務依賴 PostgreSQL。請先確保資料庫已就緒：

```bash
# 等待資料庫就緒
kubectl -n supabase wait --for=condition=ready pod -l component=db --timeout=120s

# 重新啟動所有 Deployment 以重試連線
kubectl -n supabase rollout restart deploy --all
```

#### Kong 返回 502 錯誤

上游服務無法連線。檢查目標服務 Pod 是否正在運行：

```bash
# 列出所有 Pod 以找出哪個服務已停止
kubectl -n supabase get pods

# 檢查 Kong 日誌以查看特定上游服務
kubectl -n supabase logs deploy/kong
```

#### Studio 無法載入

Studio 透過 Kong 連線。確認兩者都在運行：

```bash
kubectl -n supabase get pods -l component=studio
kubectl -n supabase get pods -l component=kong
kubectl -n supabase logs deploy/studio
```

#### 身份驗證無法運作

```bash
# 檢查 GoTrue 身份驗證服務
kubectl -n supabase get pods -l component=auth
kubectl -n supabase logs deploy/auth

# 驗證 JWT Secret 在各服務間是否一致
kubectl -n supabase get secret supabase-secret -o jsonpath='{.data.JWT_SECRET}' | base64 -d
```

#### 即時訂閱失敗

```bash
# 檢查 Realtime 服務
kubectl -n supabase logs deploy/realtime

# 從 Realtime Pod 驗證資料庫連線
kubectl -n supabase exec deploy/realtime -- nc -zv db 5432
```

#### 儲存上傳失敗

```bash
# 檢查 Storage 服務
kubectl -n supabase logs deploy/storage

# 驗證 PVC 有可用空間
kubectl -n supabase exec deploy/storage -- df -h /var/lib/storage

# 檢查 imgproxy 影像轉換問題
kubectl -n supabase logs deploy/imgproxy
```

### 正式環境檢查清單

- [ ] 將 `secret.yaml` 中的所有預設值替換為強隨機值
- [ ] 使用 `supabase gen keys` 產生正確的 JWT 金鑰（`ANON_KEY` 和 `SERVICE_ROLE_KEY`）
- [ ] 更新 configmap 中的 `SITE_URL`、`API_EXTERNAL_URL` 和 `SUPABASE_PUBLIC_URL`
- [ ] 設定電子郵件傳送的 SMTP 設定（`GOTRUE_SMTP_*`）
- [ ] 正式環境設定 `ENABLE_EMAIL_AUTOCONFIRM: "false"`
- [ ] 考慮增加 `db-data` 和 `storage-data` 的 PVC 大小
- [ ] 切換至網路附加儲存類別以獲得高可用性
- [ ] 放置在 TLS 終止反向代理後方（Nginx、Traefik、Cloudflare Tunnel）
- [ ] 根據需求設定適當的 `FILE_SIZE_LIMIT`
- [ ] 檢視並調整連線池設定（`POOLER_*`）
- [ ] 設定自動化資料庫備份
- [ ] 監控資源使用量並調整 `resources.requests/limits`

### 檔案結構

```
supabase/
├── kustomization.yaml              # Kustomize 編排檔（所有資源）
├── namespace.yaml                  # 命名空間: supabase
├── secret.yaml                     # 所有敏感憑證（base64 編碼）
├── configmap.yaml                  # 共用非敏感設定
├── pvc.yaml                        # PVC: db-data (8Gi), db-config (1Gi), storage-data (10Gi)
├── db-statefulset.yaml             # PostgreSQL StatefulSet（含 Supabase 擴充套件）
├── db-service.yaml                 # PostgreSQL ClusterIP 服務
├── kong-deployment.yaml            # Kong API 閘道器 Deployment
├── kong-service.yaml               # Kong NodePort 服務 (30080/30443)
├── auth-deployment.yaml            # GoTrue 身份驗證 Deployment
├── auth-service.yaml               # GoTrue ClusterIP 服務
├── rest-deployment.yaml            # PostgREST API Deployment
├── rest-service.yaml               # PostgREST ClusterIP 服務
├── realtime-deployment.yaml        # Realtime WebSocket Deployment
├── realtime-service.yaml           # Realtime ClusterIP 服務
├── storage-deployment.yaml         # Storage API Deployment
├── storage-service.yaml            # Storage ClusterIP 服務
├── imgproxy-deployment.yaml        # imgproxy 影像轉換 Deployment
├── imgproxy-service.yaml           # imgproxy ClusterIP 服務
├── meta-deployment.yaml            # Postgres Meta API Deployment
├── meta-service.yaml               # Postgres Meta ClusterIP 服務
├── functions-deployment.yaml       # 邊緣函式執行環境 Deployment
├── functions-service.yaml          # 邊緣函式 ClusterIP 服務
├── studio-deployment.yaml          # Studio 儀表板 Deployment
├── studio-service.yaml             # Studio ClusterIP 服務
├── supavisor-deployment.yaml       # 連線池 Deployment
├── supavisor-service.yaml          # 連線池 ClusterIP 服務
├── analytics-deployment.yaml       # Logflare 分析 Deployment
├── analytics-service.yaml          # Logflare ClusterIP 服務
├── vector-deployment.yaml          # Vector 日誌管線 Deployment
├── vector-service.yaml             # Vector ClusterIP 服務
└── README.md                       # 本檔案
```
