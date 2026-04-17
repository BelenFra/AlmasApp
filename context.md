# Alma App — Project Context

**Version:** 1.0
**Last updated:** 2026-04-17
**Status:** Pre-development — documentation phase complete

---

## Table of Contents

1. [Business Overview](#1-business-overview)
2. [Problem Statement](#2-problem-statement)
3. [Users & Roles](#3-users--roles)
4. [Phased Scope](#4-phased-scope)
5. [Business Rules](#5-business-rules)
6. [Visual Identity](#6-visual-identity)
7. [Data Relationships](#7-data-relationships)
8. [Key Decisions & Rationale](#8-key-decisions--rationale)
9. [Assumptions](#9-assumptions)
10. [Out of Scope](#10-out-of-scope)
11. [Glossary](#11-glossary)

---

## 1. Business Overview

**Alma App** is an internal management system for a family-owned food business operating two units:

| Unit | Type | Products |
|---|---|---|
| **Alma Pastelería** | Bakery | Pastries, cakes, baked goods |
| **Alma Bodegón** | Bistro | Savory dishes, beverages |

Both units share a single application but maintain separate product catalogs, sales records, expense tracking, and inventory. Owners have visibility across both units simultaneously.

---

## 2. Problem Statement

The business currently relies on manual spreadsheets for daily sales logging, expense tracking, and rough stock estimates. This causes:

- No single source of truth for sales across both units
- Inability to calculate real cash flow without manual cross-referencing
- No visibility into ingredient consumption or waste
- Stock counts that drift from reality with no way to detect discrepancies

**Goal:** Replace spreadsheets with a single, mobile-first tool that works reliably in a busy kitchen/bakery environment — fast to load, simple to operate, and accessible on any phone without installation.

---

## 3. Users & Roles

| Role | Primary Tasks | Access Level |
|---|---|---|
| **Employee** | Log daily sales, look up product prices | Sales entry, product read-only |
| **Owner** | Review financials, manage catalog, oversee inventory | Full access to all modules |

There are no more than ~10 users total. Authentication is email + password. Owners assign roles directly in the app.

---

## 4. Phased Scope

### Phase 1 — MVP: Product Catalog & Sales *(Weeks 1–5)*
Core operations needed to go live and replace spreadsheets immediately.

- Product management with auto-generated SKUs
- Bulk product import via Excel (upsert logic)
- Manual and bulk sales registration
- Price override with optional master price update prompt
- Bulk import with column-match preview and downloadable error report

### Phase 2 — Cash Flow & Analytics *(Weeks 6–9)*
Financial visibility for owners.

- Expense module with category tagging, business unit tagging, and receipt photo upload
- Sales dashboards: bar charts with drill-down (Year → Month → Week → Day)
- Heatmaps: sales density by day-of-week and hour-of-day
- Period-over-period comparisons (e.g. April 1–17 vs March 1–17)
- Cash flow summary card: Total Sales − Total Expenses

### Phase 3 — Stock & Recipe Management *(Weeks 10–14)*
Operational control of ingredient usage.

- Ingredient inventory with minimum stock / reorder-point alerts
- Scalable recipes linking products to ingredients
- Automatic stock deduction on every confirmed sale
- Waste/shrinkage quick-entry with notes
- Blind audit: owner enters physical counts, system shows discrepancy vs. theoretical

---

## 5. Business Rules

### Product Management

| Rule | Detail |
|---|---|
| SKU format | System-generated. Bakery: `BKR-XXXX`. Bistro: `BST-XXXX`. Sequential. |
| SKU immutability | SKU cannot be changed after creation. Not accepted from Excel imports. |
| Timestamps | `created_at` and `updated_at` auto-populate; never editable by users. |
| Business unit | Required on every product. Cannot be changed after creation. |
| Soft delete | Products are never hard-deleted. Deactivated via `is_active = false`. |

### Sales Registration

| Rule | Detail |
|---|---|
| Price auto-fill | Selecting a product auto-fills the unit price from the master catalog. |
| Price override | User may edit price. System then prompts: *"Would you like to update the master price in the database?"* — Yes updates the product; No records the sale at the custom price only. |
| Payment methods | Cash, Transfer, Debit, Credit. One per sale. |
| Change | Calculated automatically: `amount_paid − total_amount`. |
| Client name | Optional. |
| Date | Defaults to today. Editable (allows backdating for corrections). |
| Business unit | Inherited from the products in the sale; must all belong to the same unit. |

### Bulk Import — Products

| Scenario | Behavior |
|---|---|
| SKU in file matches existing product | Update all fields except SKU, business unit, and created_at |
| SKU in file does not exist | Ignore the file's SKU; generate a new system SKU |
| Required field missing | Skip row; include in error report |

### Bulk Import — Sales

| Step | Behavior |
|---|---|
| Pre-import | Show column-matching preview modal; user must confirm before processing |
| On success | "Successfully loaded [X] sales" |
| On partial failure | "Successfully loaded [X] sales, but [Y] failed due to errors in columns: [Column Names]" |
| Error output | Downloadable table containing only the failed rows for correction and re-import |

### Stock Deduction (Phase 3)

- Deduction is triggered on sale confirmation, not on item add
- Deduction quantity = `sale_item.quantity × recipe_item.quantity / recipe.yield_qty`
- If a product has no recipe, no deduction occurs
- All deductions are written to `stock_movements` with `reference_id = sale_id`

---

## 6. Visual Identity

| Token | Hex | Usage |
|---|---|---|
| Background | `#FAF8F3` | Page background, off-white/cream |
| Primary | `#C8E6C9` | Cards, secondary buttons |
| Secondary | `#2E7D32` | Text, icons, primary action buttons |
| Accent | `#D4A017` | Dividers, input borders, highlights |
| Error | `#D32F2F` | Validation errors, destructive actions |
| Success | `#388E3C` | Confirmations, positive indicators |

**Typography:**
- Headers: Serif (e.g. Playfair Display) — elegant, boutique feel
- Body / forms / data: Sans-serif (e.g. Inter) — clean and readable

**Design Principles:**
- Mobile-first: all layouts designed for 375px width first
- Border radius: ≥ 12px on all cards and inputs for a friendly feel
- Transitions: smooth (150–200ms ease) on all interactive elements
- Minimalist: no unnecessary decoration; whitespace is intentional

---

## 7. Data Relationships

```
users
 └── sales (created_by)
 └── expenses (created_by)
 └── stock_movements (created_by)
 └── audit_sessions (conducted_by)

products
 └── sale_items
 └── recipes
       └── recipe_items
             └── ingredients
                   └── stock_movements

sales
 └── sale_items → products → recipes → ingredients (stock deduction chain)

expenses (standalone, linked to business_unit)

audit_sessions
 └── audit_items → ingredients
```

Every confirmed sale triggers the following chain:
`sale confirmed → sale_items → recipe_items → stock_movements (SALE_DEDUCTION)`

---

## 8. Key Decisions & Rationale

| Decision | Choice | Reason |
|---|---|---|
| Single app for two units | Yes, one codebase | Small team, same owners, shared infrastructure |
| File uploads | Supabase Storage | Already in stack; avoids a third-party service |
| No offline mode | In scope for v1 | Business has stable WiFi; PWA offline adds significant complexity |
| Soft delete for products | `is_active` flag | Historical sales must retain valid product references |
| Sale price snapshot | `unit_price` on `sale_items` | Master price changes must not retroactively alter past revenue |
| Monorepo (Next.js API routes) | Chosen over separate backend | Team size doesn't justify a separate service; simpler deployment |

---

## 9. Assumptions

- All users have smartphones with a modern mobile browser (Chrome/Safari)
- Internet connectivity is available at both locations during operating hours
- The business operates in a single currency (Argentine Peso, ARS)
- There are no more than ~500 active products at any given time
- Sales volume is under 200 transactions per day per unit
- Owners will manage user accounts manually (no self-registration)

---

## 10. Out of Scope (v1)

- Customer-facing ordering, reservations, or loyalty programs
- Integration with external accounting or tax software (e.g. AFIP)
- Multi-currency support
- Locations beyond Bakery + Bistro
- SMS / push notifications
- Offline mode / PWA service worker caching
- Point-of-sale hardware integration (printers, cash drawers)

---

## 11. Glossary

| Term | Definition |
|---|---|
| **SKU** | Stock Keeping Unit. A unique identifier assigned by the system to each product. |
| **Business Unit** | One of the two operating units: Bakery (`BAKERY`) or Bistro (`BISTRO`). |
| **Upsert** | Insert or update. On product import: update if SKU exists, create if new. |
| **Master price** | The canonical price stored on the product record, used to auto-fill sales. |
| **Yield qty** | How many sellable units a single recipe batch produces (e.g. a cake recipe yields 1 unit). |
| **Reorder point** | The `min_stock` threshold at which a low-stock alert is triggered. |
| **Blind audit** | A stock count process where the user enters physical quantities without seeing the theoretical expected values, to avoid bias. |
| **Theoretical stock** | The quantity the system calculates based on purchases minus sales deductions minus waste. |
| **Cash flow** | Total Sales − Total Expenses for a given period and/or business unit. |
