# Alma App — Implementation Plan

## Technology Stack

### Frontend
| Layer | Choice | Rationale |
|---|---|---|
| Framework | **Next.js 14** (App Router) | SSR + file-based routing, great for mobile-first PWA |
| UI library | **shadcn/ui** + Tailwind CSS | Headless components, easy theming, small bundle |
| Charts | **Recharts** | Drill-down bar charts + heatmaps with React bindings |
| Forms | **React Hook Form** + Zod | Schema validation, great DX |
| State | **Zustand** | Lightweight, no boilerplate |
| Tables | **TanStack Table** | Virtual scroll, filtering, export |
| Excel import | **SheetJS (xlsx)** | Client-side Excel parsing |

### Backend
| Layer | Choice | Rationale |
|---|---|---|
| API | **Next.js Route Handlers** (REST) | Co-located with frontend, no separate server |
| ORM | **Prisma** | Type-safe DB access, migrations, schema-first |
| Auth | **NextAuth.js v5** | Built-in session/role support |
| File uploads | **Uploadthing** or **Supabase Storage** | Receipt photo uploads |

### Database
| Layer | Choice | Rationale |
|---|---|---|
| Primary DB | **PostgreSQL** (via Supabase) | Relational, free tier, real-time, built-in auth |
| Hosting | **Supabase** | Managed Postgres + storage + RLS policies |

### Infrastructure & DevOps
| Layer | Choice | Rationale |
|---|---|---|
| Hosting | **Vercel** | Zero-config Next.js deployment, free tier |
| CI/CD | **GitHub Actions** | Lint, type-check, test on every PR |
| Env secrets | **Vercel Environment Variables** | Injected at build |
| Error tracking | **Sentry** (free tier) | Client + server error monitoring |

---

## Database Schema

