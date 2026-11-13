# Puchi

**Learn Vietnamese the way it's really spoken.**

Puchi is an app for learning Vietnamese, built for learners around the world. Every word and sentence comes in both Northern and Southern accents. Lessons are short, review is spaced, and stories come from everyday life in Vietnam.

🌏 [puchi.io.vn](https://puchi.io.vn)

---

## What we're building

| Part | Stack |
|---|---|
| Learner app (iOS, Android, web) | Flutter |
| Public website | Astro, on Cloudflare Workers |
| Content admin | React + shadcn/ui, behind Cloudflare Access |
| API | Go modular monolith: protobuf + connect-go on chi, Postgres, River, Valkey |
| Infrastructure | Kubernetes (k3s) with GitOps via Argo CD; Cloudflare Tunnel, R2 |

A single protobuf contract generates the Go server and the Dart and TypeScript clients, and `buf breaking` protects mobile apps that have already shipped.

## Repositories

Development happens in private repositories:

| Repo | Purpose |
|---|---|
| `puchi-api` | Backend and API contract (`proto/`) |
| `puchi-app` | Flutter learner app |
| `puchi-web` | Public website and SEO pages |
| `puchi-admin` | Content and operations admin |
| `puchi-design` | Brand, mascot, design tokens, illustrations |
| `puchi-gitops` | Kubernetes manifests and Argo CD apps |

---

<sub>Puchi Next — rebuilt from scratch in 2026. Only the name and the domain are carried over from the previous version.</sub>
