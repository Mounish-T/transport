# SEATU — Developer Build Walkthrough

## Hands-on tutorial: blank machine to running system

> **What this is:** a step-by-step guide where you copy commands, paste them into your terminal, and watch a working chit-fund platform come up on your machine. Every step includes the command, what to configure, and what you should see when it works.

> **What this is NOT:** the strategy / architecture document. That's `SEATU-Complete-Procedure.md`. Read that one first if you want to know *why* we made decisions. Read this one if you want to know *how* to actually run and extend the code.

> **Time to complete first time:** 4–8 hours including downloads. Plan an afternoon.

> **What you'll have at the end:** backend running locally, Flutter app on your emulator, full onboarding flow working with a fake OTP, and the muscle memory to add new features.

---

## Table of contents

- [Part 0 — What you'll have at the end (preview)](#part-0)
- [Part 1 — Prerequisites & developer machine setup](#part-1)
- [Part 2 — Get the code on your machine](#part-2)
- [Part 3 — Backend: from blank to running API](#part-3)
- [Part 4 — Flutter app: from blank to onboarding flow](#part-4)
- [Part 5 — Add a feature (backend + frontend)](#part-5)
- [Part 6 — Run the tests](#part-6)
- [Part 7 — Deploying to a real environment](#part-7)
- [Part 8 — Troubleshooting](#part-8)
- [Part 9 — Short-term vs long-term: what to build next](#part-9)

---

## Part 0 — What you'll have at the end <a id="part-0"></a>

When you've followed this whole tutorial, your machine will look like this:

```mermaid
flowchart LR
    YOU[Your machine]

    subgraph DOCKER[Docker containers]
        PG[(Postgres 16<br/>:5432)]
        REDIS[(Redis 7<br/>:6379)]
    end

    subgraph BACKEND[Backend process]
        API[NestJS API<br/>:3000]
    end

    subgraph EMULATOR[Android emulator]
        APP[Flutter app]
    end

    YOU --> DOCKER
    YOU --> BACKEND
    YOU --> EMULATOR
    API --> PG
    API --> REDIS
    APP -->|http via 10.0.2.2| API
```

You'll be able to:

1. Open the app
2. Enter a phone number (`9900000001`)
3. Enter the OTP `123456`
4. Land on the home screen
5. Create a chit-fund room
6. Publish it
7. See it in your rooms list

All running entirely on your laptop, no internet required after first setup.

---

## Part 1 — Prerequisites & developer machine setup <a id="part-1"></a>

### 1.1 — Operating system

This tutorial works on:

| OS | Status |
|---|---|
| macOS (Apple Silicon or Intel) | ✓ Tested |
| Ubuntu 22.04 / 24.04 | ✓ Tested |
| Windows 11 with WSL 2 | ✓ Tested (use Ubuntu inside WSL) |
| Windows 11 without WSL | Not recommended |

If you're on Windows, install WSL 2 first (`wsl --install` in PowerShell as admin, then reboot), then install Ubuntu from the Microsoft Store, and follow the Linux instructions inside WSL.

### 1.2 — Install Node.js 20 (for the backend)

We use Node.js 20 LTS. The easiest cross-platform way is via [nvm](https://github.com/nvm-sh/nvm) (or [nvm-windows](https://github.com/coreybutler/nvm-windows)).

```bash
# Install nvm (Mac/Linux)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Restart your terminal, then:
nvm install 20
nvm use 20
```

**Verify it worked:**

```bash
node --version
```

**You should see:**

```
v20.11.1
```

(any `v20.x.x` is fine).

### 1.3 — Install pnpm (the package manager we use)

We use pnpm (faster than npm, stricter about dependencies):

```bash
npm install -g pnpm@9
```

**Verify:**

```bash
pnpm --version
```

**You should see:**

```
9.0.0
```

### 1.4 — Install Docker Desktop

We use Docker to run Postgres and Redis without polluting your machine with system installs.

- **Mac/Windows:** Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop). Install. Launch it. Wait for the whale icon in your menu bar / system tray to say "Docker is running".
- **Linux:** Install `docker-ce` and `docker-compose-plugin` per your distro's instructions. Add your user to the `docker` group so you don't need sudo.

**Verify:**

```bash
docker --version
docker compose version
```

**You should see (versions may differ):**

```
Docker version 27.0.3, build 7d4bcd8
Docker Compose version v2.28.1
```

### 1.5 — Install Flutter SDK 3.22+

Flutter is the framework for the mobile app.

**Mac/Linux:**

```bash
# Pick a parent directory (e.g., your home folder)
cd ~
git clone https://github.com/flutter/flutter.git -b stable
echo 'export PATH="$PATH:$HOME/flutter/bin"' >> ~/.bashrc   # or ~/.zshrc on Mac
source ~/.bashrc
```

**Verify:**

```bash
flutter --version
```

**You should see:**

```
Flutter 3.22.x • channel stable • https://github.com/flutter/flutter.git
Framework • revision ...
Engine • revision ...
Tools • Dart 3.4.x • DevTools 2.34.x
```

### 1.6 — Install Android Studio (for the Android emulator)

Even if you don't use Android Studio as your IDE, you need it to get the Android SDK and emulator.

Download from [developer.android.com/studio](https://developer.android.com/studio). Install. Open it. Through the welcome wizard:

1. Click **More Actions → SDK Manager**
2. In **SDK Platforms**, check **Android 14 (API 34)**. Apply.
3. In **SDK Tools**, check **Android SDK Build-Tools**, **Android Emulator**, **Android SDK Platform-Tools**. Apply.

Now create an emulator:

1. **More Actions → Virtual Device Manager**
2. Click **Create device**
3. Pick **Pixel 7** (or any modern Pixel)
4. Pick the **Android 14** system image (download if needed)
5. Click **Finish**

**Verify Flutter sees the emulator:**

```bash
flutter doctor
```

**You should see (everything checked except maybe iOS if you're not on Mac):**

```
[✓] Flutter (Channel stable, 3.22.x)
[✓] Android toolchain - develop for Android devices
[✓] Chrome - develop for the web
[✓] Android Studio (version 2024.x)
[✓] Connected device (1 available)
[✓] Network resources
```

If anything has a `[✗]`, run `flutter doctor --android-licenses` and accept all licenses; that fixes 90% of issues.

### 1.7 — Install Git (you probably already have it)

```bash
git --version
```

**You should see something like:**

```
git version 2.39.3
```

If not: Mac has it via Xcode tools (`xcode-select --install`); Linux: `sudo apt install git`; Windows: WSL has it.

### 1.8 — Quick prerequisites sanity check

Run this:

```bash
echo "--- Node ---"      && node --version
echo "--- pnpm ---"      && pnpm --version
echo "--- Docker ---"    && docker --version
echo "--- Flutter ---"   && flutter --version | head -1
echo "--- Git ---"       && git --version
```

If all five print versions, you're ready.

---

## Part 2 — Get the code on your machine <a id="part-2"></a>

### 2.1 — Create a workspace folder

```bash
mkdir -p ~/seatu
cd ~/seatu
```

### 2.2 — Extract the project tarballs

The Phase 9 delivery shipped three tarballs. Extract them all:

```bash
# Assuming you downloaded the tarballs to ~/Downloads
cd ~/seatu
tar -xzf ~/Downloads/SEATU-backend-Phase9.tar.gz
tar -xzf ~/Downloads/SEATU-app-Phase9.tar.gz
tar -xzf ~/Downloads/SEATU-deploy-kit.tar.gz
```

**Verify the layout:**

```bash
ls -la ~/seatu/
```

**You should see:**

```
drwxr-xr-x   seatu-app/
drwxr-xr-x   seatu-backend/
drwxr-xr-x   seatu-deploy-kit/
```

### 2.3 — Initialize git (optional but recommended)

```bash
cd seatu-backend
git init && git add -A && git commit -m "Phase 9 baseline"
cd ../seatu-app
git init && git add -A && git commit -m "Phase 9 baseline"
cd ..
```

Now if you break something while learning, `git diff` shows you what changed and `git checkout .` restores.

---

## Part 3 — Backend: from blank to running API <a id="part-3"></a>

We'll do this in 8 steps. Each step has: the command, an explanation of what it's doing, and what you should see.

### 3.1 — Boot Postgres and Redis

```bash
cd ~/seatu/seatu-backend
docker compose up -d
```

**What this does:** Reads `docker-compose.yml`, downloads Postgres 16 and Redis 7 images if you don't have them (~200 MB total, first time only), and starts them in the background. They listen on the default ports (Postgres 5432, Redis 6379) and persist data in named Docker volumes.

**Expected output:**

```
[+] Running 4/4
 ✔ Network seatu-backend_default  Created
 ✔ Volume "seatu-backend_postgres-data" Created
 ✔ Volume "seatu-backend_redis-data"    Created
 ✔ Container seatu-postgres       Started
 ✔ Container seatu-redis          Started
```

**Verify both are healthy:**

```bash
docker compose ps
```

**You should see:**

```
NAME             IMAGE                COMMAND                 STATUS         PORTS
seatu-postgres   postgres:16-alpine   "docker-entrypoint…"    Up (healthy)   0.0.0.0:5432->5432/tcp
seatu-redis      redis:7-alpine       "docker-entrypoint…"    Up (healthy)   0.0.0.0:6379->6379/tcp
```

The word **healthy** matters. If it says **starting** for more than 30 seconds, something's wrong — check `docker compose logs postgres`.

### 3.2 — Copy the env file

```bash
cp .env.example .env
```

**What this does:** Creates a `.env` file with sensible local defaults. The `.env.example` is committed; the `.env` is gitignored so secrets don't leak.

**Inspect what's inside:**

```bash
head -20 .env
```

**You should see something like:**

```
# Node
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug

# Postgres
DATABASE_URL=postgresql://seatu:seatu@localhost:5432/seatu?schema=public

# Redis
REDIS_URL=redis://localhost:6379
...
```

For local dev you don't need to change anything. Adapter providers are set to `fake` by default — no real KYC, no real SMS, all stubbed locally.

### 3.3 — Install Node dependencies

```bash
pnpm install
```

**What this does:** Reads `package.json`, downloads ~600 npm packages into `node_modules/`, generates lockfile. First time takes ~2 minutes; subsequent runs are instant when nothing changed.

**Expected output (tail):**

```
Progress: resolved 712, reused 0, downloaded 712, added 712, done

dependencies:
+ @nestjs/common 10.3.7
+ @nestjs/core 10.3.7
...

devDependencies:
+ @nestjs/cli 10.3.2
...

Done in 1m 47s
```

If you see warnings about peer dependencies, ignore them — they're harmless for our setup.

### 3.4 — Generate the Prisma client

```bash
pnpm prisma:generate
```

**What this does:** Reads `prisma/schema.prisma`, generates a typed TypeScript client at `node_modules/.prisma/client/`. This client is what gives you `prisma.user.findMany({...})` with autocomplete.

**Expected output:**

```
Environment variables loaded from .env
Prisma schema loaded from prisma/schema.prisma

✔ Generated Prisma Client (v5.13.0) to ./node_modules/@prisma/client in 234ms
```

### 3.5 — Apply the database migrations

```bash
pnpm prisma:migrate:dev
```

**What this does:** Connects to the Postgres in your Docker container, creates the 12 schemas (`iam`, `room`, `auction`, ...), creates all 40+ tables, applies the append-only triggers, and records the migration in `_prisma_migrations`.

**Expected output:**

```
Environment variables loaded from .env
Prisma schema loaded from prisma/schema.prisma
Datasource "db": PostgreSQL database "seatu" at "localhost:5432"

Applying migration `00000000000000_bootstrap`
Applying migration `00000000000001_append_only_triggers`

The following migration(s) have been applied:

migrations/
  └─ 00000000000000_bootstrap/
    └─ migration.sql
  └─ 00000000000001_append_only_triggers/
    └─ migration.sql

Your database is now in sync with your schema.
```

If you want to peek at the DB to confirm the tables exist:

```bash
docker compose exec postgres psql -U seatu -d seatu -c "\dn"
```

**You should see the 12 schemas:**

```
       Name        |  Owner
-------------------+---------
 admin             | seatu
 audit             | seatu
 auction           | seatu
 dispute           | seatu
 gems              | seatu
 iam               | seatu
 ledger            | seatu
 notification      | seatu
 public            | seatu
 room              | seatu
 shared            | seatu
 trust             | seatu
 verification      | seatu
```

### 3.6 — Seed demo data

```bash
pnpm db:seed
```

**What this does:** Runs `prisma/seed.ts`, which inserts 10 demo users (Lakshmi, Karthik, Rekha, …), 3 rooms in various states, 1 open auction. Useful for local testing without going through the signup flow every time.

**Expected output:**

```
🌱 Seeding SEATU dev data...
✅ Seeded 10 users, 3 rooms, 1 open auction

Demo logins (OTP = 123456 in dev):
  +919000000001  Lakshmi    (gold)
  +919000000002  Karthik    (silver)
  +919000000003  Rekha      (bronze)
  +919000000004  Murugan    (silver)
  +919000000005  Anitha     (bronze)
  +919000000006  Suresh     (platinum)
  +919000000007  Diana      (bronze)
  +919000000008  Vimal      (bronze)
  +919000000009  Founder    (diamond)
  +919000000010  Ops        (diamond)
```

Note the phones and OTP — you'll use them in Part 4.

### 3.7 — Start the API

```bash
pnpm start:dev
```

**What this does:** Boots the NestJS app in watch mode. Listens on `:3000`. Auto-reloads when you edit any file in `src/`.

**Expected output (tail):**

```
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [NestFactory] Starting Nest application...
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [InstanceLoader] AppModule dependencies initialized
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [InstanceLoader] PrismaModule dependencies initialized
...
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [RoutesResolver] AuthController {/v1/auth}: +12ms
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [RouterExplorer] Mapped {/v1/auth/otp/request, POST}
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [RouterExplorer] Mapped {/v1/auth/otp/verify, POST}
[Nest] 1234  - 06/03/2026, 10:23:01     LOG [RouterExplorer] Mapped {/v1/auth/refresh, POST}
...
SEATU API listening on :3000
```

**The API is now live.** Leave this terminal running.

### 3.8 — Smoke test: hit the API with curl

Open a **new** terminal (the first one is busy running the API).

**Liveness check:**

```bash
curl http://localhost:3000/healthz
```

**You should see:**

```json
{"status":"ok"}
```

**Readiness check (verifies the DB is reachable):**

```bash
curl http://localhost:3000/readyz
```

**You should see:**

```json
{"status":"ready"}
```

**Full onboarding flow via curl:**

```bash
# Step 1 — Request an OTP for Lakshmi (one of the seeded users)
curl -X POST http://localhost:3000/v1/auth/otp/request \
  -H "Content-Type: application/json" \
  -H "X-Device-Id: 11111111-1111-4111-8111-111111111111" \
  -H "Idempotency-Key: test-$(date +%s)" \
  -d '{"phone":"+919000000001"}'
```

**You should see:**

```json
{"otp_ref":"abc123...","resend_after_s":60}
```

**In the API terminal, you'll see the fake OTP printed in the logs:**

```
LOG [FakeSmsAdapter] {"phone":"+919000000001","otp":"482915 (dev; bypass=123456)"} [fake-sms] OTP
```

(In real prod with MSG91 wired, an actual SMS would be sent. Locally it just logs.)

**Step 2 — Verify the OTP using the dev-bypass code:**

```bash
curl -X POST http://localhost:3000/v1/auth/otp/verify \
  -H "Content-Type: application/json" \
  -H "X-Device-Id: 11111111-1111-4111-8111-111111111111" \
  -H "Idempotency-Key: verify-$(date +%s)" \
  -d '{
    "phone": "+919000000001",
    "otp": "123456",
    "device_fingerprint": "aabbccddeeff00112233445566778899",
    "device_platform": "android"
  }'
```

**You should see:**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "rt_xY...",
  "access_expires_at": "2026-06-03T10:38:01.234Z",
  "refresh_expires_at": "2026-07-03T10:23:01.234Z",
  "user": {
    "id": "uuid-here",
    "phone": "+919000000001",
    "name": "Lakshmi",
    "tier": "gold",
    "score": 580,
    "gem_balance": 100,
    "verifications": {"phone": true, "pan": false, "bank": false},
    ...
  }
}
```

**Step 3 — Use the access token to fetch /me:**

```bash
ACCESS_TOKEN="paste the access_token from above"

curl http://localhost:3000/v1/me \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Device-Id: 11111111-1111-4111-8111-111111111111"
```

**You should see:** the same user object as above.

**🎉 The backend is fully working.** You've just exercised the entire auth flow that the Flutter app will use.

---

## Part 4 — Flutter app: from blank to onboarding flow <a id="part-4"></a>

Leave the backend terminal running. The app talks to it.

### 4.1 — Fetch Flutter dependencies

```bash
cd ~/seatu/seatu-app
flutter pub get
```

**What this does:** Reads `pubspec.yaml`, downloads ~50 Dart packages into the global Pub cache, generates `pubspec.lock`.

**Expected output (tail):**

```
Resolving dependencies...
Downloading packages...
Got dependencies!
```

### 4.2 — Run code generation

```bash
dart run build_runner build --delete-conflicting-outputs
```

**What this does:** Reads every `.dart` file with `@freezed`, `@riverpod`, `@JsonSerializable` annotations. Generates the matching `.freezed.dart`, `.g.dart` files. First run takes ~30 seconds; incremental runs are fast.

**Expected output (tail):**

```
[INFO] Generating build script completed, took 358ms
[INFO] Initializing inputs
[INFO] Building new asset graph completed, took 412ms
...
[INFO] Running build completed, took 18s
[INFO] Caching finalized dependency graph completed, took 67ms
[INFO] Succeeded after 18.6s with 47 outputs (143 actions)
```

The `47 outputs` are the generated files — one per `@freezed`/`@riverpod` annotation. If you see errors here, run `dart run build_runner build --delete-conflicting-outputs` again — sometimes the first run has order issues that resolve on the second.

**Tip:** While developing, run `dart run build_runner watch --delete-conflicting-outputs` in a separate terminal. It regenerates on every save — no more manual reruns.

### 4.3 — Start the Android emulator

```bash
flutter emulators
```

**You should see your emulator listed:**

```
1 available emulator:

Id           • Name        • Manufacturer • Platform

Pixel_7_API_34 • Pixel 7 API 34 • Google       • android
```

**Launch it:**

```bash
flutter emulators --launch Pixel_7_API_34
```

A black window appears, then an Android home screen. The first launch takes ~1 minute.

### 4.4 — Run the app

```bash
flutter run --dart-define-from-file=env/dev.json
```

**What this does:**
- Compiles the Flutter app
- Installs it on the running emulator
- Connects DevTools for hot-reload
- The `--dart-define-from-file` flag injects the dev environment values (API base URL, etc.)

**Expected output:**

```
Launching lib/main.dart on sdk gphone64 arm64 in debug mode...
Running Gradle task 'assembleDebug'...
✓ Built build/app/outputs/flutter-apk/app-debug.apk
Installing build/app/outputs/flutter-apk/app-debug.apk...
Syncing files to device sdk gphone64 arm64...

Flutter run key commands.
r Hot reload. 🔥🔥🔥
R Hot restart.
h List all available interactive commands.
d Detach (terminate "flutter run" but leave application running).
c Clear the screen
q Quit (terminate the application on the device).

A Dart VM Service on sdk gphone64 arm64 is available at:
http://127.0.0.1:50451/...
The Flutter DevTools debugger is available at:
http://127.0.0.1:9100?uri=http://127.0.0.1:50451/...
```

The emulator screen now shows the SEATU app.

### 4.5 — What you should see on screen (walkthrough)

**Screen 1: Welcome**

```
┌────────────────────────────┐
│                  Language ▾│
│                            │
│           ┌──┐              │
│           │S │              │
│           └──┘              │
│                            │
│         SEATU              │
│   Run your chit fund —     │
│      together, safely.     │
│                            │
│                            │
│   ┌────────────────────┐   │
│   │     Get Started    │   │
│   └────────────────────┘   │
└────────────────────────────┘
```

Tap **Get Started**.

**Screen 2: Phone input**

```
┌────────────────────────────┐
│ ←                          │
│                            │
│   Enter your phone number  │
│                            │
│   ┌────────────────────┐   │
│   │ +91   9900000001  │   │
│   └────────────────────┘   │
│                            │
│                            │
│                            │
│                            │
│                            │
│   ┌────────────────────┐   │
│   │     Continue       │   │
│   └────────────────────┘   │
└────────────────────────────┘
```

Type `9000000001` (Lakshmi's phone from the seed). Tap **Continue**.

**What happens in the background:** The app calls `POST /v1/auth/otp/request`. Watch the API terminal — you'll see the request logged.

**Screen 3: OTP verify**

```
┌────────────────────────────┐
│ ←                          │
│                            │
│   Enter the 6-digit OTP    │
│   +91 90000 00001          │
│                            │
│       ┌───────────────┐    │
│       │  1 2 3 4 5 6  │    │
│       └───────────────┘    │
│                            │
│       Resend OTP (60s)     │
│                            │
│                            │
│   ┌────────────────────┐   │
│   │     Verify         │   │
│   └────────────────────┘   │
└────────────────────────────┘
```

Type `123456`. The app auto-submits when you hit 6 digits.

**What happens:** The app calls `POST /v1/auth/otp/verify`. On success, the response includes tokens + user. The router auto-redirects to the home screen.

**Screen 4: Home (Rooms tab)**

```
┌────────────────────────────┐
│ Your Rooms              ⟳  │
├────────────────────────────┤
│ ┌──────────────────────┐   │
│ │ Lakshmi's TextileFund│   │
│ │ LAK001        [Active]│   │
│ │                      │   │
│ │ 💰 ₹2,000 / 30d  👥 5/5│   │
│ └──────────────────────┘   │
│                            │
│ ┌──────────────────────┐   │
│ │ Lakshmi's Friends ₹5k│   │
│ │ LAK002    [Recruiting]│   │
│ │                      │   │
│ │ 💰 ₹5,000 / 30d  👥 4/10│   │
│ └──────────────────────┘   │
│                            │
│                  + Create  │
│                            │
├────────────────────────────┤
│ 🏠 Rooms 🛡 Trust 💎 Gems 👤 │
└────────────────────────────┘
```

You're signed in as Lakshmi. The seeded rooms appear.

### 4.6 — Create a new room

Tap **+ Create**. Fill in the form:

```
┌────────────────────────────┐
│ ← Create Room              │
├────────────────────────────┤
│  Room name                 │
│  ┌──────────────────────┐  │
│  │ My new chit         │   │
│  └──────────────────────┘  │
│                            │
│  Contribution per member   │
│  ┌──────────────────────┐  │
│  │ ₹ 2000              │   │
│  └──────────────────────┘  │
│                            │
│  Number of members         │
│  ┌──────────────────────┐  │
│  │ 5                   │   │
│  └──────────────────────┘  │
│                            │
│ ┌──────────────────────┐   │
│ │ Each cycle pool      │   │
│ │ ₹10,000              │   │
│ │ Total over 5 cycles: │   │
│ │ ₹50,000              │   │
│ └──────────────────────┘   │
│                            │
│ ┌──────────────────────┐   │
│ │   Create Room        │   │
│ └──────────────────────┘   │
└────────────────────────────┘
```

Tap **Create Room**. You'll be navigated to the room detail screen.

### 4.7 — Publish the room (test optimistic UI)

The newly-created room is in `draft` status. Tap **Publish to recruiting**.

**What you should observe:** The status badge flips to `Recruiting` **instantly** — before the server responds. This is the Phase 7 optimistic UI in action. The actual network call happens behind the scenes and reconciles.

If you want to *see* the optimistic-rollback behaviour:
1. Stop the backend (`Ctrl+C` in the backend terminal)
2. In the app, tap Publish again on another draft room
3. The status flips to `Recruiting` (optimistic)
4. After ~10 seconds the request times out, the status snaps back to `draft`, and a toast appears: "No internet connection. Check your network and try again."
5. Restart the backend with `pnpm start:dev` to recover.

**🎉 The Flutter app is fully working.** You now have the entire onboarding + room creation + room lifecycle flow.

---

## Part 5 — Add a feature (backend + frontend) <a id="part-5"></a>

This is where you stop running existing code and start writing new code. We'll add a **simple feature** end-to-end: a `GET /v1/me/stats` endpoint that returns aggregate stats for the current user (rooms count, total contributions tracked, current trust score).

This walks through the exact pattern you'll use for every new feature.

### 5.1 — Backend: add the endpoint

**Open** `seatu-backend/src/modules/iam/controllers/me.controller.ts` in your editor.

**Find the existing `MeController` class** (around line 11). At the bottom, add:

```typescript
  @Get('stats')
  async stats(@CurrentUser() user: AuthenticatedUser) {
    const [roomsCount, contributionsCount, trust] = await Promise.all([
      this.prisma.roomMember.count({ where: { userId: user.userId } }),
      this.prisma.contribution.count({
        where: { userId: user.userId, status: 'confirmed' },
      }),
      this.prisma.trustScoreCurrent.findUnique({
        where: { userId: user.userId },
      }),
    ]);

    return {
      rooms_count: roomsCount,
      contributions_count: contributionsCount,
      trust_score: trust?.score ?? 100,
      tier: trust?.tier ?? 'bronze',
    };
  }
```

**Save the file.** The backend (running in watch mode) auto-reloads. You should see in the API terminal:

```
[Nest] 1234  - 06/03/2026, 11:00:01     LOG [RouterExplorer] Mapped {/v1/me/stats, GET}
```

**Test it with curl** (use the access token from earlier):

```bash
curl http://localhost:3000/v1/me/stats \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Device-Id: 11111111-1111-4111-8111-111111111111"
```

**You should see:**

```json
{
  "rooms_count": 3,
  "contributions_count": 0,
  "trust_score": 580,
  "tier": "gold"
}
```

That's the backend side done.

### 5.2 — Frontend: consume the new endpoint

**Open** `seatu-app/lib/features/profile/presentation/profile_tab.dart`.

We'll add a stats card below the verification status.

**At the top of the file**, near the other Riverpod providers, add:

```dart
final meStatsProvider = FutureProvider.autoDispose((ref) async {
  final dio = ref.read(dioClientProvider);
  final res = await dio.get<Map<String, dynamic>>('/v1/me/stats');
  return res.data!;
});
```

(You'll need to add the import: `import 'package:dio/dio.dart';` and `import 'package:seatu/core/api/api_client.dart';`)

**Find the `ListView` in `ProfileTab.build`.** After the `_VerificationStatus` card, add a new `_StatsCard`:

```dart
          _VerificationStatus(me: me),
          const SizedBox(height: 24),
          // ⬇️ NEW:
          const _StatsCard(),
          const SizedBox(height: 24),
```

**At the bottom of the file**, add the `_StatsCard` widget:

```dart
class _StatsCard extends ConsumerWidget {
  const _StatsCard();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final stats = ref.watch(meStatsProvider);
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Your activity',
                style: Theme.of(context).textTheme.titleMedium),
            const SizedBox(height: 12),
            stats.when(
              loading: () => const Padding(
                padding: EdgeInsets.symmetric(vertical: 12),
                child: Center(child: CircularProgressIndicator()),
              ),
              error: (err, _) => Text('Failed to load: $err'),
              data: (data) => Column(
                children: [
                  _statRow(
                      context, 'Rooms joined', '${data['rooms_count']}'),
                  _statRow(context, 'Contributions tracked',
                      '${data['contributions_count']}'),
                  _statRow(context, 'Current trust score',
                      '${data['trust_score']}'),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _statRow(BuildContext context, String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label),
          Text(value,
              style: Theme.of(context).textTheme.titleMedium),
        ],
      ),
    );
  }
}
```

**Save the file.** In the running `flutter run` terminal, press `r` for **hot reload**.

**Switch to the Profile tab in the app.** You should see a new "Your activity" card with the three stats.

```
┌────────────────────────────┐
│ Profile                    │
├────────────────────────────┤
│ ┌──────────────────────┐   │
│ │ Lakshmi              │   │
│ │ +91 90000 00001      │   │
│ │ [🥇 Gold · 580]      │   │
│ └──────────────────────┘   │
│                            │
│ ┌──────────────────────┐   │
│ │ Verification         │   │
│ │ ✓ Phone              │   │
│ │ ✗ PAN                │   │
│ │ ✗ Bank               │   │
│ └──────────────────────┘   │
│                            │
│ ┌──────────────────────┐   │
│ │ Your activity        │   │  ⬅ NEW
│ │ Rooms joined      3  │   │
│ │ Contributions tr… 0  │   │
│ │ Current trust sc… 580│   │
│ └──────────────────────┘   │
└────────────────────────────┘
```

**🎉 You just added a feature end-to-end.** The same pattern applies to every new endpoint:
1. Backend: add controller method → it auto-mounts → test with curl
2. Frontend: add a provider, add a widget, hot-reload, see it on screen

### 5.3 — The recipe in summary

```mermaid
flowchart TB
    A[Decide what data the UI needs] --> B[Backend: add controller method]
    B --> C[Backend: auto-reload]
    C --> D[Backend: test with curl]
    D --> E[Frontend: add Riverpod provider]
    E --> F[Frontend: add widget]
    F --> G[Frontend: hot-reload<br/>press r]
    G --> H[Verify on emulator]
```

For more complex features (new modules), follow the `iam` and `room` modules as templates — they're the reference implementations. The detailed recipe is in `seatu-backend/SEATU-Backend-Phase5.md` §4.

---

## Part 6 — Run the tests <a id="part-6"></a>

### 6.1 — Backend tests

In the backend directory:

```bash
cd ~/seatu/seatu-backend
pnpm test:unit
```

**What this does:** Runs the unit tests in `test/unit/` (pure logic, no DB). Fast — finishes in under a second.

**Expected output:**

```
 PASS  test/unit/tier-rules.spec.ts
  tier-rules
    tierAtLeast
      ✓ returns true when actual equals minimum (3 ms)
      ✓ returns true when actual exceeds minimum (1 ms)
      ✓ returns false when actual is below minimum (1 ms)
    ...
Test Suites: 1 passed, 1 total
Tests:       7 passed, 7 total
Time:        0.812s
```

**Integration tests** (uses Testcontainers — boots fresh Postgres+Redis):

```bash
pnpm test:integration
```

**Takes ~30s the first time** (downloads container images), faster after.

**Expected output:**

```
 PASS  test/integration/onboarding.spec.ts
  Onboarding flow (integration)
    ✓ OTP request -> verify -> me -> refresh -> me with rotated token (3245 ms)
    ✓ rejects request without Idempotency-Key on mutating endpoint (89 ms)
    ✓ rejects unauthenticated /me (45 ms)

Test Suites: 1 passed, 1 total
Tests:       3 passed, 3 total
Time:        38.234s
```

### 6.2 — Flutter tests

```bash
cd ~/seatu/seatu-app
flutter test
```

**Expected output:**

```
00:00 +0: loading test/integration/cache_box_test.dart
00:02 +12: All tests passed!
```

(The exact count depends on which tests are present.)

**To run a specific test file:**

```bash
flutter test test/integration/rooms_controller_test.dart
```

**To regenerate the golden tier-badge image** after a visual change:

```bash
flutter test --update-goldens test/golden/
```

### 6.3 — Adding your own test

Following the pattern from Part 5, here's how to add a test for the `/v1/me/stats` endpoint you just built:

Create `seatu-backend/test/integration/me-stats.spec.ts`:

```typescript
import * as request from 'supertest';
import { buildTestApp, TestAppHandle, baseHeaders } from '../helpers/test-app';

describe('GET /v1/me/stats', () => {
  let handle: TestAppHandle;

  beforeAll(async () => { handle = await buildTestApp(); });
  afterAll(async () => { await handle?.stop(); });

  it('returns zeros for a fresh user', async () => {
    const server = handle.app.getHttpServer();

    // Sign up
    await request(server)
      .post('/v1/auth/otp/request')
      .set(baseHeaders)
      .send({ phone: '+919900099099' })
      .expect(200);

    const verify = await request(server)
      .post('/v1/auth/otp/verify')
      .set({ ...baseHeaders, 'Idempotency-Key': 'verify-stats-1' })
      .send({
        phone: '+919900099099',
        otp: '123456',
        device_fingerprint: 'b'.repeat(64),
        device_platform: 'android',
      });

    const accessToken = verify.body.access_token;

    // Call /me/stats
    const stats = await request(server)
      .get('/v1/me/stats')
      .set({
        Authorization: `Bearer ${accessToken}`,
        'X-Device-Id': baseHeaders['X-Device-Id'],
      })
      .expect(200);

    expect(stats.body).toMatchObject({
      rooms_count: 0,
      contributions_count: 0,
      trust_score: 100,
      tier: 'bronze',
    });
  });
});
```

Run it:

```bash
pnpm test:integration
```

**You should see the new test pass alongside the existing ones.**

---

## Part 7 — Deploying to a real environment <a id="part-7"></a>

The full deployment procedure is in `seatu-deploy-kit/SEATU-Launch-Runbook.md`. That's a 15 KB checklist covering infrastructure, providers (MSG91/Karza/FCM), Flutter store builds, and compliance.

The TL;DR is:

```mermaid
flowchart LR
    A[Provision Postgres + Redis] --> B[Set env secrets in host dashboard]
    B --> C[Push to main]
    C --> D[CI runs, deploys]
    D --> E[Smoke test against staging]
    E --> F{All green?}
    F -->|Yes| G[Promote to production]
    F -->|No| H[Rollback]
    G --> I[Build Flutter APK]
    I --> J[Upload to Play Store internal track]
```

For local development you don't need any of this. Stay on `localhost` until you're ready to launch.

---

## Part 8 — Troubleshooting <a id="part-8"></a>

The errors you'll actually hit, in rough order of likelihood:

### "Cannot connect to the Docker daemon"

Docker Desktop isn't running. Open it; wait for the whale icon.

### "Port 5432 is already allocated"

Another Postgres is running on your machine (system install). Either stop it (`sudo systemctl stop postgresql` on Linux, `brew services stop postgresql` on Mac) or change the port in `docker-compose.yml` from `5432:5432` to `5433:5432`, then update `DATABASE_URL` in `.env` to use `localhost:5433`.

### "Prisma can't reach the database"

Three possibilities:
1. Docker not running → fix per above
2. Wrong `DATABASE_URL` → check it matches your Docker port
3. Container not healthy → `docker compose logs postgres`

### "EADDRINUSE: address already in use :::3000"

The backend is already running in another terminal, or some other app is on port 3000. Find it: `lsof -i :3000` (Mac/Linux). Kill it or change `PORT=3001` in `.env`.

### Flutter: "No connected devices"

Emulator not running. `flutter emulators --launch Pixel_7_API_34`. Wait 30 seconds. `flutter run` again.

### Flutter: "build_runner has errors"

Run it again with `--delete-conflicting-outputs`. If that fails, the most common cause is a syntax error in a `.dart` file — read the error message carefully, fix the file, re-run.

### Flutter app: "No internet connection" banner appears immediately

The app can't reach the backend. Check:
- Backend running? (`curl localhost:3000/healthz` from your machine)
- Using emulator? Then `SEATU_API_BASE` should be `http://10.0.2.2:3000` (the emulator's address for "host machine")
- Using real device? `SEATU_API_BASE` should be your laptop's LAN IP (e.g. `http://192.168.1.5:3000`) — and the laptop firewall must allow it

### "AUTH_TOKEN_REUSE_DETECTED" on second refresh

You're calling `/v1/auth/refresh` twice with the same refresh token. The second call kills the entire family per Phase 5 §8 — this is correct behaviour for theft response. In the app, sign out and back in.

### The OTP I see in logs doesn't work — it asks me to enter again

You probably typed it from the log instead of using `123456` (the dev bypass). In dev mode, both work — the actual OTP printed in logs OR the hardcoded `123456`. Use `123456`; it always works in dev.

---

## Part 9 — Short-term vs long-term: what to build next <a id="part-9"></a>

You now have a working system. From here, every new feature has a short-term and long-term variant. Pick the right one based on whether you're optimising for **shipping fast** or **scaling cleanly**.

### Short-term (next 30 days): finish the MVP

In rough priority order, the highest-value additions are:

| # | What | Where | Effort |
|---|---|---|---|
| 1 | Verification module (PAN + bank KYC) | Backend skeleton → real | 2 days |
| 2 | KYC screens in Flutter | New feature in `lib/features/verification/` | 3 days |
| 3 | Auction module + bid endpoints | Backend skeleton → real | 3 days |
| 4 | Auction screen in Flutter | New feature in `lib/features/auctions/` | 2 days |
| 5 | Contribution mark-paid + screenshot upload | Backend ledger + Flutter screen | 4 days |
| 6 | Dispute file + respond | Backend + Flutter | 5 days |

Each follows the Part 5 recipe. Total: ~20 dev-days = 4 weeks for one developer.

### Long-term (next 6 months): scale prep

Don't start any of these until you have actual scaling pain:

| # | What | When to invest |
|---|---|---|
| 1 | Extract OutboxDispatcher to its own worker binary | When push+SMS volume > 100k/day |
| 2 | Read replicas for Postgres | When read load > 50% of master capacity |
| 3 | WebSocket for live notifications | When users complain about "inbox feels stale" |
| 4 | Multi-region deployment | When you have users outside India |
| 5 | E2E test suite | When deployment confidence drops |
| 6 | iOS App Store build | When Android validates the model (typically month 3-6) |

The mistake is investing in long-term work before short-term value is delivered. Ship → measure → scale. Not scale → ship.

### How to decide

Every Monday morning ask yourself two questions:

1. **Can a real user accomplish their goal end-to-end with what we have?**
   - If NO → short-term work (build missing features)
   - If YES → ask question 2

2. **Is the system in pain at current scale?**
   - If NO → keep adding short-term features
   - If YES → long-term work to relieve pain

That's the whole decision framework. Everything else is rationalisation.

---

## Quick reference: the commands you'll run most

```bash
# Backend
cd ~/seatu/seatu-backend
docker compose up -d          # start postgres + redis
pnpm start:dev                # start API
pnpm db:reset                 # wipe + migrate + reseed
pnpm test                     # run all tests
pnpm prisma:studio            # open DB GUI in browser

# Flutter
cd ~/seatu/seatu-app
flutter pub get               # install deps
dart run build_runner watch --delete-conflicting-outputs   # codegen on save
flutter run --dart-define-from-file=env/dev.json           # start the app
flutter test                  # run tests
# Then in the running 'flutter run' terminal:
r                             # hot reload
R                             # hot restart (full reload)
q                             # quit

# Both
docker compose down            # stop postgres + redis
docker compose down -v         # stop AND wipe data (start over)
```

---

*If you got stuck at any step, the most useful thing you can do is paste the exact error message into the team chat. The patterns above cover the common cases; the rest is just reading error messages carefully.*

*— CTO, June 2026*
