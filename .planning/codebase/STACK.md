# Technology Stack

**Analysis Date:** 2026-03-19

## Languages

**Primary:**
- JavaScript (ES Modules) — Runtime and all application code; `import`/`export` throughout
- SQL — SQLite schema and migrations

**Secondary:**
- HTML/CSS — Server-rendered templates with Tailwind CSS (CDN); static marketing site in `mostlypostly-site/`

## Runtime

**Environment:**
- Node.js 18.0.0 or higher (specified in `package.json` `engines.node`)
- Render.com — Deployment platform with auto-deploy from `main` (production) and `dev` (staging) branches

**Package Manager:**
- npm
- Lockfile: `package-lock.json` (committed)

## Frameworks

**Core:**
- Express.js 4.19.2 — HTTP server, routing, middleware (auth, CSRF, sessions)
- socket.io 4.8.1 — Real-time bidirectional communication

**Database:**
- better-sqlite3 12.4.1 — Synchronous SQLite client; raw queries via `.prepare()`, no ORM
- better-sqlite3-session-store 0.1.0 — Express session persistence in SQLite WAL

**Testing:**
- vitest 3.2.4 — Test runner
- ajv 8.17.1 — JSON schema validator
- ajv-formats 3.0.1 — Format validators for ajv

**Build/Dev:**
- puppeteer 24.39.1 — Headless Chrome for server-side image rendering (SVG → PNG for availability/promotion cards)
- sharp 0.34.5 — Image processing (resize, format conversion)
- nodemon 3.1.10 — Dev mode file watching and auto-restart
- Tailwind CSS (CDN) — Utility-first CSS framework; no build step (inline config per page)

## Key Dependencies

**Critical:**
- openai 6.8.1 — GPT-4o-mini for caption generation, vision-based post type classification
- twilio 5.10.4 — SMS/MMS inbound/outbound, RCS messaging, webhook validation
- stripe 20.4.1 — Billing, checkout sessions, subscription management (API version 2024-06-20)
- uuid 13.0.0 — UUID generation for database records

**Authentication & Security:**
- bcryptjs 3.0.3 — Password hashing; no plain-text passwords
- helmet 8.1.0 — HTTP security headers
- express-session 1.18.2 — Session management with better-sqlite3 store
- cookie-parser 1.4.7 — Cookie middleware
- express-rate-limit 8.3.1 — Rate limiting on public endpoints

**Image & Data Processing:**
- luxon 3.7.2 — Timezone-aware date/time (salon local times for scheduling)
- json2csv 6.0.0-alpha.2 — CSV import/export for vendor campaigns
- qrcode 1.5.4 — QR code generation (internal tools)
- resend 6.9.3 — Transactional email API (password resets, welcome emails)

**Chat & Bot Frameworks:**
- botbuilder 4.23.3 — Microsoft Bot Framework SDK
- botframework-connector 4.23.3 — Bot Framework connector

**Utilities:**
- dotenv 16.4.5 — Environment variable loading from `.env`
- chokidar 4.0.3 — File system watching (local salons.json hot reload)
- multer 2.1.1 — Multipart form data handling (file uploads)
- node-fetch 3.3.2 — HTTP fetch for external APIs
- otplib 13.3.0 — One-time password generation (2FA)

**Frontend:**
- @fontsource/open-sans 5.2.7 — Local font distribution

## Configuration

**Environment Variables:**
- `.env` file (not committed; see `.env.template`)
  - `APP_ENV` — `local` | `staging` | `production` (controls DB path, seeders, logging)
  - `NODE_ENV` — Standard Express env (`production` in Render)
  - Required vars: OPENAI_API_KEY, TWILIO_*, STRIPE_*, FACEBOOK_*, GOOGLE_*, RESEND_API_KEY, etc.

**Database:**
- SQLite 3 via better-sqlite3
- Location depends on APP_ENV:
  - Production: `/data/postly.db` (Render persistent volume)
  - Staging: `/tmp/postly.db`
  - Local: `./postly.db` in working directory
- Schema: `schema.sql` (idempotent base schema)
- Migrations: `migrations/` folder with numbered files (001–043+), applied by `src/core/migrationRunner.js`
- PRAGMAs: `journal_mode = WAL`, `synchronous = FULL`, 10s connection timeout

**Deployment:**
- `.render.yaml` — Render deployment config
  - Build command: `npm install && npx puppeteer browsers install chrome`
  - Start command: `node server.js`
  - Plan: Starter (512MB RAM, 0.5 CPU)
  - Requires Puppeteer flags: `--single-process --no-zygote` for RAM constraint

## Platform Requirements

**Development:**
- Node.js 18+
- npm
- Chrome/Chromium binary (installed via `npm postinstall`)

**Production:**
- Render.com Starter plan
- Persistent volume for database

---

*Stack analysis: 2026-03-19*
