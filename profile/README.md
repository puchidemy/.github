![System Design](assets/system-design.png)

# Puchi

**Puchi** — nền tảng web học tiếng Việt.

Kiến trúc: **Next.js PWA (frontend) + Go microservices (backend)**, self-host trên K3s cluster, GitOps với ArgoCD.

---

## Repositories

| Repo | Mô tả |
|------|-------|
| [puchi-frontend](https://github.com/puchidemy/puchi-frontend) | Next.js 16 PWA (React 19, Tailwind v4, Shadcn UI, Bun) |
| [puchi-backend](https://github.com/puchidemy/puchi-backend) | Go monorepo — 5 services với Kratos v3 |
| [puchi-infra](https://github.com/puchidemy/puchi-infra) | Infrastructure as Code — ArgoCD GitOps trên K3s |

---

## Kiến trúc

```
User ──► Cloudflare CDN ──► Next.js PWA ──► Envoy Gateway ──► Go Services ──► PostgreSQL / NATS / R2
(puchi.io.vn)                                  (api.puchi.io.vn)   (Kratos v3)    (K3s + Cloudflare)
```

### Frontend

| Thành phần | Công nghệ |
|------------|-----------|
| Framework | Next.js 16 (App Router) + React 19 |
| Styling | Tailwind CSS v4 + Shadcn UI (New York) |
| State | Zustand v5 |
| Runtime | Bun (install) + Node.js (build) |
| i18n | next-intl (9 ngôn ngữ) |
| Animation | Motion + Rive |
| Auth | **Limen** — `limen-auth/react` + bearerPlugin |
| CI/CD | GitHub Actions → GHCR |

### Backend

| Thành phần | Công nghệ |
|------------|-----------|
| Ngôn ngữ | Go 1.26 |
| Framework | Kratos v3 (HTTP/gRPC hybrid, Protobuf-first, Wire DI) |
| Auth | Auth-service (**Limen**) — opaque session + Bearer |
| API Gateway | Envoy Gateway (CORS-only, Bearer passthrough) |
| Database | PostgreSQL 18 (CloudNativePG) |
| Cache | Valkey |
| Message Queue | NATS (event-driven) |
| Object Storage | Cloudflare R2 (S3 API) + CDN `cdn.puchi.io.vn` |
| Email | Tino SMTP via notification (`email.send`) |
| Migration | goose |

### Cluster

| Node | IP | Role |
|------|-----|------|
| k3s-node1 | 192.168.100.201 | control-plane, etcd |
| k3s-node2 | 192.168.100.202 | control-plane, etcd |
| k3s-node3 | 192.168.100.203 | control-plane, etcd |

K3s v1.35.5, ArgoCD GitOps, Traefik ingress, Cloudflare Tunnel.

---

## Authentication (Limen)

Auth qua [Limen](https://limenauth.dev/) — opaque session + Bearer (không JWT/JWKS).

```
Browser ──► puchi.io.vn (Next.js + limen-auth)
 │
 ├─► api.puchi.io.vn/auth/* → auth-service:8000 (Limen)
 │   ├── credential-password
 │   ├── oauth-google / oauth-facebook / oauth-tiktok
 │   └── session cookie + opaque Bearer
 │
 └─► api.puchi.io.vn/{v1/profile,v1/social,v1/learn,...} → Go services
     └── pkg/auth introspect → GET /internal/session
```

Auth methods: Email/Password + Google + Facebook + TikTok.

---

## 5 Services

| Service | Module | Trách nhiệm |
|---------|--------|-------------|
| **auth** | `app/auth/` | Identity (Limen): credential, OAuth, session |
| **core** | `app/core/` | Profile, onboarding, social, stats |
| **learn** | `app/learn/` | Curriculum, attempts, guest trial, grading |
| **media** | `app/media/` | Upload (Cloudflare R2) |
| **notification** | `app/notification/` | Prefs, Gotify push, email.send → Tino SMTP |

### Event-driven (NATS)

```
Auth   ──► "auth.user.created"      ──► Core (lazy profile)
Auth   ──► "email.send"             ──► Notification (Tino SMTP)
Learn  ──► "learn.lesson.completed" ──► Core (XP / streak)
Learn  ──► "learn.unit.completed"   ──► Core
```

---

## Domains

| Domain | Service |
|--------|---------|
| `puchi.io.vn` | Frontend |
| `api.puchi.io.vn` | API Gateway → Go services + auth |
| `cdn.puchi.io.vn` | R2 public CDN |
