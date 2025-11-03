# 🛠️ OpenMenu Builder — Step‑by‑Step Development Roadmap (for GPT‑5 Codex)

**Purpose:** A sequenced, implementation‑ready plan you can feed to GPT‑5 Codex (or any code agent) to build the **OpenMenu Builder** app end‑to‑end.  
**Stack (opinionated defaults):** React + Vite + Tailwind (frontend), Node.js + Express (backend), PostgreSQL + Prisma (ORM), JWT auth, Docker, Nginx reverse proxy.  
**Non‑goals (MVP):** Payments, ordering, POS integration.

---

## 🔑 Guiding Principles

- **Small, verifiable steps.** Every step ends with runnable code and tests.  
- **Contracts first.** Define API + schema before UI wiring.  
- **Security by default.** HTTPS, secure headers, input validation, least privilege.  
- **Docs & scripts.** Each milestone updates README and adds `package.json` scripts.  
- **Automate.** Lint, format, typecheck, tests, and CI from day one.

---

## 📦 Repo & Project Layout

```
openmenu-builder/
├── apps/
│   ├── web/                # React (Vite + TS + Tailwind)
│   └── api/                # Node (Express + TS)
├── packages/
│   └── ui/                 # (optional) shared UI components
├── infra/
│   ├── docker/             # Dockerfiles & compose
│   ├── nginx/              # reverse proxy config
│   └── scripts/            # db/ci utility scripts
├── prisma/                 # Prisma schema & migrations
├── .github/workflows/      # CI pipelines
├── docs/                   # ADRs, API, onboarding
└── README.md
```

---

## 🧩 Data Model (MVP)

> Prisma schema excerpt — use Postgres

```
model User {
  id           String  @id @default(cuid())
  email        String  @unique
  passwordHash String
  role         Role    @default(OWNER)
  restaurants  Restaurant[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

enum Role { OWNER ADMIN }

model Restaurant {
  id          String  @id @default(cuid())
  ownerId     String
  owner       User    @relation(fields: [ownerId], references: [id])
  name        String
  slug        String  @unique
  logoUrl     String?
  phone       String?
  address     String?
  theme       Json    @default("{}")
  menus       Menu[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}

model Menu {
  id            String @id @default(cuid())
  restaurantId  String
  restaurant    Restaurant @relation(fields: [restaurantId], references: [id])
  title         String
  description   String?
  language      String   @default("en")
  isPublished   Boolean  @default(false)
  categories    Category[]
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
}

model Category {
  id      String @id @default(cuid())
  menuId  String
  menu    Menu   @relation(fields: [menuId], references: [id])
  title   String
  sort    Int    @default(0)
  items   MenuItem[]
}

model MenuItem {
  id          String  @id @default(cuid())
  categoryId  String
  category    Category @relation(fields: [categoryId], references: [id])
  name        String
  description String?
  priceCents  Int
  currency    String   @default("USD")
  photoUrl    String?
  tags        String[] @default([]) // allergens, vegan, spicy, etc.
  available   Boolean  @default(true)
  sort        Int      @default(0)
  views       Int      @default(0)
}
```

---

## 🔌 API Contract (MVP)

`/auth/*`: `POST /auth/register`, `POST /auth/login`, `POST /auth/logout`  
`/restaurants`: `GET /me/restaurants`, `POST /restaurants`, `PATCH /restaurants/:id`  
`/menus`: `GET /restaurants/:rid/menus`, `POST /restaurants/:rid/menus`, `PATCH /menus/:id`, `DELETE /menus/:id`  
`/categories`: `POST /menus/:mid/categories`, `PATCH /categories/:id`, `DELETE /categories/:id`  
`/items`: `POST /categories/:cid/items`, `PATCH /items/:id`, `DELETE /items/:id`  
`/public/menu/:slug/:menuId` — **no auth**, cached; increments item/menu views  
`/qr/:menuId` — returns PNG/SVG QR

- **Auth:** Bearer JWT (httpOnly refresh cookie + short‑lived access token).  
- **Validation:** Zod schemas for all payloads.  
- **Errors:** RFC 7807 problem+json.

---

## 🧭 Milestones (Sequential, Each ~0.5–2 days)

> For each milestone: **Goal → Tasks → Acceptance (DoD) → Example prompts for Codex**

