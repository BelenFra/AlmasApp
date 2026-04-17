# Alma App

> Internal management system for **Alma Pastelería** (Bakery) and **Alma Bodegón** (Bistro).

A mobile-first web application that gives employees a fast way to log daily sales and gives owners complete operational oversight — from product catalog and cash flow to ingredient inventory and recipe-based stock control.

---

## Features

### Phase 1 — Product Catalog & Sales (MVP)
- **Product management** with auto-generated SKUs (BKR-XXXX / BST-XXXX), business unit tagging, and automatic timestamps
- **Excel bulk import** with upsert logic: updates existing SKUs, assigns new system SKUs to unknown entries
- **Sales registration** with product auto-fill, editable price (with optional master price update), payment method, change calculation, and editable date
- **Bulk sales import** with column-matching preview and downloadable error report

### Phase 2 — Cash Flow & Analytics
- **Expense tracking** by category (Raw Materials, Salaries, Services, Rent, Maintenance, Others) with receipt photo upload
- **Sales dashboards** — drill-down bar charts (Year → Month → Week → Day)
- **Heatmaps** — sales density by day of week and hour of day
- **Period-over-period comparisons** and a cash flow summary card (Sales − Expenses)

### Phase 3 — Stock & Recipe Management
- **Ingredient inventory** with minimum stock / reorder-point alerts
- **Scalable recipes** linking products to ingredients with automatic stock deduction on every sale
- **Waste / shrinkage** quick-entry
- **Blind audit** — physical counts vs. theoretical stock with discrepancy highlighting

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 (App Router), Tailwind CSS, shadcn/ui |
| Charts | Recharts |
| Forms & Validation | React Hook Form + Zod |
| Database | PostgreSQL via Supabase |
| ORM | Prisma |
| Auth | NextAuth.js v5 |
| File Uploads | Supabase Storage |
| Hosting | Vercel |
| CI/CD | GitHub Actions |
| Error Tracking | Sentry |

---

## Getting Started

### Prerequisites

```bash
node >= 20.x
pnpm >= 9.x
```

### Installation

```bash
# Clone the repository
git clone https://github.com/your-org/almas-app.git
cd almas-app

# Install dependencies
pnpm install

# Copy environment variables
cp .env.example .env.local
# Fill in your Supabase and NextAuth credentials

# Run database migrations
pnpm prisma migrate dev

# Seed with demo data
pnpm prisma db seed

# Start the development server
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### Environment Variables

| Variable | Description |
|---|---|
| `DATABASE_URL` | Supabase connection string (pooled) |
| `DIRECT_URL` | Supabase direct connection for migrations |
| `NEXTAUTH_SECRET` | Random secret for NextAuth |
| `NEXTAUTH_URL` | App URL (e.g. `https://almasapp.vercel.app`) |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase anon key |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role key (server only) |
| `SENTRY_DSN` | Sentry project DSN |

---

## Project Structure

```
almas-app/
├── app/
│   ├── (auth)/              # Login page
│   ├── (dashboard)/
│   │   ├── products/        # Product catalog
│   │   ├── sales/           # Sales registration & history
│   │   ├── expenses/        # Phase 2
│   │   ├── analytics/       # Phase 2
│   │   └── inventory/       # Phase 3
│   └── api/                 # REST route handlers
├── components/
│   ├── ui/                  # shadcn/ui primitives
│   ├── forms/
│   ├── charts/
│   └── tables/
├── lib/
│   ├── db.ts                # Prisma client singleton
│   ├── auth.ts              # NextAuth configuration
│   ├── sku.ts               # SKU generation utilities
│   └── validations/         # Zod schemas
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── public/
│   └── templates/           # Excel import templates
├── context.md               # Full project context & business rules
└── IMPLEMENTATION_PLAN.md   # Phased plan + database schema
```

---

## Database Overview

Core entity relationships:

```
Ingredients ──► RecipeItems ──► Recipes ──► Products
                                                └──► SaleItems ──► Sales
                                                                      └──► Expenses
```

Full schema with all tables, enums, indexes, and constraints is documented in [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md).

---

## Roles & Access

| Role | Access |
|---|---|
| **Employee** | Log sales, view products |
| **Owner** | Full access: products, sales, expenses, analytics, inventory |

---

## Visual Identity

| Token | Value |
|---|---|
| Background | Off-white / cream `#FAF8F3` |
| Primary | Light pastel green `#C8E6C9` |
| Secondary | Forest green `#2E7D32` |
| Accent | Soft gold `#D4A017` |
| Typography | Serif headers · Sans-serif forms |

---

## Roadmap

- [x] Repository setup and documentation
- [ ] **Phase 1** — Product catalog + Sales MVP
- [ ] **Phase 2** — Cash flow + Analytics dashboards
- [ ] **Phase 3** — Inventory + Recipe + Audit

See [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) for the detailed week-by-week plan.

---

## Contributing

This is an internal project for Alma Pastelería & Bodegón. For questions, contact the development team.
