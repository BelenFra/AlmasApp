# Alma App — Implementation Plan

**Version:** 1.0
**Last updated:** 2026-04-17

> Before starting Week 1, complete the full environment setup in [SETUP.md](./SETUP.md).
> For business rules and requirements, see [context.md](./context.md).

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [Database Schema](#2-database-schema)
3. [Delivery Plan](#3-delivery-plan)
4. [Definition of Done](#4-definition-of-done)
5. [Testing Strategy](#5-testing-strategy)
6. [Branching & Deployment](#6-branching--deployment)
7. [Performance Targets](#7-performance-targets)
8. [Security Requirements](#8-security-requirements)

---

## 1. Technology Stack

### Frontend

| Concern | Library | Version | Rationale |
|---|---|---|---|
| Framework | Next.js (App Router) | 14.x | SSR, file-based routing, co-located API routes |
| Styling | Tailwind CSS | 3.x | Utility-first, easy brand theming |
| Component library | shadcn/ui | latest | Headless, accessible, Tailwind-native |
| Charts | Recharts | 2.x | React-native, supports drill-down and custom shapes |
| Forms | React Hook Form | 7.x | Performant, minimal re-renders |
| Validation | Zod | 3.x | Schema-first, shared between client and server |
| State management | Zustand | 4.x | Lightweight global state, no boilerplate |
| Data tables | TanStack Table | 8.x | Headless, supports virtual scroll, filtering, export |
| Excel parsing | SheetJS (xlsx) | 0.18.x | Client-side Excel read/write |

### Backend

| Concern | Library | Version | Rationale |
|---|---|---|---|
| API layer | Next.js Route Handlers | 14.x | Co-located with frontend; no separate server needed |
| ORM | Prisma | 5.x | Type-safe, schema-first, migration management |
| Authentication | NextAuth.js | 5.x | Built-in session handling, Supabase adapter |

### Infrastructure

| Concern | Service | Rationale |
|---|---|---|
| Database | Supabase (PostgreSQL 15) | Managed Postgres, RLS, Storage, generous free tier |
| File storage | Supabase Storage | Already in stack; no third-party service needed |
| Hosting | Vercel | Zero-config Next.js deployment, PR preview environments |
| CI/CD | GitHub Actions | Lint + type-check + tests on every pull request |
| Error tracking | Sentry | Client + server error capture, free tier |

---

## 2. Database Schema

> The schema below is the **authoritative reference**. Prisma's `schema.prisma` must mirror this exactly.
> Tables are ordered by dependency (no forward references in foreign keys).

```sql
-- ═══════════════════════════════════════════════
--  ENUMS
-- ═══════════════════════════════════════════════
CREATE TYPE business_unit    AS ENUM ('BAKERY', 'BISTRO');
CREATE TYPE unit_of_measure  AS ENUM ('UNIT', 'GRAMS', 'KG');
CREATE TYPE payment_method   AS ENUM ('CASH', 'TRANSFER', 'DEBIT', 'CREDIT', 'QR');
CREATE TYPE expense_category AS ENUM ('RAW_MATERIALS', 'SALARIES', 'SERVICES', 'RENT', 'MAINTENANCE', 'OTHERS');
CREATE TYPE movement_type    AS ENUM ('SALE_DEDUCTION', 'WASTE', 'PURCHASE', 'AUDIT_ADJUSTMENT');

-- ═══════════════════════════════════════════════
--  USERS  (must be created before referencing tables)
-- ═══════════════════════════════════════════════
CREATE TABLE users (
  id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  email      TEXT        NOT NULL UNIQUE,
  name       TEXT,
  role       TEXT        NOT NULL DEFAULT 'EMPLOYEE', -- EMPLOYEE | OWNER
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ═══════════════════════════════════════════════
--  PRODUCTS
-- ═══════════════════════════════════════════════
CREATE TABLE products (
  id            UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
  sku           TEXT            NOT NULL UNIQUE, -- system-generated: BKR-0001 / BST-0001
  name          TEXT            NOT NULL,
  business_unit business_unit   NOT NULL,
  price         NUMERIC(10,2)   NOT NULL,
  unit          unit_of_measure NOT NULL DEFAULT 'UNIT',
  description   TEXT,
  comments      TEXT,
  is_active     BOOLEAN         NOT NULL DEFAULT TRUE,
  created_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

-- Auto-update updated_at on any row change
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_updated_at
  BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- ═══════════════════════════════════════════════
--  SALES
-- ═══════════════════════════════════════════════
CREATE TABLE sales (
  id             UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
  business_unit  business_unit  NOT NULL,
  payment_method payment_method NOT NULL,
  client_name    TEXT,
  total_amount   NUMERIC(10,2)  NOT NULL,
  amount_paid    NUMERIC(10,2)  NOT NULL,
  change_given   NUMERIC(10,2)  GENERATED ALWAYS AS (amount_paid - total_amount) STORED,
  sale_date      DATE           NOT NULL DEFAULT CURRENT_DATE,
  created_by     UUID           REFERENCES users(id) ON DELETE SET NULL,
  created_at     TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE TABLE sale_items (
  id         UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  sale_id    UUID          NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
  product_id UUID          NOT NULL REFERENCES products(id),
  quantity   NUMERIC(10,3) NOT NULL,
  unit_price NUMERIC(10,2) NOT NULL, -- snapshot of price at time of sale
  subtotal   NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- ═══════════════════════════════════════════════
--  EXPENSES  (Phase 2)
-- ═══════════════════════════════════════════════
CREATE TABLE expenses (
  id            UUID             PRIMARY KEY DEFAULT gen_random_uuid(),
  business_unit business_unit    NOT NULL,
  category      expense_category NOT NULL,
  description   TEXT             NOT NULL,
  amount        NUMERIC(10,2)    NOT NULL,
  expense_date  DATE             NOT NULL DEFAULT CURRENT_DATE,
  receipt_url   TEXT,            -- Supabase Storage signed URL
  created_by    UUID             REFERENCES users(id) ON DELETE SET NULL,
  created_at    TIMESTAMPTZ      NOT NULL DEFAULT NOW()
);

-- ═══════════════════════════════════════════════
--  INGREDIENTS  (Phase 3)
-- ═══════════════════════════════════════════════
CREATE TABLE ingredients (
  id            UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT            NOT NULL,
  unit          unit_of_measure NOT NULL,
  stock_qty     NUMERIC(10,3)   NOT NULL DEFAULT 0,
  min_stock     NUMERIC(10,3)   NOT NULL DEFAULT 0, -- reorder point / alert threshold
  cost_per_unit NUMERIC(10,4),
  created_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

CREATE TRIGGER ingredients_updated_at
  BEFORE UPDATE ON ingredients
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- ═══════════════════════════════════════════════
--  RECIPES  (Phase 3)
-- ═══════════════════════════════════════════════
CREATE TABLE recipes (
  id         UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID          NOT NULL UNIQUE REFERENCES products(id),
  yield_qty  NUMERIC(10,3) NOT NULL DEFAULT 1, -- units this recipe batch produces
  notes      TEXT,
  created_at TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE TABLE recipe_items (
  id            UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  recipe_id     UUID          NOT NULL REFERENCES recipes(id) ON DELETE CASCADE,
  ingredient_id UUID          NOT NULL REFERENCES ingredients(id),
  quantity      NUMERIC(10,4) NOT NULL, -- ingredient qty needed per yield_qty
  UNIQUE (recipe_id, ingredient_id)
);

CREATE TRIGGER recipes_updated_at
  BEFORE UPDATE ON recipes
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- ═══════════════════════════════════════════════
--  STOCK MOVEMENTS  (Phase 3)
-- ═══════════════════════════════════════════════
CREATE TABLE stock_movements (
  id            UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  ingredient_id UUID          NOT NULL REFERENCES ingredients(id),
  movement_type movement_type NOT NULL,
  quantity      NUMERIC(10,3) NOT NULL, -- negative = outflow (deduction, waste)
  reference_id  UUID,                   -- sale_id | expense_id | audit_session_id
  notes         TEXT,
  created_by    UUID          REFERENCES users(id) ON DELETE SET NULL,
  created_at    TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- ═══════════════════════════════════════════════
--  AUDIT  (Phase 3)
-- ═══════════════════════════════════════════════
CREATE TABLE audit_sessions (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  conducted_by UUID        REFERENCES users(id) ON DELETE SET NULL,
  audit_date   DATE        NOT NULL DEFAULT CURRENT_DATE,
  notes        TEXT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_items (
  id               UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  audit_session_id UUID          NOT NULL REFERENCES audit_sessions(id) ON DELETE CASCADE,
  ingredient_id    UUID          NOT NULL REFERENCES ingredients(id),
  theoretical_qty  NUMERIC(10,3) NOT NULL,
  physical_qty     NUMERIC(10,3) NOT NULL,
  discrepancy      NUMERIC(10,3) GENERATED ALWAYS AS (physical_qty - theoretical_qty) STORED
);

-- ═══════════════════════════════════════════════
--  INDEXES
-- ═══════════════════════════════════════════════
CREATE INDEX idx_products_sku           ON products(sku);
CREATE INDEX idx_products_unit          ON products(business_unit);
CREATE INDEX idx_sales_date             ON sales(sale_date);
CREATE INDEX idx_sales_unit             ON sales(business_unit);
CREATE INDEX idx_sale_items_sale        ON sale_items(sale_id);
CREATE INDEX idx_sale_items_product     ON sale_items(product_id);
CREATE INDEX idx_expenses_date          ON expenses(expense_date);
CREATE INDEX idx_expenses_unit          ON expenses(business_unit);
CREATE INDEX idx_stock_movements_ing    ON stock_movements(ingredient_id);
CREATE INDEX idx_stock_movements_ref    ON stock_movements(reference_id);
```

---

## 3. Delivery Plan

### Phase 1 — MVP (Weeks 1–5)

#### Week 1 — Project Scaffolding
- [ ] Initialize Next.js 14 project with TypeScript + Tailwind CSS
- [ ] Configure ESLint, Prettier, and Husky pre-commit hooks
- [ ] Create `.env.example` and `.gitignore`
- [ ] Configure Prisma with Supabase dev database
- [ ] Run initial migrations (users, products, sales, sale_items, enums)
- [ ] Set up NextAuth.js v5 with email/password and Supabase adapter
- [ ] Create `.github/workflows/ci.yml` (lint + typecheck on PR)
- [ ] Deploy skeleton to Vercel; confirm build passes
- [ ] Apply brand theme in `tailwind.config.ts` (colors, fonts, border-radius)
- [ ] Create `lib/db.ts` Prisma singleton and `lib/auth.ts` NextAuth config

#### Week 2 — Product Catalog
- [ ] Product list page: search, filter by business unit, pagination
- [ ] Create/Edit product form with all fields (Zod validation)
- [ ] `lib/sku.ts`: auto-generation logic (`BKR-XXXX` / `BST-XXXX`)
- [ ] `updated_at` trigger in Postgres (from schema above)
- [ ] Product soft-delete via `is_active` toggle
- [ ] Excel template download endpoint (`GET /api/products/template`)
- [ ] Bulk import: column-preview modal before processing
- [ ] Bulk import: upsert logic (update if SKU exists; new system SKU if not)

#### Week 3 — Sales Registration
- [ ] New sale form: product selector with price auto-fill
- [ ] Price override flow: modal prompt "Update master price?"
- [ ] Payment method selector, client name, amount paid, change display
- [ ] Date field defaulting to today (editable)
- [ ] Business unit validation (all items must share the same unit)
- [ ] Sales list page with date-range filter and business unit filter
- [ ] Bulk sales import with column-matching preview (`POST /api/sales/import`)
- [ ] Error report: downloadable `.xlsx` of failed rows only

#### Week 4 — Polish & Testing
- [ ] Mobile-first responsive QA on all pages (375px baseline)
- [ ] Skeleton loaders on all data-fetching pages
- [ ] Role-based route guards: employee redirect on owner-only routes
- [ ] Unit tests: SKU generation, bulk import parser, change calculation
- [ ] Integration tests: `POST /api/sales`, `POST /api/products/import`
- [ ] Seed script (`prisma/seed.ts`) with realistic demo data (2 units, ~30 products, ~50 sales)

#### Week 5 — Phase 1 Release
- [ ] UAT session with business owners — document feedback
- [ ] Fix all P1 bugs from UAT
- [ ] Production deployment to Vercel + Supabase prod
- [ ] Smoke test critical paths on production
- [ ] Write internal user guide (Notion or in-app `/help` page)

---

### Phase 2 — Cash Flow & Analytics (Weeks 6–9)

#### Week 6 — Expense Module
- [ ] Expense create / edit / delete with all fields
- [ ] Receipt photo upload to Supabase Storage bucket `receipts` (5 MB max, images only)
- [ ] Expense list with filter by category, business unit, and date range
- [ ] `GET /api/expenses/summary` — aggregated totals for dashboard

#### Week 7 — Core Dashboards
- [ ] Cash flow summary card: Sales − Expenses, filterable by period and unit
- [ ] Bar chart component with Year → Month → Week → Day drill-down (Recharts)
- [ ] Period-over-period comparison component

#### Week 8 — Advanced Analytics
- [ ] Sales heatmap: day-of-week × hour-of-day density grid
- [ ] Top-N products by revenue table
- [ ] Export dashboard snapshot as CSV/Excel

#### Week 9 — Phase 2 Release
- [ ] UAT and bug fixes
- [ ] Slow query audit: check `EXPLAIN ANALYZE` on all dashboard queries
- [ ] Add missing indexes if needed

---

### Phase 3 — Stock & Recipes (Weeks 10–14)

#### Week 10 — Ingredient Inventory
- [ ] Ingredient CRUD with `stock_qty` and `min_stock`
- [ ] Low-stock alert: banner on dashboard when `stock_qty < min_stock`
- [ ] `GET /api/ingredients` with low-stock filter

#### Week 11 — Recipe Builder
- [ ] Recipe create/edit UI: link product to ingredient list with quantities
- [ ] `yield_qty` support and per-unit cost calculator
- [ ] Recipe read-only viewer

#### Week 12 — Automatic Stock Deduction
- [ ] On sale confirmation: trigger deduction chain via `recipe_items`
- [ ] Deduction formula: `sale_item.quantity × recipe_item.quantity / recipe.yield_qty`
- [ ] Write one `stock_movements` row per ingredient per sale
- [ ] Products without a recipe: skip deduction silently

#### Week 13 — Waste & Blind Audit
- [ ] Waste quick-entry: ingredient, quantity, notes → writes `WASTE` movement
- [ ] Blind audit flow: owner enters physical counts without seeing theoretical
- [ ] Audit results view: theoretical vs. physical with discrepancy per ingredient
- [ ] Audit history list

#### Week 14 — Phase 3 Release
- [ ] Full UAT across all three phases
- [ ] Load test: 50 concurrent users, 100 sales in 60 seconds
- [ ] Update all documentation to reflect any spec changes from UAT

---

## 4. Definition of Done

A task is **done** when all of the following are true:

- [ ] Feature works end-to-end on a real mobile device (375px)
- [ ] All Zod validations are applied on both client and API
- [ ] New API routes are protected by NextAuth session check
- [ ] Relevant unit or integration tests pass
- [ ] `pnpm lint` and `pnpm typecheck` pass with no errors
- [ ] PR reviewed and merged to `main`
- [ ] Vercel preview deployment is smoke-tested

---

## 5. Testing Strategy

| Layer | Tool | Scope |
|---|---|---|
| Unit | Vitest | Pure functions: SKU generation, import parsing, price calculation |
| Integration | Vitest + Prisma test DB | API route handlers against a real (dev) database |
| E2E | Playwright *(Phase 2+)* | Critical user journeys: login, create sale, view dashboard |
| Manual QA | — | Mobile device check after each weekly milestone |

Test files live next to the code they test (`*.test.ts` alongside `*.ts`).

---

## 6. Branching & Deployment

```
main  ──────────────────────────────────────────► production (Vercel auto-deploy)
  └── feature/xxx  ──► PR ──► CI pass ──► merge
  └── fix/xxx      ──► PR ──► CI pass ──► merge
```

- `main` is always deployable
- Every PR triggers a Vercel Preview deployment and CI checks
- No direct commits to `main`; all changes go through pull requests
- Releases are tagged: `v1.0.0` (Phase 1), `v2.0.0` (Phase 2), `v3.0.0` (Phase 3)

---

## 7. Performance Targets

| Metric | Target | Measured by |
|---|---|---|
| First Contentful Paint | < 1.5s on 4G | Vercel Analytics |
| Largest Contentful Paint | < 2.5s on 4G | Core Web Vitals |
| Time to Interactive | < 3s on 4G | Lighthouse |
| API list endpoints (p95) | < 200ms | Vercel Function logs |
| Dashboard queries (p95) | < 500ms | Supabase Query Performance |

---

## 8. Security Requirements

| Concern | Implementation |
|---|---|
| Authentication | All routes require active NextAuth session; unauthenticated requests return `401` |
| Authorization | Owner-only routes return `403` for employee sessions |
| Row-Level Security | Supabase RLS enabled on all tables (see [SETUP.md](./SETUP.md)) |
| Input validation | Zod schemas on every API route handler and form |
| File uploads | MIME type check + 5 MB size limit; files stored in private Supabase bucket |
| Secrets | No secrets in source code; all via `.env.local` / Vercel env vars |
| `SUPABASE_SERVICE_ROLE_KEY` | Never exposed client-side (`NEXT_PUBLIC_` prefix forbidden) |