### 0) Project Bootstrap
**Goal:** Foundations for monorepo, TypeScript, lint/format, CI, commit hooks.  
**Tasks:**
- Init pnpm workspaces + Turbo (or Nx).  
- Setup `apps/web` (Vite+React+TS+Tailwind), `apps/api` (Express+TS).  
- Prettier, ESLint, lint-staged, Husky; tsconfig base.  
- Health endpoints + simple home page.  
- CI: node matrix (18/20), run lint, typecheck, test.  
**Acceptance:**
- `pnpm i && pnpm build && pnpm test` green locally and in GitHub Actions.  
**Codex Prompt:**
> Scaffold a pnpm monorepo with apps/web (Vite+React+TS+Tailwind) and apps/api (Express+TS). Add ESLint+Prettier, Husky pre-commit for lint-staged, and a GitHub Actions workflow to run lint, typecheck, and tests on push/PR. Provide scripts in root and per app.

---

### 1) Database & ORM
**Goal:** Postgres + Prisma ready with migrations.  
**Tasks:** Add Prisma, schema above, `.env` var parsing (zod), migration scripts.  
**Acceptance:** `pnpm db:migrate` creates tables; seed script adds demo data.  
**Codex Prompt:** 
> Add Prisma to apps/api with PostgreSQL. Implement the provided schema, write migration and seed scripts, and expose a `pnpm db:migrate` and `pnpm db:seed` command.

---

### 2) Auth (JWT + Refresh)
**Goal:** Secure auth with password hashing and sessions.  
**Tasks:** `register/login/logout`, bcrypt, refresh token (httpOnly), access token (Bearer), rate limit.  
**Acceptance:** Auth routes tested (supertest), invalid creds blocked, refresh works.  
**Codex Prompt:**
> Implement auth in Express using bcrypt and JWT (access 15m, refresh 7d via httpOnly cookie). Add Zod validation and supertest tests. Return RFC7807 errors.

---

### 3) Restaurant CRUD
**Goal:** Owners manage restaurant profile.  
**Tasks:** `GET /me/restaurants`, `POST /restaurants`, `PATCH /restaurants/:id`, slug generation, RBAC (OWNER/ADMIN).  
**Acceptance:** Only owner/admin can mutate; validation + tests.  
**Codex Prompt:** 
> Implement restaurant endpoints with owner scoping and slug creation. Add unit tests and integration tests with an authenticated agent.

---

### 4) Menu & Category CRUD
**Goal:** Create menus, categories, items.  
**Tasks:** REST endpoints + Zod DTOs; cascading deletes with Prisma. Sorting fields update.  
**Acceptance:** Full CRUD works; tests cover auth and bad paths.  
**Codex Prompt:** 
> Build CRUD for Menu, Category, and MenuItem with input validation, sorting, and cascading deletes. Provide Jest tests and Prisma transactional operations.

---

### 5) Public Menu Page
**Goal:** Render published menu (unauthenticated) by `slug` + `menuId`.  
**Tasks:** SSR or CSR page that fetches `/public/menu/...`; responsive layout; SEO meta.  
**Acceptance:** Lighthouse ≥ 90 (perf, a11y); loads <1s on local throttling.  
**Codex Prompt:** 
> Create a public menu page in web that fetches `/public/menu/:slug/:menuId`, renders categories/items, is fully responsive, keyboard navigable, and includes basic SEO (title, og tags).

---

### 6) QR Code Generation
**Goal:** Shareable QR that opens the public menu.  
**Tasks:** API route `/qr/:menuId` producing PNG/SVG using `qrcode` lib; download in dashboard.  
**Acceptance:** Scanning opens correct URL; caching headers set.  
**Codex Prompt:** 
> Implement `/qr/:menuId` in api returning PNG and SVG QR codes for the public menu URL. Add caching headers and ETag. Add a download button in the dashboard.

---

### 7) Theming & Branding
**Goal:** Logo upload, colors, fonts stored in `Restaurant.theme`.  
**Tasks:** Theme editor UI; CSS vars; preview; persist to DB; sanitize uploads.  
**Acceptance:** Theme applies to public page; persists across sessions.  
**Codex Prompt:** 
> Add a Theme Editor panel in web to change primary color, font family, and logo. Persist to `Restaurant.theme`. Use CSS variables and Tailwind config extension.

---

### 8) Image Uploads
**Goal:** Safe image handling.  
**Tasks:** Signed upload to local disk or S3‑compatible storage; mime check, size limits, basic transform (max width, compress).  
**Acceptance:** Invalid types rejected; images lazy‑load & have width/height.  
**Codex Prompt:** 
> Implement signed upload endpoints with file type/size validation and Sharp-based resizing. Provide a React Dropzone component with progress and error handling.

---

