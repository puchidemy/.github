# Puchi

**Puchi** — nền tảng web học tiếng Việt.

Kiến trúc: **Next.js PWA (frontend) + Go microservices (backend)**, self-host trên K3s cluster, GitOps với ArgoCD.

---

## Repositories

| Repo | Mô tả |
|------|-------|
| [puchi-frontend](https://github.com/puchidemy/puchi-frontend) | Next.js 16 PWA (React 19, Tailwind v4, Shadcn UI, Bun) |
| [puchi-backend](https://github.com/puchidemy/puchi-backend) | Go monorepo — 7 services với Kratos v3 |
| [puchi-infra](https://github.com/puchidemy/puchi-infra) | Infrastructure as Code — ArgoCD GitOps trên K3s |

---

## Kiến trúc

```
User ──► Cloudflare CDN ──► Next.js PWA ──► Envoy Gateway ──► Go Services ──► PostgreSQL / NATS / Garage
(puchi.io.vn)                                  (api.puchi.io.vn)   (Kratos v3)    (K3s cluster)
```

### Frontend

| Thành phần | Công nghệ |
|------------|-----------|
| Framework | Next.js 16 (App Router) + React 19 |
| Styling | Tailwind CSS v4 + Shadcn UI (New York) |
| State | Zustand v5 |
| Runtime | Bun |
| i18n | next-intl (9 ngôn ngữ) |
| Animation | Motion + Rive |
| Auth (tạm) | Clerk v6 → Supertoken |
| CI/CD | GitHub Actions → GHCR |

### Backend

| Thành phần | Công nghệ |
|------------|-----------|
| Ngôn ngữ | Go 1.26 |
| Framework | Kratos v3 (HTTP/gRPC hybrid, Protobuf-first, Wire DI) |
| Auth | Supertoken (self-host) |
| API Gateway | Envoy Gateway |
| Database | PostgreSQL 18 (CloudNativePG) |
| Cache | Valkey / Redis |
| Message Queue | NATS (event-driven) |
| Object Storage | Garage (S3 API, self-host) |
| Migration | goose |

### Cluster

| Node | IP | Role |
|------|-----|------|
| k3s-node1 | 192.168.100.201 | control-plane, etcd |
| k3s-node2 | 192.168.100.202 | control-plane, etcd |
| k3s-node3 | 192.168.100.203 | control-plane, etcd |

K3s v1.35.5, ArgoCD GitOps, Traefik ingress, Cloudflare Tunnel.

---

## 7 Services

| Service | Module | Trách nhiệm |
|---------|--------|-------------|
| **core** | `app/core/` | Auth + User + Game (Phase 1 monolith) |
| **content** | `app/content/` | Courses, units, lessons, vocabulary |
| **grading** | `app/grading/` | Chấm dictation, listening; lịch sử làm bài |
| **user** | `app/user/` | Profile, avatar, settings, progress |
| **game** | `app/game/` | XP, level, leaderboard, quests, shop |
| **media** | `app/media/` | Upload, resize ảnh, transcode audio |
| **notification** | `app/notification/` | Push notification, email |

---

## Deployed Infrastructure

| Service | Chart | Namespace |
|---------|-------|-----------|
| PostgreSQL 18 | CloudNativePG CRD | puchi-db |
| Supertoken | supertokens/helm | puchi-infra |
| Envoy Gateway | envoy/gateway-helm | envoy-gateway-system |
| NATS | nats/helm | platform (shared) |
| Garage | custom Helm | platform (shared) |
| Valkey | bitnami/valkey | platform (shared) |

---

## Event-driven (NATS)

```
Grading ──► "lesson.completed" ──► Game (+XP), User (progress), Notification (push)
Game     ──► "level.up"          ──► User (unlock), Content (gợi ý bài), Notification (push)
```

Subject prefix: `puchi.*`

---

## Lộ trình triển khai

| Phase | Thời gian | Mô tả |
|-------|-----------|-------|
| 1 | 1-2 tháng | Monolith modular: core (auth+user+game) + content + media |
| 2 | tháng 3-4 | Tách auth + grading service, NATS event-driven |
| 3 | tháng 5-6 | Tách game + user + notification, Istio service mesh |

---

## Tài liệu tham khảo

- [Kratos](https://go-kratos.dev/) — Go microservice framework
- [Supertoken](https://supertokens.com/) — Open source auth
- [Envoy Gateway](https://gateway.envoyproxy.io/) — API Gateway
- [NATS](https://nats.io/) — Message broker
- [Garage](https://garagehq.deuxfleurs.fr/) — S3-compatible object storage
- [CloudNativePG](https://cloudnative-pg.io/) — PostgreSQL operator
