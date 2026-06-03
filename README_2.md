# SEATU

> **Trust infrastructure for India's chit fund economy.**
> A mobile-first coordination platform for chit fund (Tamil: *seettu*) groups. Track contributions, run transparent auctions, build portable trust scores. **We coordinate; we do not handle money.**

---

[![Status](https://img.shields.io/badge/status-pre--launch-amber)]()
[![Stack](https://img.shields.io/badge/backend-NestJS%2010-red)]()
[![Stack](https://img.shields.io/badge/mobile-Flutter%203.22-blue)]()
[![Stack](https://img.shields.io/badge/db-PostgreSQL%2016-blue)]()
[![Region](https://img.shields.io/badge/region-ap--south--1-green)]()
[![Languages](https://img.shields.io/badge/i18n-Tamil%20%2B%20English-orange)]()

---

## Table of contents

1. [Overview](#1-overview)
2. [Project status](#2-project-status)
3. [Architecture at a glance](#3-architecture-at-a-glance)
4. [Tech stack](#4-tech-stack)
5. [Repository structure](#5-repository-structure)
6. [Prerequisites](#6-prerequisites)
7. [Quick start](#7-quick-start)
8. [Detailed development setup](#8-detailed-development-setup)
9. [Development workflow](#9-development-workflow)
10. [Configuration & credentials](#10-configuration--credentials)
11. [Testing](#11-testing)
12. [Deployment](#12-deployment)
13. [Hosting & infrastructure](#13-hosting--infrastructure)
14. [Monitoring & observability](#14-monitoring--observability)
15. [Operations & management](#15-operations--management)
16. [Security](#16-security)
17. [Troubleshooting](#17-troubleshooting)
18. [Roadmap](#18-roadmap)
19. [Contributing](#19-contributing)
20. [Legal & licensing](#20-legal--licensing)
21. [References & deeper documentation](#21-references--deeper-documentation)
22. [Contact](#22-contact)

---

## 1. Overview

### What SEATU is

SEATU digitises the operation of chit funds — a centuries-old group-savings instrument popular across India, especially Tamil Nadu, Andhra Pradesh, Karnataka, and Kerala. A typical chit fund has 10 members contributing ₹2,000/month for 10 months. Each month, one member wins the ₹20,000 pool through a sealed-bid auction. It's both a savings vehicle and a credit instrument, with no banks required.

But the trust infrastructure is paper-based. Records live in notebooks and WhatsApp groups. Foreman fraud and member defaults are common. There's no portable reputation when joining a new group.

SEATU is the coordination layer:

- **Every contribution is recorded** with screenshot evidence and member confirmation
- **Auctions run sealed-bid** with full history visible after close
- **A trust score follows the member** across rooms (Bronze → Silver → Gold → Platinum → Diamond)
- **Disputes have evidence trails** rather than he-said-she-said

### What SEATU is NOT

This is the most important design decision in the project: **money does not move through SEATU**. Members pay each other directly via UPI, bank transfer, or cash. We record evidence of those transactions; we don't facilitate them.

This "coordination only" posture (see [ADR-008](#4-non-negotiable-foundational-decisions)) lets us:
- Launch in 14–16 weeks instead of 18+ months
- Avoid Chit Funds Act 1982 licensing requirements
- Sidestep RBI Payment Aggregator regulations
- Operate without holding customer funds (no float, no settlement risk)

### Market

- **Registered chit fund market in India:** ₹35,000+ Cr annual turnover
- **Registered companies:** 30,000+
- **Subscriber base across South India:** several crore (~tens of millions)
- **Our launch market:** Tamil Nadu (Coimbatore + Chennai), Q3 2026

### Users

A 30-to-55-year-old chit fund member or foreman in Tamil Nadu. Smartphone-comfortable but not technical. May prefer Tamil over English. Has 0–5 active chit-fund participations.

---

## 2. Project status

**Pre-launch.** Built, tested locally, awaiting provider onboarding + legal sign-offs + first deploy.

| Area | Status | Detail |
|---|---|---|
| Product requirements | Done | `SEATU-PRD-Phase1.md` |
| System architecture | Done | 10 ADRs locked, `SEATU-Architecture-Phase2.md` |
| Database design | Done | 12 schemas, append-only triggers, seed script |
| API contract | Done | OpenAPI 3.1, 76 paths, 81 operations |
| Backend implementation | **Partial** | 2 of 10 modules fully built (IAM + Room); 8 skeletons with TODO stubs |
| Mobile UI | **Partial** | 2 of 6 features fully built (Auth + Rooms); 4 stub tabs |
| State management | Done | Cache, optimistic UI, lifecycle, offline banner |
| Testing | Done (critical paths) | 10 test files; critical optimistic-rollback test included |
| External adapters | Done | MSG91, Karza, FCM, Qwikcilver, Supabase wired with real impls |
| Deployment configs | Done | Railway, Render, GitHub Actions, smoke test |
| Legal templates | Done — **need lawyer review** | Privacy + ToS templates with India-specific markers |
| Provider onboarding | **Not started** | MSG91 DLT, Karza prod, FCM project, Qwikcilver |
| Domain + DNS | **Not started** | `api.seatu.in` to be set up |
| App Store submission | **Not started** | Android-first; iOS deferred |

---

## 3. Architecture at a glance

```mermaid
flowchart LR
    Users[Real users<br/>Android phones]

    subgraph Hosting[Railway / Render — asia-south1]
        API[NestJS API<br/>modular monolith]
    end

    subgraph Data[Managed data services]
        PG[(Supabase Postgres<br/>ap-south-1<br/>12 schemas)]
        Redis[(Upstash Redis<br/>ap-south-1)]
    end

    subgraph Providers[External providers]
        MSG91[MSG91<br/>SMS / OTP]
        Karza[Karza<br/>PAN + bank KYC]
        FCM[FCM<br/>push notifications]
        Qwik[Qwikcilver<br/>vouchers]
        Supa[Supabase Storage<br/>signed-URL uploads]
    end

    subgraph Observability[Observability]
        Sentry
        PostHog
    end

    Users -->|HTTPS REST + JWT| API
    API --> PG
    API --> Redis
    API --> MSG91
    API --> Karza
    API --> FCM
    API --> Qwik
    API --> Supa
    API -.->|errors| Sentry
    API -.->|events| PostHog
    Users -.->|crashes| Sentry
    Users -.->|events| PostHog
```

**Single-process modular monolith** with 11 modules (IAM, Room, Auction, Ledger, Trust, Gems, Dispute, Notification, Verification, Admin, plus shared infrastructure). Inter-module communication via in-process EventBus. Strict schema separation in Postgres preserves the future extraction path.

For deeper architecture documentation, see [`SEATU-Architecture-Phase2.md`](#21-references--deeper-documentation).

---

## 4. Tech stack

| Layer | Choice | Version | Why |
|---|---|---|---|
| Backend runtime | Node.js | 20 LTS | Stable, well-supported |
| Backend framework | NestJS | 10.x | Decorator-driven, modular, TypeScript-first |
| Language | TypeScript | 5.x strict | Type safety end-to-end |
| Database | PostgreSQL | 16 | Triggers, JSONB, mature multi-schema support |
| ORM | Prisma | 5.13 | Typed client, predictable migrations |
| Cache / queues | Redis | 7 (via ioredis) | Rate limiting, idempotency cache, outbox dispatch |
| Auth | JWT RS256 + opaque refresh | — | Standard mobile pattern |
| Logging | Pino | — | Fast structured logging with PII redaction |
| Mobile framework | Flutter | 3.22+ | One codebase, Material 3, strong i18n |
| State management | Riverpod | 2.x (codegen) | Compile-time safety, easy testing |
| Routing | GoRouter | — | Declarative, auth-aware |
| HTTP client | Dio | — | Interceptor-friendly |
| Local storage | flutter_secure_storage + shared_preferences | — | Tokens vs cache |
| Tests | Jest + Testcontainers (backend), flutter_test (mobile) | — | Standard for each platform |
| Region | asia-south1 (Mumbai/Singapore) | — | Closest to user base |
| API host | Railway or Render | — | PaaS, region available |
| Postgres host | Supabase | — | Managed, ap-south-1 available |
| Redis host | Upstash | — | Managed, ap-south-1 available |
| Errors | Sentry | — | Free tier sufficient for launch |
| Analytics | PostHog | — | Free tier sufficient for launch |

---

## 5. Repository structure

The project has three repos that work together. They live as sibling folders:

```
seatu/
├── seatu-backend/          # NestJS API
├── seatu-app/              # Flutter mobile app
└── seatu-deploy-kit/       # Launch runbook + legal + store-listing copy
```

### `seatu-backend/`

```
seatu-backend/
├── src/
│   ├── main.ts                              # App bootstrap
│   ├── app.module.ts                        # Root module
│   ├── openapi.yaml                         # API contract (2,274 lines)
│   ├── config/
│   │   └── env.config.ts                    # Zod-validated env schema
│   ├── shared/                              # Cross-cutting infrastructure
│   │   ├── prisma/                          # Database client + ALS context
│   │   ├── errors/                          # Error codes + global filter
│   │   ├── logging/                         # Pino + redaction
│   │   ├── auth/                            # JWT strategy + guards
│   │   ├── authz/                           # Policy guard (5 gates)
│   │   ├── audit/                           # Tx-aware audit service
│   │   ├── eventbus/                        # In-process events + outbox
│   │   ├── ratelimit/                       # Redis sliding-window
│   │   ├── pagination/                      # Cursor pagination helpers
│   │   └── idempotency/                     # Idempotency-Key interceptor
│   ├── adapters/                            # External service wrappers
│   │   ├── adapters.module.ts               # Env-driven Fake-vs-Real factory
│   │   ├── kyc/        kyc.adapter.ts       # Karza (real) + Fake
│   │   ├── sms/        sms.adapter.ts       # MSG91 (real) + Fake
│   │   ├── push/       push.adapter.ts      # FCM (real) + Fake
│   │   ├── voucher/    voucher.adapter.ts   # Qwikcilver (real) + Fake
│   │   └── storage/    storage.adapter.ts   # Supabase (real) + Fake
│   └── modules/
│       ├── iam/                             # ✅ FULLY IMPLEMENTED
│       ├── room/                            # ✅ FULLY IMPLEMENTED
│       ├── verification/                    # ⬜ Skeleton
│       ├── auction/                         # ⬜ Skeleton
│       ├── ledger/                          # ⬜ Skeleton
│       ├── trust/                           # ⬜ Skeleton
│       ├── gems/                            # ⬜ Skeleton
│       ├── dispute/                         # ⬜ Skeleton
│       ├── notification/                    # ⬜ Skeleton
│       └── admin/                           # ⬜ Skeleton
├── prisma/
│   ├── schema.prisma                        # 924-line schema, 12 schemas
│   ├── migrations/                          # SQL migrations
│   └── seed.ts                              # 10 users, 5 rooms
├── test/
│   ├── helpers/                             # Testcontainers bootstrap
│   ├── unit/                                # Pure-logic tests
│   └── integration/
│       └── onboarding.spec.ts               # Full auth flow E2E
├── scripts/
│   └── smoke.ts                             # Smoke test against deployed env
├── .github/
│   └── workflows/
│       ├── ci.yml                           # Lint + tests on every PR
│       └── deploy.yml                       # Deploy on push to main
├── docker-compose.yml                       # Local Postgres + Redis
├── Dockerfile                               # Production image
├── railway.json                             # Railway deploy config
├── render.yaml                              # Render deploy config
├── package.json
├── .env.example                             # Public-safe defaults
└── README.md
```

### `seatu-app/`

```
seatu-app/
├── lib/
│   ├── main.dart                            # App bootstrap (Sentry + PostHog init)
│   ├── app/
│   │   ├── app.dart                         # MaterialApp + offline banner wrapper
│   │   ├── router.dart                      # GoRouter with auth-aware redirects
│   │   └── theme.dart                       # Material 3 with deep teal seed
│   ├── core/                                # Cross-cutting infrastructure
│   │   ├── api/                             # Dio client + 3 interceptors
│   │   ├── auth/                            # Token storage + device identity
│   │   ├── cache/                           # Stale-while-revalidate cache
│   │   ├── connectivity/                    # Online/offline reactive provider
│   │   ├── lifecycle/                       # App foreground/background
│   │   ├── analytics/                       # PostHog + Sentry breadcrumbs
│   │   ├── error/                           # Error code → UX mapping
│   │   ├── i18n/                            # Tamil + English strings
│   │   ├── widgets/                         # Tier badge, offline banner, etc.
│   │   └── utils/                           # Formatters, validators
│   ├── features/
│   │   ├── auth/                            # ✅ FULLY IMPLEMENTED
│   │   ├── rooms/                           # ✅ FULLY IMPLEMENTED
│   │   ├── home/                            # Bottom nav scaffold
│   │   ├── profile/                         # /me + sign out
│   │   ├── trust/                           # ⬜ Stub
│   │   ├── gems/                            # ⬜ Stub
│   │   ├── auctions/                        # ⬜ Stub
│   │   └── notifications/                   # ⬜ Stub
│   └── shared/                              # Cross-feature models + providers
├── test/
│   ├── helpers/                             # FakeDioAdapter + buildTestApp
│   ├── unit/                                # Formatters, validators
│   ├── widget/                              # Auth + rooms widget tests
│   ├── golden/                              # Tier badge visual regression
│   └── integration/                         # Cache + optimistic rollback
├── env/
│   ├── dev.json                             # Local dev (no Sentry/PostHog)
│   ├── staging.json                         # Staging (with keys)
│   └── production.example.json              # Template; real one gitignored
├── android/                                 # Android-specific config
├── ios/                                     # iOS-specific config (deferred for v1)
├── pubspec.yaml
└── README.md
```

### `seatu-deploy-kit/`

```
seatu-deploy-kit/
├── SEATU-Launch-Runbook.md                  # Procedural launch document
├── legal/
│   ├── privacy-policy.template.md           # DPDP-aware, [LAWYER] markers
│   └── terms-of-service.template.md         # Chit Funds Act-aware
├── store-listing/
│   └── store-copy.md                        # Play + App Store, EN + TA
└── README.md
```

---

## 6. Prerequisites

You need the following installed on your development machine:

| Tool | Version | Purpose | Install |
|---|---|---|---|
| Node.js | 20 LTS | Backend runtime | [nvm](https://github.com/nvm-sh/nvm) recommended |
| pnpm | 9.x | Package manager | `npm install -g pnpm@9` |
| Docker | 24+ | Local Postgres + Redis | [docker.com](https://docker.com) |
| Flutter SDK | 3.22+ stable | Mobile framework | [flutter.dev/get-started](https://flutter.dev) |
| Android Studio | latest | Android emulator + SDK | [developer.android.com](https://developer.android.com/studio) |
| Git | 2.30+ | Version control | system package manager |

**Operating systems tested:** macOS (Apple Silicon + Intel), Ubuntu 22.04/24.04, Windows 11 with WSL 2.

For detailed install instructions per OS, see [`SEATU-Developer-Walkthrough.md`](#21-references--deeper-documentation).

---

## 7. Quick start

After [prerequisites](#6-prerequisites) are installed:

```bash
# 1. Get the code (assuming you've extracted the tarballs to ~/seatu/)
cd ~/seatu

# 2. Boot the backend
cd seatu-backend
docker compose up -d            # Postgres + Redis in containers
cp .env.example .env
pnpm install
pnpm prisma:generate
pnpm prisma:migrate:dev
pnpm db:seed                    # Demo users + rooms
pnpm start:dev                  # API now at http://localhost:3000

# 3. In a new terminal: boot the Flutter app
cd ~/seatu/seatu-app
flutter pub get
dart run build_runner build --delete-conflicting-outputs
flutter emulators --launch Pixel_7_API_34   # or your emulator's ID
flutter run --dart-define-from-file=env/dev.json
```

**Sign in with:** any seeded phone (e.g., `+919000000001`), OTP `123456` (dev bypass).

You should see the home screen with three seeded rooms. Tap **+ Create** to make a new one.

---

## 8. Detailed development setup

### 8.1 Backend setup

```bash
cd seatu-backend
docker compose up -d
```

Verify both containers are healthy:

```bash
docker compose ps
# seatu-postgres   Up (healthy)   0.0.0.0:5432->5432/tcp
# seatu-redis      Up (healthy)   0.0.0.0:6379->6379/tcp
```

Copy the example env file:

```bash
cp .env.example .env
```

Install Node dependencies (first run takes ~2 minutes):

```bash
pnpm install
```

Generate the Prisma client:

```bash
pnpm prisma:generate
```

Run migrations against your local Postgres:

```bash
pnpm prisma:migrate:dev
```

This creates 12 schemas and applies the append-only triggers on `ledger.ledger_events` and `audit.audit_log`.

Seed demo data:

```bash
pnpm db:seed
```

Start the API in watch mode:

```bash
pnpm start:dev
```

Visit:
- `http://localhost:3000/healthz` → `{"status":"ok"}`
- `http://localhost:3000/readyz` → `{"status":"ready"}` (checks DB)
- `http://localhost:3000/v1/openapi.yaml` → API contract

### 8.2 Flutter app setup

```bash
cd seatu-app
flutter pub get
dart run build_runner build --delete-conflicting-outputs
```

`build_runner` generates `*.freezed.dart`, `*.g.dart`, and `*.providers.g.dart` files for every model, JSON-serializable class, and Riverpod provider. First run is slow (~30s); incremental builds are fast.

For development with hot reload, run `build_runner` in watch mode in a separate terminal:

```bash
dart run build_runner watch --delete-conflicting-outputs
```

Launch the emulator:

```bash
flutter emulators
# 1 available emulator:
# Pixel_7_API_34 • Pixel 7 API 34 • Google • android

flutter emulators --launch Pixel_7_API_34
```

Run the app pointed at your local backend:

```bash
flutter run --dart-define-from-file=env/dev.json
```

The `dev.json` file sets `SEATU_API_BASE=http://10.0.2.2:3000` (the special Android-emulator hostname for the host machine).

### 8.3 Useful commands

| Backend | Effect |
|---|---|
| `pnpm start:dev` | Hot-reload dev server |
| `pnpm build` | Production bundle to `dist/` |
| `pnpm test` | All tests |
| `pnpm test:unit` | Just unit tests |
| `pnpm test:integration` | Integration tests (Testcontainers) |
| `pnpm prisma:studio` | DB GUI in browser |
| `pnpm prisma:migrate:dev` | Apply new migrations |
| `pnpm prisma:migrate:reset` | Drop DB, re-migrate, re-seed |
| `pnpm db:seed` | Re-run seed script |
| `pnpm lint` | ESLint |
| `pnpm typecheck` | TypeScript-only type check |

| Flutter | Effect |
|---|---|
| `flutter run` | Build + run on connected device |
| `flutter test` | All tests |
| `flutter test test/integration/` | Specific subdir |
| `flutter test --update-goldens` | Regenerate golden images |
| `flutter analyze` | Static analysis |
| `flutter pub get` | Install deps |
| `dart run build_runner watch` | Codegen on save |
| `flutter build apk --release` | Android APK |
| `flutter build appbundle --release` | Play Store bundle |

---

## 9. Development workflow

### 9.1 Adding a backend feature

Follow the IAM module as the reference. The pattern:

1. **Define the API contract** in `src/openapi.yaml` first.
2. **Add or update Prisma models** if data changes; run `pnpm prisma:migrate:dev`.
3. **Generate or update DTOs** under `src/modules/<feature>/dto/`.
4. **Add a service** with the business logic. Use transactions for any write. Inject `PrismaService`, `AuditService`, `EventBus`, and any adapters you need.
5. **Add a controller** that calls the service. Apply the right guards (`@Auth()`, `@RequireRole()`, `@RequireVerification()`, `@RequireTier()`, `@RequireRoomMember()` etc.).
6. **Define any new events** under `src/modules/<feature>/events/`.
7. **Write an integration test** in `test/integration/<feature>.spec.ts`.

A minimal backend service method looks like this:

```typescript
async publishRoom(roomId: string, userId: string): Promise<Room> {
  return this.prisma.$transaction(async (tx) => {
    const room = await tx.room.findUniqueOrThrow({ where: { id: roomId } });
    this.assertCanTransition(room.status, 'recruiting');

    const updated = await tx.room.update({
      where: { id: roomId },
      data: { status: 'recruiting' },
    });

    await this.auditService.log(tx, {
      actorUserId: userId,
      action: 'room.published',
      entityType: 'room',
      entityId: roomId,
    });

    await this.eventBus.publish(tx, new RoomPublishedEvent(roomId));

    return updated;
  });
}
```

The `EventBus.publish(tx, ...)` writes to the `outbox_events` table in the same transaction. The `OutboxDispatcher` worker reads pending events and fires side effects — at-least-once delivery.

### 9.2 Adding a Flutter feature

Follow the auth or rooms feature as the reference. The pattern:

1. **Create the feature directory** `lib/features/<feature>/` with subdirs `data/`, `domain/`, `application/`, `presentation/`.
2. **Define the domain model** in `domain/<entity>.dart` using freezed.
3. **Add the API client** in `data/<feature>_api.dart` using Dio.
4. **Add Riverpod providers / controllers** in `application/<feature>_controller.dart`.
5. **Build the screens** in `presentation/screens/<screen>.dart`.
6. **Add routes** to `lib/app/router.dart`.
7. **Run codegen** if you added freezed models or `@riverpod` providers:

```bash
dart run build_runner build --delete-conflicting-outputs
```

8. **Hot reload** with `r` in the running `flutter run` terminal.

Imports flow strictly downward: `presentation/` → `application/` → `domain/` → `data/`. Never upward.

### 9.3 Code conventions

| Convention | Backend | Frontend |
|---|---|---|
| File naming | `kebab-case.ts` | `snake_case.dart` |
| Class naming | `PascalCase` | `PascalCase` |
| Variable naming | `camelCase` | `camelCase` |
| Constant naming | `UPPER_SNAKE_CASE` | `UPPER_SNAKE_CASE` |
| Imports | absolute from `src/` | absolute from `package:seatu/` |
| Money | `Int` paise, never floats | `Int` paise, format on display |
| Timestamps | `Date` (UTC); ISO 8601 with offset on wire | `DateTime.toUtc()`; format with locale |
| Async | `async/await`, no `.then()` chains | `async/await`, no `.then()` chains |
| Error handling | `throw new DomainException(ErrorCode.X)` | `catch (e) { mapToUserMessage(e); }` |

### 9.4 Git workflow

```
main                  # protected, deploys to staging on push
  ├── feat/auction-bid-placement
  ├── feat/kyc-screens
  ├── fix/cache-stale-on-resume
  └── ...
```

- Branch from `main`
- One feature per branch
- Open a PR; CI runs (lint + typecheck + tests)
- Squash merge to `main`
- `main` deploys to staging automatically
- Manual promotion to production via `workflow_dispatch`

Commit message format:

```
feat(rooms): add optimistic publish rollback test

The Phase 7 optimistic UI pattern needs verification that the
rollback path works on failure. This test simulates a 409 response
and asserts both AsyncError state and snapshot restoration.
```

---

## 10. Configuration & credentials

All configuration is via environment variables. `src/config/env.config.ts` validates them at boot with Zod — if a required var is missing or malformed, the app refuses to start.

### 10.1 Backend env vars

Grouped by tier:

#### Tier 0 — without these the app cannot boot

| Variable | Example | Notes |
|---|---|---|
| `NODE_ENV` | `development` / `staging` / `production` | |
| `PORT` | `3000` | |
| `DATABASE_URL` | `postgresql://user:pass@host:5432/db?schema=public` | Use connection pooler in production |
| `REDIS_URL` | `rediss://...:6380` | Upstash URL with TLS |
| `JWT_ALGORITHM` | `RS256` | Locked; do not change |
| `JWT_PRIVATE_KEY` | (RSA 2048 PEM, newlines as `\n`) | `openssl genrsa -out jwt-priv.pem 2048` |
| `JWT_PUBLIC_KEY` | (RSA 2048 PEM, newlines as `\n`) | `openssl rsa -in jwt-priv.pem -pubout` |
| `JWT_ISSUER` | `https://api.seatu.in` | |
| `JWT_ACCESS_TTL_SECONDS` | `900` | 15 minutes |
| `JWT_REFRESH_TTL_SECONDS` | `2592000` | 30 days |
| `APP_ENCRYPTION_KEY` | (32-byte base64) | `openssl rand -base64 32` |

#### Tier 1 — key features degrade without these

| Variable | Example | Notes |
|---|---|---|
| `SMS_PROVIDER` | `msg91` / `fake` | `fake` in dev, `msg91` in prod |
| `MSG91_AUTH_KEY` | (from MSG91 dashboard) | |
| `MSG91_OTP_TEMPLATE_ID` | (DLT-approved template) | Required for OTPs |
| `MSG91_SENDER_ID` | `SEATUI` | 6-char registered header |
| `MSG91_TPL_MEMBER_JOINED` | (DLT-approved template) | |
| `MSG91_TPL_CONTRIB_REMINDER` | (DLT-approved template) | |
| `MSG91_TPL_AUCTION_OPENED` | (DLT-approved template) | |
| `MSG91_TPL_PAYOUT_DUE` | (DLT-approved template) | |
| `KYC_PROVIDER` | `karza` / `fake` | |
| `KARZA_API_KEY` | (from Karza dashboard) | |
| `KARZA_BASE_URL` | `https://api.karza.in` | |

#### Tier 2 — full feature parity needs these

| Variable | Example | Notes |
|---|---|---|
| `PUSH_PROVIDER` | `fcm` / `fake` | |
| `FCM_SERVICE_ACCOUNT_JSON` | (full JSON, single-line) | Firebase service account |
| `STORAGE_PROVIDER` | `supabase` / `fake` | |
| `SUPABASE_URL` | `https://xxx.supabase.co` | |
| `SUPABASE_SERVICE_KEY` | (service role key) | Server-only; never expose to client |
| `SUPABASE_BUCKET` | `seatu-uploads` | Private bucket |
| `VOUCHER_PROVIDER` | `qwikcilver` / `fake` | |
| `QWIKCILVER_API_KEY` | (from QC dashboard) | |
| `QWIKCILVER_BASE_URL` | `https://api.qwikcilver.com` | |
| `QC_SKU_AMAZON_500` | (partner SKU code) | |
| `QC_SKU_FLIPKART_500` | (partner SKU code) | |
| `QC_SKU_BOOKMYSHOW_500` | (partner SKU code) | |

#### Tier 3 — observability, set before taking traffic

| Variable | Example |
|---|---|
| `SENTRY_DSN` | `https://xxx@xxx.ingest.sentry.io/xxx` |
| `POSTHOG_API_KEY` | (from PostHog dashboard) |
| `LOG_LEVEL` | `info` (prod) / `debug` (dev) |

#### Development conveniences

| Variable | Example | Notes |
|---|---|---|
| `DEV_OTP_BYPASS` | `123456` | Set in dev only. Lets you sign in without real SMS. |
| `DISABLE_RATE_LIMIT` | `true` | Dev only. |

### 10.2 Flutter env vars

Flutter reads env at build time via `--dart-define-from-file=env/<env>.json`. Three files:

**`env/dev.json`** (committed):

```json
{
  "SEATU_ENV": "dev",
  "SEATU_API_BASE": "http://10.0.2.2:3000",
  "SENTRY_DSN": "",
  "POSTHOG_API_KEY": ""
}
```

**`env/staging.json`** (committed; staging keys are not secret):

```json
{
  "SEATU_ENV": "staging",
  "SEATU_API_BASE": "https://api-staging.seatu.in",
  "SENTRY_DSN": "REPLACE_WITH_STAGING_SENTRY_DSN",
  "POSTHOG_API_KEY": "REPLACE_WITH_STAGING_POSTHOG_KEY"
}
```

**`env/production.json`** (gitignored; only on CI / release machines):

```json
{
  "SEATU_ENV": "production",
  "SEATU_API_BASE": "https://api.seatu.in",
  "SENTRY_DSN": "real-dsn-here",
  "POSTHOG_API_KEY": "real-key-here"
}
```

### 10.3 Secrets management

**Never commit real secrets.** The `.env` and `env/production.json` files are gitignored.

**Recommended approach:**

1. **1Password** (or Bitwarden) — store all real secrets in a shared vault. One entry per environment.
2. **GitHub Secrets** — for CI/CD. Each Tier 0–2 variable becomes a separate secret in the repo settings.
3. **Hosting provider dashboard** — Railway/Render let you set env vars per service. CI pushes the values; the dashboard is the runtime source.

**Rotation cadence:**

| Secret type | Rotation |
|---|---|
| `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` | Annually, or immediately on compromise. Plan a key-rotation script that issues new keys while old refresh tokens still validate against the old public key for 7 days. |
| `APP_ENCRYPTION_KEY` | Annually. Migration needed for encrypted records in DB. |
| Provider API keys (MSG91, Karza, etc.) | Annually, or whenever provider mandates |
| Database credentials | Quarterly |
| Sentry / PostHog | Only when compromised |

### 10.4 Local development credentials

For local development, the `.env.example` file you copy to `.env` sets every provider to `fake`. No real keys needed.

Sign in with:

| Phone | Name | Tier | OTP |
|---|---|---|---|
| `+919000000001` | Lakshmi | gold | `123456` |
| `+919000000002` | Karthik | silver | `123456` |
| `+919000000003` | Rekha | bronze | `123456` |
| `+919000000004` | Murugan | silver | `123456` |
| `+919000000005` | Anitha | bronze | `123456` |
| `+919000000006` | Suresh | platinum | `123456` |
| `+919000000007` | Diana | bronze | `123456` |
| `+919000000008` | Vimal | bronze | `123456` |
| `+919000000009` | Founder | diamond | `123456` |
| `+919000000010` | Ops | diamond | `123456` |

These are populated by `prisma/seed.ts`. To reset: `pnpm prisma:migrate:reset` (drops everything and re-seeds).

---

## 11. Testing

### 11.1 Backend tests

**Unit tests** — pure logic, no DB or network. Run in <1 second:

```bash
pnpm test:unit
```

**Integration tests** — uses Testcontainers (real Postgres + Redis in throwaway Docker containers). First run downloads images (~30s); subsequent runs are faster:

```bash
pnpm test:integration
```

What's covered today:

- `test/integration/onboarding.spec.ts` — OTP request → verify → /me → refresh → /me with rotated token; also asserts rejection of unauthenticated requests and missing `Idempotency-Key` on mutating endpoints.

**Outstanding** (to be added with the respective skeleton modules):

- `room-lifecycle.spec.ts` — create → publish → join (×5) → start → cycle → complete
- `auction-cycle.spec.ts` — open → bid (×4) → close → tie-break → payout
- `contribution-dispute.spec.ts` — mark-paid → confirm → dispute → respond → decide

### 11.2 Frontend tests

```bash
flutter test                  # All
flutter test test/widget/     # Just widget tests
flutter test test/golden/     # Visual regression
```

Test categories:

- **`test/unit/`** — pure logic (formatters, validators)
- **`test/widget/`** — UI smoke tests using `FakeDioAdapter` + provider overrides
- **`test/golden/`** — visual regression (currently just `tier_badge`)
- **`test/integration/`** — multi-component (cache box, optimistic rollback)

**The most important test is `test/integration/rooms_controller_test.dart`** — it verifies the Phase 7 optimistic-publish pattern rolls back correctly on failure. If only one test in the project survives, that's the one.

To regenerate goldens after an intentional visual change:

```bash
flutter test --update-goldens test/golden/
```

### 11.3 Smoke tests (against deployed environments)

```bash
SEATU_API_BASE=https://api-staging.seatu.in \
SMOKE_PHONE=+919900000099 \
ALLOW_OTP_BYPASS=true \
tsx scripts/smoke.ts
```

Covers: `/healthz`, `/readyz`, `/v1/openapi.yaml`, OTP request, OTP verify with dev bypass.

This script runs automatically in the deploy workflow against the new environment after each deploy. Manual runs are useful during incidents.

### 11.4 What we deliberately don't test

For honesty (and so a reviewer doesn't audit it as missing):

- Refresh-on-401 interceptor full chain (smoke-tested in staging instead)
- Router redirect logic (covered implicitly by widget tests)
- Theme tokens (visual inspection)
- Connectivity plugin (it's a vendor plugin)
- Performance (no perf tests in v1; profile when problem observed)
- A11y compliance (manual audit before launch)

---

## 12. Deployment

### 12.1 Environments

| Environment | Purpose | URL | Trigger |
|---|---|---|---|
| Local | Developer machine | `http://localhost:3000` | `pnpm start:dev` |
| Staging | Pre-prod testing | `https://api-staging.seatu.in` | Auto-deploy on push to `main` |
| Production | Real users | `https://api.seatu.in` | Manual `workflow_dispatch` |

### 12.2 CI/CD pipeline

On every push to a feature branch:

1. **Lint** (`pnpm lint`, `flutter analyze`)
2. **Type check** (`pnpm typecheck`)
3. **Unit tests** + integration tests with Testcontainers
4. **Build** (verify it compiles)
5. **Status reported on PR**

On push to `main`:

1. CI runs as above
2. If green, **deploy workflow** triggers automatically:
   - Run Prisma migrations on staging DB
   - Build Docker image
   - Push to Railway/Render
   - Wait for healthcheck
   - Run smoke test
   - Slack notification on failure

On manual `workflow_dispatch` (production):

1. Same as staging, but against production env
2. **Pre-flight check**: smoke must pass on staging first
3. Migrations applied
4. Deploy
5. Smoke against production
6. Slack notification

### 12.3 First-time production deploy

Follow `seatu-deploy-kit/SEATU-Launch-Runbook.md`. The runbook has four streams:

1. **Infrastructure** — provision Postgres, Redis, API host, DNS
2. **Provider setup** — MSG91 DLT, Karza, FCM, Qwikcilver, Supabase
3. **Flutter builds** — keystore, Play Store internal track
4. **Compliance** — lawyer-reviewed legal pages live, Chit Funds Act opinion on file

**Stream 4 GATES go-live.** Do not promote to public production without it.

### 12.4 Deploy commands (manual fallback)

If the CI pipeline is broken and you need to deploy from your laptop:

**Railway:**

```bash
railway login
railway link --project <project-id>
railway up --detach
```

**Render:**

Use the dashboard — trigger a manual deploy from the latest commit on `main`.

### 12.5 Rollback

**Backend (Railway):**

```bash
railway rollback --service seatu-api
```

**Backend (Render):**

Dashboard → Deploys → click "Rollback" on the last green deploy.

**Mobile (Play Store):**

You cannot roll back. Options:

1. Pause the staged rollout (Play Console)
2. Ship a hotfix release
3. Server-side kill switch (if it's a feature-flag-able feature)

**Database:**

Append-only data (ledger, audit) cannot be rolled back at DB level — they're protected by triggers. To "undo" a bad write, insert a compensating event. Backups via Supabase: daily snapshots; restore via support ticket (2–4 hour SLA).

---

## 13. Hosting & infrastructure

### 13.1 Provider list

| Service | Provider | Plan | Region | Monthly cost (estimate) |
|---|---|---|---|---|
| API hosting | Railway (or Render) | Hobby → Pro | asia-south1 | $5–25 |
| Postgres | Supabase | Pro | ap-south-1 | $25 |
| Redis | Upstash | Pay-as-you-go | ap-south-1 | $0–10 (free tier covers MVP) |
| Object storage | Supabase Storage | Pro (bundled) | ap-south-1 | Bundled |
| Push | Firebase Cloud Messaging | Free (Spark) | Global | $0 |
| SMS | MSG91 | Pay-per-message | India | ~₹0.15/SMS (~$0.002) |
| KYC | Karza | Pay-per-call | India | ~₹3/PAN verify, ~₹5/bank verify |
| Vouchers | Qwikcilver | Revenue-share | India | ~3–5% take rate (we earn) |
| Errors | Sentry | Developer (free) | US | $0 (up to 5k events/mo) |
| Analytics | PostHog Cloud | Free | US | $0 (up to 1M events/mo) |
| Domain | (registrar of choice) | — | — | ~₹800/year for `.in` |
| DNS / CDN | Cloudflare | Free | Global | $0 |
| Email | SendGrid (or transactional API) | Free up to 100/day | Global | $0–15 |
| Slack | (for alerts) | Free | — | $0 |
| **Total monthly** (MVP scale) | | | | **~$50–100 + SMS/KYC variable** |

At 10k users: ~$200/month + variable. At 100k users: ~$1500/month.

### 13.2 Region choice

All compute and data lives in **ap-south-1** (Mumbai) or **asia-south1** (Mumbai for GCP, Singapore for Render).

Why:
- Users are in India; latency from US/EU is 200–300ms
- DPDP Act 2023 has data-residency implications (not strictly localisation, but cross-border transfer requires disclosure)
- Recovery / support is in business hours

Exception: Sentry and PostHog are US-hosted. Both are disclosed in the privacy policy as cross-border processors.

### 13.3 Scaling notes

The MVP architecture handles **~10,000 active users on a single API instance with a single Postgres**. Beyond that:

- **API**: scale horizontally on Railway/Render (stateless); add load balancer
- **Postgres**: add read replicas via Supabase (~$50/month each)
- **Redis**: Upstash scales transparently; pay-as-you-go
- **Outbox dispatcher**: extract to its own worker binary when push+SMS exceeds 100k events/day (Phase 9-equivalent work)

See [`SEATU-Architecture-Phase2.md`](#21-references--deeper-documentation) §15 for the long-term scaling plan.

### 13.4 Backups

- **Postgres**: Supabase Pro includes daily snapshots, 7-day retention. Configure point-in-time recovery for production.
- **Object storage**: Supabase Storage replication is automatic within region.
- **Application state**: stateless API; nothing to back up.
- **Test recovery quarterly**: practice restore on a non-production project. The restore button only matters if it works.

---

## 14. Monitoring & observability

### 14.1 Sentry (errors)

- Both backend and Flutter send errors here
- **Free tier**: 5k events/month is enough for MVP
- **Configure**: one Sentry project, two environments (`staging` + `production`)
- **DSN**: set as `SENTRY_DSN` env var on both sides
- **Sampling**: production samples at 10% for performance traces, 100% for errors

**Dashboards to set up:**

1. **Issues by environment** — production vs staging side-by-side
2. **Top errors last 24h** — for daily triage
3. **Affected users** — by error type, to prioritise

### 14.2 PostHog (product analytics)

- **Tracked events** (defined in `lib/core/analytics/analytics_service.dart`):
  - `app_opened`, `otp_requested`, `otp_verified`, `signed_out`
  - `room_created`, `room_published`, `room_started`, `room_cancelled`
  - `verification_started`, `verification_completed`
- **Free tier**: 1M events/month is enough for MVP

**Funnels to set up:**

1. **Onboarding funnel**: `app_opened` → `otp_requested` → `otp_verified` (look for the cliff)
2. **Activation funnel**: `otp_verified` → `room_created` → `room_published` (where do users stall?)
3. **KYC funnel**: `verification_started` → `verification_completed` (drop-off by step)

### 14.3 Logs

- **Backend logs** are structured JSON via Pino
- **Production**: Railway/Render captures stdout. Set `LOG_LEVEL=info`. Sample queries:
  - `grep "error" | jq '.error.code'` — distribution of error codes
  - `grep "req_id" | jq '.duration_ms'` — request latency outliers
- **PII redaction is on by default.** The Pino config redacts paths: `req.body.password`, `req.body.otp`, `req.headers.authorization`, plus full PAN/bank patterns. NEVER log them yourself either.

### 14.4 Status page

Recommend setting up a public status page once you have real users. Free options: [betterstack.com/status](https://betterstack.com), [instatus.com](https://instatus.com).

---

## 15. Operations & management

### 15.1 Database operations

**Browsing data locally:**

```bash
pnpm prisma:studio              # Opens GUI at http://localhost:5555
```

**Connecting to production Postgres:**

```bash
docker run --rm -it postgres:16-alpine psql "$PROD_DATABASE_URL"
```

**WARNING**: Never run UPDATE or DELETE on `ledger.ledger_events` or `audit.audit_log` — append-only triggers will reject the operation. For ledger corrections, write a compensating event instead.

**Common queries:**

```sql
-- Active user count (last 30 days)
SELECT count(distinct user_id)
FROM iam.devices
WHERE last_seen_at > now() - interval '30 days';

-- Rooms by status
SELECT status, count(*) FROM room.rooms GROUP BY 1 ORDER BY 2 DESC;

-- Tier distribution
SELECT tier, count(*) FROM trust.trust_scores_current GROUP BY 1;

-- Outbox backlog (notification.outbox_events should drain to 0)
SELECT status, count(*) FROM notification.outbox_events GROUP BY 1;
```

### 15.2 User account operations

These all require admin JWT audience (`aud=seatu-admin`), via admin endpoints (`/v1/admin/...`). The admin module is still a skeleton; for now, use direct SQL via Prisma Studio or psql.

**Common operations:**

| Task | How |
|---|---|
| Look up a user by phone | `SELECT * FROM iam.users WHERE phone = '+91...'` |
| Suspend a user | `UPDATE iam.users SET status = 'suspended' WHERE id = '...'` |
| Force-sign-out a user (all devices) | `UPDATE iam.refresh_tokens SET revoked_at = now() WHERE user_id = '...' AND revoked_at IS NULL` |
| Reset a user's trust score | Insert a compensating event in `trust.trust_events`; recompute the projection |
| Delete a user account (DPDP request) | Cascade is application-managed (we have no cross-schema FKs); follow the runbook §"User deletion" |

### 15.3 Feature flags

Not yet implemented. When needed, recommend [Unleash](https://www.getunleash.io/) (self-hosted) or [LaunchDarkly](https://launchdarkly.com/). For v1, feature toggles are env vars + restart.

### 15.4 On-call playbook

For incident response — symptoms, likely causes, first actions — see `seatu-deploy-kit/SEATU-Launch-Runbook.md` §"On-call playbook".

Highlights:

- **OTP not arriving**: MSG91 DLT template flagged / wallet empty / authkey rotated
- **5xx errors spike**: Postgres saturated — find the long-running query via `pg_stat_activity`
- **Push notifications not delivering**: FCM token churn / service account key rotated
- **Sign-in loop**: refresh token rotation broke / device fingerprint mismatch

---

## 16. Security

### 16.1 What's in place

| Mitigation | Where |
|---|---|
| TLS 1.2+ everywhere | Hosting provider auto-managed |
| JWT RS256 with short access TTL (15 min) | `src/shared/auth/` |
| Opaque rotating refresh tokens | `iam.refresh_tokens` table |
| Family revocation on reuse detection | Critical — defends against token theft |
| Device binding via `X-Device-Id` | Every request validated |
| Per-route rate limits (Redis sliding window) | `src/shared/ratelimit/` |
| Idempotency keys mandatory on writes | Prevents replay attacks |
| PII minimisation (PAN/bank last4 + score only) | `verification.verification_results` table |
| Encryption at rest for voucher codes (AES-256-GCM) | `gems.redemptions.encrypted_code` |
| Encryption at rest for DB | Supabase managed |
| Append-only triggers on ledger + audit | `prisma/migrations/00000000000001_append_only_triggers/` |
| Audit log of every state-changing operation | `audit.audit_log` |
| PII redaction in logs | Pino config |
| Cross-schema FK ban (extraction path preservation) | Per ADR-002 |
| Pre-signed URL uploads (server never sees bytes) | Storage adapter |
| Webhook signature verification | For Karza callbacks if used |
| CSP, HSTS, X-Frame-Options | NestJS Helmet middleware |

### 16.2 Things to be careful about

- **Never log tokens, OTPs, full PAN, or bank account numbers.** Pino redaction is on, but a `console.log` you add bypasses it.
- **Never store full PAN or bank account numbers.** The KYC adapter returns last4 + score; the service must never persist the full input. This is enforced by code review and tested.
- **Refresh-token reuse detection is not optional.** If you weaken it for UX reasons, you've undermined the device-theft defence.
- **Admin endpoints require admin JWT audience**, not just `role='admin'`. Don't downgrade this.
- **All inputs go through class-validator DTOs.** Don't bypass with `@Body() body: any`.

### 16.3 Reporting security issues

If you find a security vulnerability:

1. **Do NOT open a public GitHub issue.**
2. Email `[security@seatu.in]` with the details.
3. Expected acknowledgement: within 48 hours.
4. We'll coordinate disclosure after a fix is deployed.

---

## 17. Troubleshooting

The most common issues, in rough order of frequency:

### "Cannot connect to the Docker daemon"

Docker Desktop isn't running. Open it; wait for the whale icon.

### "Port 5432 is already allocated"

Another Postgres is running. Stop it (`brew services stop postgresql` / `sudo systemctl stop postgresql`) or change the port in `docker-compose.yml` and `.env`.

### Prisma can't reach the database

Three possibilities:
1. Docker not running
2. Wrong `DATABASE_URL` in `.env`
3. Container starting — `docker compose logs postgres`

### `EADDRINUSE: address already in use :::3000`

API already running in another terminal. Kill it (`lsof -i :3000`) or change `PORT` in `.env`.

### Flutter: "No connected devices"

Emulator not running. `flutter emulators --launch Pixel_7_API_34`. Wait 30 seconds. Retry `flutter run`.

### Flutter app: "No internet connection" banner appears immediately

App can't reach the backend.
- Backend running? `curl localhost:3000/healthz`
- Using emulator? `SEATU_API_BASE=http://10.0.2.2:3000` (the emulator's host alias)
- Using real device? Use your LAN IP + firewall must allow it

### `AUTH_TOKEN_REUSE_DETECTED` on second refresh

You used the same refresh token twice. Family-revocation kicked in. Sign out, sign back in.

### Build_runner errors

Run again with `--delete-conflicting-outputs`. If still failing, look for a syntax error in a `.dart` file — the message usually tells you which.

### OTP from logs doesn't work

In dev, use the hardcoded bypass `123456` (set via `DEV_OTP_BYPASS=123456`). The OTP shown in logs is the real one, but `123456` always works.

### "AUTH_DEVICE_MISMATCH" after re-installing the app

Device identity regenerates on fresh install. Backend sees a new device. Force a fresh sign-in.

For deployment-specific issues, see the **rollback** section above and the launch runbook's incident playbook.

---

## 18. Roadmap

### Immediate (next 30 days)

The backend Phase 5.1 and mobile Phase 6.1 work. Most-impactful first:

| Priority | Task | Owner | Effort |
|---|---|---|---|
| P0 | Verification module (PAN + bank) | Backend | 2 days |
| P0 | KYC screens (Flutter) | Mobile | 3 days |
| P0 | Auction module | Backend | 3 days |
| P0 | Auction screen (Flutter) | Mobile | 4 days |
| P1 | Contribution mark-paid + screenshot upload | Both | 4 days |
| P1 | Dispute file + respond | Both | 5 days |
| P2 | Gem redemption marketplace | Both | 3 days |
| P2 | Notification inbox + push handling | Both | 2 days |
| P3 | Backend integration tests for room/auction/dispute flows | Backend | 4 days |

### Year 1 (post-launch)

- iOS App Store build (only if Android validates the model in 3–6 months)
- Premium foreman subscription tier
- Trust tier detail screen
- Tamil-language professional copy review
- Real performance baseline + optimisation

### Year 2+

See `SEATU-Complete-Procedure.md` §1–§9 long-term sections. Highlights:

- Extract OutboxDispatcher to its own worker binary
- Read replicas
- WebSocket for notification inbox (if user demand)
- Enterprise SaaS for registered chit fund companies
- Expansion to Karnataka + Andhra (Kannada + Telugu)
- Eventually: trust-data-licensed financial products

---

## 19. Contributing

### 19.1 For internal team members

1. Pick an item from §18 roadmap
2. Branch from `main`: `git checkout -b feat/verification-pan-endpoint`
3. Implement following [§9 Development workflow](#9-development-workflow)
4. Write tests
5. Open a PR; assign reviewer
6. CI must be green
7. Squash merge to `main`

### 19.2 Code review standards

A PR must:

- Pass all CI checks (lint, typecheck, tests)
- Include tests for new logic
- Follow the code conventions in §9.3
- Not violate any of the 10 ADRs
- Have a clear description: what changed and why
- Not include secrets, generated files, or `.env` content
- Update docs if behaviour-changing

### 19.3 Outsiders

The repo is currently private. If we open-source any component in the future, we'll add a `CONTRIBUTING.md` with the public process.

---

## 20. Legal & licensing

### 20.1 Project ownership

SEATU is operated by **[FILL: registered entity name]**. All code, designs, and documentation are proprietary unless explicitly marked otherwise.

### 20.2 Third-party licenses

Dependencies are listed in `package.json` (backend) and `pubspec.yaml` (mobile). Their licenses (MIT, BSD, Apache 2) permit commercial use. Notable:

- NestJS — MIT
- Prisma — Apache 2
- Flutter / Dart — BSD 3-clause
- pptxgenjs — MIT
- All Riverpod / Dio / freezed packages — BSD or MIT

### 20.3 Compliance documents

- **Privacy policy**: `seatu-deploy-kit/legal/privacy-policy.template.md` — DPDP Act 2023-aware, needs lawyer review before publication.
- **Terms of service**: `seatu-deploy-kit/legal/terms-of-service.template.md` — Chit Funds Act 1982-aware, needs lawyer review.
- **Both files have `[LAWYER]` markers** indicating sections specifically requiring counsel input.

### 20.4 Important compliance notes

- **Chit Funds Act 1982**: SEATU's coordination-only posture (ADR-008) is the foundation of our regulatory framing. **Any feature that touches money flow triggers a fresh regulatory opinion.** Do not introduce one without counsel review.
- **DPDP Act 2023**: requires named grievance officer in privacy policy. Marked in the template.
- **TRAI / DLT**: all SMS templates must be DLT-registered. MSG91 will reject unregistered templates in production.
- **Play Store + App Store**: in-app account-deletion path is REQUIRED. Implement before submitting to stores.

---

## 21. References & deeper documentation

These documents go deeper into specific phases of the build:

| Document | Purpose | Length |
|---|---|---|
| `SEATU-PRD-Phase1.md` | Product requirements | 24 pages |
| `SEATU-Architecture-Phase2.md` | System architecture + 10 ADRs | ~20 pages |
| `SEATU-Database-Phase3.md` | Schema design rationale | ~20 pages |
| `SEATU-API-Design-Phase4.md` | API design decisions | ~20 pages |
| `SEATU-Backend-Phase5.md` | Backend implementation guide | ~20 pages |
| `SEATU-Flutter-Phase6.md` | Mobile architecture | ~15 pages |
| `SEATU-Flutter-Phase7.md` | State management deep-dive | ~10 pages |
| `SEATU-Flutter-Phase8.md` | Testing strategy | ~10 pages |
| `seatu-deploy-kit/SEATU-Launch-Runbook.md` | **The launch procedure** | ~15 pages |
| `SEATU-Complete-Procedure.md` | Strategic recap (short-term vs long-term across phases) | ~15 pages |
| `SEATU-Developer-Walkthrough.md` | Hands-on tutorial: blank machine to running system | ~15 pages |
| `SEATU-AI-Context-Primer.md` | Context bundle for asking AI assistants for help | ~15 pages |
| `SEATU-Investor-Pitch.pptx` | Seed-round pitch deck | 13 slides |

For day-to-day work, the most important are:

- **This README** for navigation
- **`SEATU-AI-Context-Primer.md`** for asking AI assistants for help
- **`seatu-deploy-kit/SEATU-Launch-Runbook.md`** when going live or troubleshooting prod
- **`SEATU-Developer-Walkthrough.md`** when onboarding a new developer

---

## 22. Contact

| Role | Name | Contact |
|---|---|---|
| Founder & CEO | [FILL] | [FILL] |
| CTO & Co-founder | [FILL] | [FILL] |
| Grievance Officer (DPDP) | [FILL] | [FILL] |
| Security disclosures | — | `[security@seatu.in]` |
| General inquiries | — | `[support@seatu.in]` |
| Investor inquiries | — | `[investors@seatu.in]` |

---

*This README is a living document. Update it as the project evolves — particularly the status table (§2), env var lists (§10), and roadmap (§18). When in doubt, this should be the most-current operational truth.*

*Last updated: [FILL]*