### 9) Basic Analytics
**Goal:** Track menu views and item popularity.  
**Tasks:** Increment counters on `/public/menu/*`; daily aggregates table (optional). Dashboard charts (views, top items).  
**Acceptance:** Dashboard shows last 30 days; bots filtered via user‑agent heuristic.  
**Codex Prompt:** 
> Add lightweight analytics: increment counters on public views, store daily aggregates via a scheduled job. Build a dashboard chart using Recharts.

---

### 10) Localization (i18n)
**Goal:** Multi‑language menus.  
**Tasks:** Add `language` on Menu; UI toggle; support i18n keys for static UI; optional translations per item later.  
**Acceptance:** Public menu switches locale; URLs stable.  
**Codex Prompt:** 
> Add i18n using `react-i18next`. Make Menu have a `language` field; allow owner to select and render right locale on the public page.

---

### 11) Admin Panel
**Goal:** Manage users/restaurants.  
**Tasks:** Admin routes guarded by role; list/search; suspend/activate accounts.  
**Acceptance:** Non‑admins get 403; audit logs stored.  
**Codex Prompt:** 
> Create an Admin section with tables for Users and Restaurants, with server-side pagination and role checks. Log admin actions to an `audit_log` table.

---

### 12) Deployments (Docker + Nginx)
**Goal:** One‑command local and production deploy.  
**Tasks:** Dockerfiles for api+web, docker‑compose, Nginx reverse proxy, TLS via Caddy or Let’s Encrypt notes.  
**Acceptance:** `docker compose up -d` runs full stack; health checks green.  
**Codex Prompt:** 
> Add Dockerfiles and a docker-compose.yml that runs Postgres, api, and web behind Nginx. Provide health checks, env templates, and a production README with TLS setup.

---

### 13) Security Hardening
**Goal:** Reasonable defaults.  
**Tasks:** Helmet, CORS allow‑list, rate limiting, input sanitization, CSRF for stateful ops, secrets rotation guide.  
**Acceptance:** OWASP ZAP baseline passes; no high findings.  
**Codex Prompt:** 
> Apply Helmet, strict CORS, express-rate-limit, and input sanitation. Add a security.md documenting headers, CORS, and auth flows. Run ZAP baseline in CI.

---

### 14) Docs & Demos
**Goal:** Great onboarding.  
**Tasks:** Update README (quick start, env vars, scripts), seed demo restaurant, record GIF, add API reference.  
**Acceptance:** New dev can run app in <10 minutes.  
**Codex Prompt:** 
> Improve README with quickstart, environment variables, and scripts. Add a seed script that creates a demo restaurant with a sample menu and screenshots.

---

## ✅ Definition of Done (Global)

- All routes validated with Zod; errors are problem+json.  
- Unit + integration tests cover critical paths (≥70% lines in api).  
- CI: lint, typecheck, test, build, ZAP baseline.  
- Docs updated; example `.env.example` present.  
- Docker local run is trivial; prod deploy guide exists.

---

## 🧪 Test Plan (MVP)

- **API:** supertest suites for auth/CRUD; negative tests for authz failures.  
- **E2E:** Playwright flows (login → create restaurant → add menu → publish → view public → scan QR).  
- **Perf:** Public menu Lighthouse ≥ 90; time‑to‑interactive < 2s locally throttled.  
- **Security:** Secret scanning (gitleaks) and dependency audit in CI.

---

## 🤖 Example High‑Level Prompt to Kick Off

> You are GPT‑5 Codex acting as a senior full‑stack engineer. Implement Milestone 0 exactly as specified in `docs/roadmap.md`. Use pnpm workspaces and Typescript. Produce a PR with commits grouped per task, passing CI. Provide a summary, setup instructions, and how to verify acceptance criteria locally.

---

## 🧰 Useful Scripts (to add)

```
# root package.json
"scripts": {
  "dev": "turbo run dev",
  "build": "turbo run build",
  "test": "turbo run test",
  "lint": "turbo run lint",
  "typecheck": "turbo run typecheck",
  "db:migrate": "prisma migrate deploy",
  "db:seed": "ts-node infra/scripts/seed.ts"
}
```

---

## 📑 Backlog (Post‑MVP)

- Schedule‑based menu visibility (breakfast/lunch/dinner windows)  
- Item options/modifiers (sizes, extras)  
- PWA offline cache for public menu  
- Structured analytics (UTM, per‑table QR codes)  
- Webhooks & public REST API for integrations

---

**Owner:** You 🙂  
**File:** `docs/roadmap.md`  
**Tip:** Create GitHub issues per milestone with the included Codex prompts and acceptance criteria.
