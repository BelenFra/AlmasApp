# Alma App

[![CI](https://github.com/BelenFra/AlmasApp/actions/workflows/ci.yml/badge.svg)](https://github.com/BelenFra/AlmasApp/actions/workflows/ci.yml)
![Status](https://img.shields.io/badge/status-in%20development-yellow)
![License](https://img.shields.io/badge/license-private-lightgrey)

> Internal management system for **Alma Pastelería** (Bakery) and **Alma Bodegón** (Bistro).

A mobile-first web application that gives employees a fast way to log daily sales and gives owners complete operational oversight — from product catalog and cash flow to ingredient inventory and recipe-based stock control.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Documentation](#documentation)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Roles & Access](#roles--access)
- [Roadmap](#roadmap)

---

## Features

### Phase 1 — Product Catalog & Sales (MVP)
- **Product management** with auto-generated SKUs (`BKR-XXXX` / `BST-XXXX`), business unit tagging, and automatic timestamps
- **Excel bulk import** with upsert logic — updates existing SKUs, assigns new system SKUs to new entries
- **Sales registration** with product auto-fill, editable price (prompts to update master), payment method, and change calculation
- **Bulk sales import** with column-matching preview and downloadable error report for failed rows

### Phase 2 — Cash Flow & Analytics
- **Expense tracking** by category (Raw Materials, Salaries, Services, Rent, Maintenance, Others) with receipt photo upload
- **Sales dashboards** — drill-down bar charts (Year → Month → Week → Day)
- **Heatmaps** — sales density by day of week and hour of day
- **Period-over-period comparisons** and cash flow summary card (Total Sales − Total Expenses)

### Phase 3 — Stock & Recipe Management
- **Ingredient inventory** with minimum stock / reorder-point alerts
- **Scalable recipes** linking products to ingredients with automatic stock deduction on every confirmed sale
- **Waste / shrinkage** quick-entry with notes
- **Blind audit** — enter physical counts; system highlights discrepancies against theoretical stock

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router) |
| Styling | Tailwind CSS + shadcn/ui |
| Charts | Recharts |
| Forms & Validation | React Hook Form + Zod |
| State Management | Zustand |
| Data Tables | TanStack Table |
| Excel Import | SheetJS (xlsx) |
| Database | PostgreSQL via Supabase |
| ORM | Prisma |
| Authentication | NextAuth.js v5 |
| File Storage | Supabase Storage |
| Hosting | Vercel |
| CI/CD | GitHub Actions |
| Error Tracking | Sentry |

---

## Documentation

| Document | Description |
|---|---|
| [SETUP.md](./SETUP.md) | **Start here** — accounts, tooling, and environment setup before development |
| [context.md](./context.md) | Business context, requirements, rules, and visual identity |
| [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) | Database schema, phased delivery plan, and technical requirements |

---

## Getting Started

> Complete [SETUP.md](./SETUP.md) first. It covers account creation, environment variables, and tooling installation.

Once your environment is ready:

```bash
# Install dependencies
pnpm install

# Set up environment variables
cp .env.example .env.local
# Fill in values from your Supabase and Vercel projects

# Run database migrations
pnpm prisma migrate dev

# Seed with demo data
pnpm prisma db seed

# Start the development server
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### Key Scripts

| Script | Description |
|---|---|
| `pnpm dev` | Start development server |
| `pnpm build` | Production build |
| `pnpm lint` | Run ESLint |
| `pnpm typecheck` | Run TypeScript compiler check |
| `pnpm test` | Run test suite |
| `pnpm prisma migrate dev` | Apply pending migrations |
| `pnpm prisma studio` | Open Prisma visual DB browser |
| `pnpm prisma db seed` | Seed database with demo data |

---

## Project Structure

```
AlmasApp/
├── .github/
│   └── workflows/
│       └── ci.yml               # Lint + typecheck on every PR
├── .vscode/
│   └── settings.json            # Shared editor config
├── app/
│   ├── (auth)/                  # Login page
│   ├── (dashboard)/
│   │   ├── products/            # Phase 1 — product catalog
│   │   ├── sales/               # Phase 1 — sales registration & history
│   │   ├── expenses/            # Phase 2 — expense tracking
│   │   ├── analytics/           # Phase 2 — dashboards
│   │   └── inventory/           # Phase 3 — stock & recipes
│   └── api/
│       ├── products/
│       ├── sales/
│       ├── expenses/
│       └── ingredients/
├── components/
│   ├── ui/                      # shadcn/ui primitives
│   ├── forms/                   # Feature-specific form components
│   ├── charts/                  # Phase 2 — Recharts wrappers
│   └── tables/                  # TanStack Table instances
├── lib/
│   ├── db.ts                    # Prisma client singleton
│   ├── auth.ts                  # NextAuth configuration
│   ├── sku.ts                   # SKU generation logic
│   ├── excel.ts                 # SheetJS import/export utilities
│   └── validations/             # Zod schemas (one file per domain)
├── prisma/
│   ├── schema.prisma
│   ├── seed.ts
│   └── migrations/
├── public/
│   └── templates/               # Downloadable Excel import templates
├── context.md
├── IMPLEMENTATION_PLAN.md
├── SETUP.md
└── README.md
```

---

## Roles & Access

| Role | Permissions |
|---|---|
| **Employee** | Log sales, view product catalog |
| **Owner** | Full access: products, sales, expenses, analytics, inventory, audits |

Role assignment is managed by owners directly in the app. All routes are protected via NextAuth session; owner-only routes return `403` for employee sessions.

---

## Roadmap

- [x] Project documentation (README, context, implementation plan, setup guide)
- [ ] **Phase 1** — Product catalog + Sales MVP *(Weeks 1–5)*
- [ ] **Phase 2** — Cash flow + Analytics dashboards *(Weeks 6–9)*
- [ ] **Phase 3** — Inventory + Recipe management + Blind audit *(Weeks 10–14)*

See [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) for the full week-by-week delivery plan.

---

## License

Private — internal use only for Alma Pastelería & Bodegón.