```sql
-- ─────────────────────────────────────────────
--  ENUMS
-- ─────────────────────────────────────────────
CREATE TYPE business_unit   AS ENUM ('BAKERY', 'BISTRO');
CREATE TYPE unit_of_measure AS ENUM ('UNIT', 'GRAMS', 'KG');
CREATE TYPE payment_method  AS ENUM ('CASH', 'TRANSFER', 'DEBIT', 'CREDIT');
CREATE TYPE expense_category AS ENUM ('RAW_MATERIALS','SALARIES','SERVICES','RENT','MAINTENANCE','OTHERS');

-- ─────────────────────────────────────────────
--  PRODUCTS
-- ─────────────────────────────────────────────
CREATE TABLE products (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku           TEXT NOT NULL UNIQUE,          -- system-generated, e.g. BKR-0001
  name          TEXT NOT NULL,
  business_unit business_unit NOT NULL,
  price         NUMERIC(10,2) NOT NULL,
  unit          unit_of_measure NOT NULL DEFAULT 'UNIT',
  description   TEXT,
  comments      TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
--  SALES
-- ─────────────────────────────────────────────
CREATE TABLE sales (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_unit  business_unit NOT NULL,
  payment_method payment_method NOT NULL,
  client_name    TEXT,
  total_amount   NUMERIC(10,2) NOT NULL,
  amount_paid    NUMERIC(10,2) NOT NULL,
  change_given   NUMERIC(10,2) GENERATED ALWAYS AS (amount_paid - total_amount) STORED,
  sale_date      DATE NOT NULL DEFAULT CURRENT_DATE,
  created_by     UUID REFERENCES users(id),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE sale_items (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sale_id      UUID NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
  product_id   UUID NOT NULL REFERENCES products(id),
  quantity     NUMERIC(10,3) NOT NULL,
  unit_price   NUMERIC(10,2) NOT NULL,  -- price at time of sale (may differ from master)
  subtotal     NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- ─────────────────────────────────────────────
--  EXPENSES
-- ─────────────────────────────────────────────
CREATE TABLE expenses (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_unit business_unit NOT NULL,
  category      expense_category NOT NULL,
  description   TEXT NOT NULL,
  amount        NUMERIC(10,2) NOT NULL,
  expense_date  DATE NOT NULL DEFAULT CURRENT_DATE,
  receipt_url   TEXT,
  created_by    UUID REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
--  INGREDIENTS  (Phase 3)
-- ─────────────────────────────────────────────
CREATE TABLE ingredients (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL,
  unit          unit_of_measure NOT NULL,
  stock_qty     NUMERIC(10,3) NOT NULL DEFAULT 0,
  min_stock     NUMERIC(10,3) NOT NULL DEFAULT 0,  -- reorder point
  cost_per_unit NUMERIC(10,4),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
--  RECIPES  (Phase 3)
-- ─────────────────────────────────────────────
CREATE TABLE recipes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL UNIQUE REFERENCES products(id),
  yield_qty  NUMERIC(10,3) NOT NULL DEFAULT 1,  -- how many units this recipe produces
  notes      TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE recipe_items (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recipe_id     UUID NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
  ingredient_id UUID NOT NULL REFERENCES ingredients(id),
  quantity      NUMERIC(10,4) NOT NULL,  -- quantity per yield_qty
  UNIQUE (recipe_id, ingredient_id)
);

-- ─────────────────────────────────────────────
--  STOCK MOVEMENTS  (Phase 3)
-- ─────────────────────────────────────────────
CREATE TYPE movement_type AS ENUM ('SALE_DEDUCTION','WASTE','PURCHASE','AUDIT_ADJUSTMENT');

CREATE TABLE stock_movements (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ingredient_id UUID NOT NULL REFERENCES ingredients(id),
  movement_type movement_type NOT NULL,
  quantity      NUMERIC(10,3) NOT NULL,  -- negative = outflow
  reference_id  UUID,                    -- sale_id, expense_id, or audit_id
  notes         TEXT,
  created_by    UUID REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ─────────────────────────────────────────────
--  AUDIT SESSIONS  (Phase 3)
-- ─────────────────────────────────────────────
CREATE TABLE audit_sessions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conducted_by    UUID REFERENCES users(id),
  audit_date      DATE NOT NULL DEFAULT CURRENT_DATE,
  notes           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_items (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  audit_session_id   UUID NOT NULL REFERENCES audit_sessions(id) ON DELETE CASCADE,
  ingredient_id      UUID NOT NULL REFERENCES ingredients(id),
  theoretical_qty    NUMERIC(10,3) NOT NULL,
  physical_qty       NUMERIC(10,3) NOT NULL,
  discrepancy        NUMERIC(10,3) GENERATED ALWAYS AS (physical_qty - theoretical_qty) STORED
);

-- ─────────────────────────────────────────────
--  USERS (managed via NextAuth / Supabase Auth)
-- ─────────────────────────────────────────────
CREATE TABLE users (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email      TEXT NOT NULL UNIQUE,
  name       TEXT,
  role       TEXT NOT NULL DEFAULT 'EMPLOYEE',  -- EMPLOYEE | OWNER
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Phase 1 — MVP (Weeks 1–5)

### Week 1 — Project Setup
- [ ] Initialize Next.js 14 project with TypeScript + Tailwind CSS
- [ ] Configure ESLint, Prettier, Husky pre-commit hooks
- [ ] Set up Supabase project (dev + prod environments)
- [ ] Configure Prisma with Supabase PostgreSQL
- [ ] Run initial migrations (users, products, sales, sale_items)
- [ ] Set up NextAuth.js with Supabase adapter (email/password)
- [ ] Deploy skeleton to Vercel; configure environment variables
- [ ] Set up GitHub Actions (lint + type-check on PR)
- [ ] Apply brand theme in Tailwind config (colors, fonts, border-radius)

### Week 2 — Product Catalog
- [ ] Product list page with search, filter by business unit
- [ ] Create/Edit product form (all fields, Zod validation)
- [ ] SKU auto-generation logic (BKR-XXXX / BST-XXXX format)
- [ ] `updated_at` trigger in Postgres (auto-populate on update)
- [ ] Product soft-delete (is_active flag)
- [ ] Excel template download endpoint
- [ ] Bulk import: column preview modal before processing
- [ ] Bulk import: upsert logic (update if SKU exists, new SKU if new)

### Week 3 — Sales Registration
- [ ] New sale form: product selector with auto-fill price
- [ ] Price edit flow: pop-up "Update master price?" prompt
- [ ] Payment method, client name, amount paid, change calculation
- [ ] Date field (defaults to today, editable)
- [ ] Business unit assignment per sale
- [ ] Sales list page with date-range filter
- [ ] Bulk sales import with column-matching preview
- [ ] Error report: downloadable table of failed rows

### Week 4 — Polish & Testing
- [ ] Mobile-first responsive audit on all pages
- [ ] Skeleton loaders and optimistic UI updates
- [ ] Role-based access: employees see sales only; owners see everything
- [ ] Unit tests for SKU generation, bulk import logic, price update flow
- [ ] Integration tests for sale creation API
- [ ] Seed script with realistic demo data

### Week 5 — Phase 1 Release
- [ ] UAT with business owners
- [ ] Fix reported bugs
- [ ] Write user guide (internal Notion page or in-app help)
- [ ] Production deployment

---

## Phase 2 — Cash Flow & Analytics (Weeks 6–9)

### Week 6 — Expense Module
- [ ] Expense create/edit/delete with all category fields
- [ ] Receipt photo upload (Supabase Storage or Uploadthing)
- [ ] Expense list with filter by category, unit, date range

### Week 7 — Dashboards
- [ ] Cash flow summary card (Sales − Expenses, filterable by period + unit)
- [ ] Bar chart with drill-down: Year → Month → Week → Day (Recharts)
- [ ] Period-over-period comparison component

### Week 8 — Advanced Analytics
- [ ] Sales heatmap: day-of-week × hour-of-day density
- [ ] Top products by revenue
- [ ] Export dashboard data as CSV/Excel

### Week 9 — Phase 2 Release
- [ ] UAT and bug fixes
- [ ] Performance audit (slow queries, index review)

---

## Phase 3 — Stock & Recipes (Weeks 10–14)

### Week 10 — Ingredient Inventory
- [ ] Ingredient CRUD with stock_qty and min_stock
- [ ] Low-stock alert banner/notification

### Week 11 — Recipes
- [ ] Recipe builder UI: link product → ingredient quantities
- [ ] Yield quantity support
- [ ] Recipe viewer / cost calculator

### Week 12 — Automatic Stock Deduction
- [ ] On sale confirmation: trigger stock deduction via recipe_items
- [ ] stock_movements log per deduction

### Week 13 — Waste & Audit
- [ ] Waste quick-entry button with notes
- [ ] Blind audit: enter physical counts, system shows discrepancy
- [ ] Audit history viewer

### Week 14 — Phase 3 Release
- [ ] Full UAT
- [ ] Performance and load testing
- [ ] Documentation update

---

## Technical Requirements

### Development Environment
```
Node.js        >= 20.x LTS
pnpm           >= 9.x
PostgreSQL     >= 15  (via Supabase cloud, no local install needed)
Git            >= 2.40
```

### Key Environment Variables
```env
DATABASE_URL=postgresql://...          # Supabase connection string
DIRECT_URL=postgresql://...            # Direct connection for Prisma migrations
NEXTAUTH_SECRET=...
NEXTAUTH_URL=https://your-app.vercel.app
NEXT_PUBLIC_SUPABASE_URL=...
NEXT_PUBLIC_SUPABASE_ANON_KEY=...
SUPABASE_SERVICE_ROLE_KEY=...
UPLOADTHING_SECRET=...                 # Phase 2, receipt uploads
SENTRY_DSN=...                         # Error tracking
```

### Performance Targets
| Metric | Target |
|---|---|
| First Contentful Paint | < 1.5s on 4G |
| Time to Interactive | < 3s on 4G |
| Core Web Vitals (LCP) | < 2.5s |
| API response (list pages) | < 200ms p95 |

### Postgres Indexes (minimum)
```sql
CREATE INDEX idx_sales_date          ON sales(sale_date);
CREATE INDEX idx_sales_unit          ON sales(business_unit);
CREATE INDEX idx_sale_items_sale     ON sale_items(sale_id);
CREATE INDEX idx_sale_items_product  ON sale_items(product_id);
CREATE INDEX idx_expenses_date       ON expenses(expense_date);
CREATE INDEX idx_products_sku        ON products(sku);
CREATE INDEX idx_stock_movements_ing ON stock_movements(ingredient_id);
```

### Security
- All API routes protected with NextAuth session check
- Row-Level Security (RLS) enabled in Supabase
- Owners-only routes for expenses, dashboards, inventory
- File uploads: type + size validation (images only, max 5MB)
- Input sanitization via Zod on all form/API inputs

---

## Repository Structure

```
almas-app/
├── app/
│   ├── (auth)/              # Login page
│   ├── (dashboard)/
│   │   ├── products/        # Product catalog
│   │   ├── sales/           # Sales registration & list
│   │   ├── expenses/        # Phase 2
│   │   ├── analytics/       # Phase 2
│   │   └── inventory/       # Phase 3
│   └── api/
│       ├── products/
│       ├── sales/
│       ├── expenses/
│       └── ingredients/
├── components/
│   ├── ui/                  # shadcn/ui primitives
│   ├── forms/
│   ├── charts/              # Phase 2
│   └── tables/
├── lib/
│   ├── db.ts                # Prisma client
│   ├── auth.ts              # NextAuth config
│   ├── sku.ts               # SKU generation
│   ├── excel.ts             # SheetJS utilities
│   └── validations/         # Zod schemas
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── public/
│   └── templates/           # Excel import templates
├── .github/
│   └── workflows/
│       └── ci.yml
├── context.md
├── IMPLEMENTATION_PLAN.md
└── README.md
```
