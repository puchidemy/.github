# Puchi

**Puchi** — nền tảng web học tiếng Việt.

Kiến trúc: **Next.js PWA (frontend) + Go microservices (backend)**, toàn bộ self-host trên Kubernetes, không lock-in SaaS.

---

## Tech Stack

### Frontend

| Thành phần | Công nghệ |
|---|---|
| Framework | Next.js 16 (App Router) + React 19 |
| Styling | Tailwind CSS v4 + Shadcn UI (New York) |
| State | Zustand v5 |
| Auth (tạm) | Clerk v6 → sẽ thay bằng Supertoken |
| i18n | next-intl (9 ngôn ngữ) |
| Animation | Motion (Framer Motion) + Rive |

### Backend

| Thành phần | Công nghệ |
|---|---|
| Ngôn ngữ | Go |
| Framework | Kratos v3 (Bilibili) — HTTP/gRPC hybrid, Protobuf-first, Wire DI |
| Auth | Supertoken (self-host) — session management, JWT, social login |
| API Gateway | Envoy Proxy — route, rate limit, TLS, JWT verify |
| Service Mesh | Envoy Sidecar (Istio) — mTLS, retry, circuit breaker, tracing |
| Database | PostgreSQL (per service) |
| Cache | Redis (leaderboard, session cache) |
| Message Queue | NATS — event-driven sync giữa các service |
| Object Storage | Garage (S3 API, self-host) — audio, ảnh, avatar |
| CDN | Cloudflare — cache static files trước Garage |
| Migration | goose |
| Auth SDK Go | supertokens-golang |

### DevOps

| Thành phần | Công nghệ |
|---|---|
| Container | Docker |
| Orchestration | Kubernetes (k0s) |
| CI/CD | GitHub Actions |
| Observability | OpenTelemetry (Kratos built-in) + Prometheus + Grafana |
| Service Mesh | Istio (Envoy sidecar) |

---

## Kiến trúc tổng thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Cloudflare (CDN + DNS)                           │
│              static.puchi.io.vn → cache audio/images                 │
└────────────┬────────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────────┐
│                    Next.js PWA (Frontend)                            │
│         Static export + i18n, không SSR auth                        │
└────────────┬────────────────────────────────────────────────────────┘
             │ HTTPS + JWT (Bearer token)
┌────────────▼────────────────────────────────────────────────────────┐
│                      Envoy Gateway                                   │
│                   api.puchi.io.vn                                    │
│          Route, rate limit, TLS terminate, JWT verify                │
└───┬──────┬───────┬───────┬───────┬───────┬──────────┬───────────────┘
    │      │       │       │       │       │          │
┌───▼──┐ ┌─▼───┐ ┌─▼───┐ ┌─▼───┐ ┌─▼───┐ ┌─▼────┐ ┌──▼──────────┐
│ Auth │ │Cont.│ │Grade│ │User │ │Game │ │Media │ │Notification │
└──┬───┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬───┘ └─────┬────────┘
   │        │       │       │       │       │           │
   └────────┼───────┼───────┼───────┼───────┼───────────┘
            │       │       │       │       │
       ┌────▼───────▼───────▼───────▼───────▼──────┐
       │                NATS                        │
       │       (event-driven sync)                  │
       └────┬───────────────────┬───────────────────┘
            │                   │
   ┌────────▼──────┐   ┌────────▼──────────┐
   │  PostgreSQL   │   │     Garage         │
   │  (per service)│   │ (audio, ảnh, file) │
   └───────────────┘   └───────────────────┘
            │
   ┌────────▼──────┐
   │    Redis       │
   │ (leaderboard,  │
   │  session cache)│
   └───────────────┘
```

---

## 7 Microservices

### 1. Auth Service
- **Trách nhiệm**: Register, login, logout, JWT issue/refresh
- **Transport**: HTTP (8080)
- **DB**: PostgreSQL (users, refresh_tokens)
- **Phụ thuộc**: Supertoken Core

### 2. Content Service
- **Trách nhiệm**: Courses, units, lessons, vocabulary, audio metadata
- **Transport**: HTTP (8080) + gRPC (9090)
- **DB**: PostgreSQL (courses, units, lessons, vocabulary, sentences)

### 3. Grading Service
- **Trách nhiệm**: Chấm dictation, listening, story; lịch sử làm bài; sau này tích hợp AI chấm phát âm
- **Transport**: HTTP (8080) + gRPC (9090)
- **DB**: PostgreSQL (answer_history, scores, mistakes)
- **Events emit**: `lesson.completed` → Game, User

### 4. User Service
- **Trách nhiệm**: Profile, avatar, settings, learning progress
- **Transport**: HTTP (8080) + gRPC (9090)
- **DB**: PostgreSQL (user_profiles, settings, progress)

### 5. Game Service
- **Trách nhiệm**: XP, level, leaderboard, quests, achievements, shop
- **Transport**: HTTP (8080) + gRPC (9090)
- **DB**: PostgreSQL (xp, achievements, quests, shop_items)
- **Cache**: Redis (leaderboard sorted set)
- **Events listen**: `lesson.completed` → +XP, check achievement

### 6. Media Service
- **Trách nhiệm**: Upload, resize ảnh, transcode audio, presigned URL, CDN cache invalidation
- **Transport**: HTTP (8080) + gRPC (9090)
- **Storage**: Garage (S3 API)
- **Events emit**: `media.ready` → Content, User
- **Processing**: Worker async: resize ảnh (3 sizes + webp), transcode audio (mp3 128kbps)

### 7. Notification Service
- **Trách nhiệm**: Push notification, email (streak, nhắc học, rank change)
- **Transport**: HTTP (8080) + gRPC (9090)
- **DB**: PostgreSQL (notification_templates, send_history)
- **Events listen**: `lesson.completed`, `level.up`, `streak.milestone`

---

## Infrastructure Services

| Thành phần | Vai trò |
|---|---|
| Supertoken Core | Session management, refresh token rotation, social login providers |
| NATS | Message broker — service này emit event, service khác listen |
| Envoy Gateway | Route request đến service, rate limit, TLS, JWT verify |
| Envoy Sidecar (Istio) | mTLS giữa các service, retry, circuit breaker, tracing |

---

## Event-driven Communication (NATS)

```
Grading Service ──► "lesson.completed" {userId, lessonId, score}
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Game Service  User Service  Notification
     (+XP,         (update        (push "Streak!")
      achievement)  progress)

Game Service ──► "level.up" {userId, newLevel}
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
     User Service  Content   Notification
     (unlock       (gợi ý    (push)
      features)     bài mới)
```

---

## Lộ trình triển khai

### Giai đoạn 1: Monolith modular (1-2 tháng)
- 1 service = Auth + User + Game (3 package)
- 1 service = Content (riêng vì core app)
- Media Service (basic: upload + resize)
- Kết nối Supertoken + Envoy Gateway

### Giai đoạn 2: Tách critical services (tháng 3-4)
- Tách Auth Service ra (security domain riêng)
- Tách Grading Service ra (chuẩn bị AI)
- Thêm NATS cho liên service communication

### Giai đoạn 3: Hoàn thiện (tháng 5-6)
- Tách Game Service (Redis leaderboard riêng)
- Tách User Service (profile độc lập)
- Notification Service
- Bật Istio service mesh

---

## Cấu trúc thư mục (monolith giai đoạn 1)

```
puchi-backend/
├── cmd/server/main.go
├── internal/
│   ├── auth/          # handler, service, repository, middleware
│   ├── content/       # courses, units, lessons, vocabulary
│   ├── grading/       # chấm bài, lịch sử
│   ├── user/          # profile, settings, progress
│   ├── game/          # XP, leaderboard, quests, shop
│   ├── media/         # upload, resize, transcode
│   └── notification/  # push, email
├── pkg/               # shared utilities
│   ├── jwt/
│   ├── response/
│   └── garage/
├── api/               # Protobuf definitions
│   ├── auth/v1/
│   ├── content/v1/
│   ├── grading/v1/
│   ├── user/v1/
│   ├── game/v1/
│   ├── media/v1/
│   └── notification/v1/
├── configs/           # Kratos config YAML
├── migrations/        # goose SQL
├── envoy/             # Envoy Gateway config
├── istio/             # Service mesh config (giai đoạn 3)
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── Makefile
└── go.mod
```

---

## Dev Local — docker-compose.yml

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: puchi
      POSTGRES_PASSWORD: puchi123
      POSTGRES_DB: supertokens
    ports:
      - "5432:5432"

  supertokens:
    image: registry.supertokens.io/supertokens/supertokens-postgresql:9.3.0
    depends_on:
      - postgres
    environment:
      POSTGRESQL_CONNECTION_URI: "postgresql://puchi:puchi123@postgres:5432/supertokens"
      API_KEYS: "dev-api-key-change-in-production"
    ports:
      - "3567:3567"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  nats:
    image: nats:2-alpine
    ports:
      - "4222:4222"

  garage:
    image: dxflrs/garage:latest
    ports:
      - "3900:3900"
      - "3901:3901"
```

---

## Tài liệu tham khảo

- [Kratos](https://go-kratos.dev/) — Go microservice framework
- [Supertoken](https://supertokens.com/) — Open source auth
- [Envoy](https://www.envoyproxy.io/) — API Gateway / Service Mesh
- [NATS](https://nats.io/) — Message broker
- [Garage](https://garagehq.deuxfleurs.fr/) — S3-compatible object storage
- [goose](https://github.com/pressly/goose) — Database migration
