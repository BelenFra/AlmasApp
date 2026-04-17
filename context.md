# Alma App — Project Context

## Business Overview

**Alma App** is an internal management system for a family-owned food business operating two units:

| Unit | Type | Description |
|---|---|---|
| **Alma Pastelería** | Bakery | Pastries, cakes, baked goods |
| **Alma Bodegón** | Bistro | Savory dishes, beverages |

**Primary Users:**
- **Employees** — daily sales logging, basic product lookup
- **Owners** — full operational oversight: inventory, financials, analytics

---

## Core Problem Statement

The business needs a single, mobile-first tool to replace manual spreadsheets for:
1. Logging daily sales across two units
2. Tracking expenses and cash flow
3. Managing ingredient inventory with automatic deduction on sale
4. Detecting stock discrepancies via blind audits

---

## Phased Scope

### Phase 1 — MVP: Product Catalog & Sales
- Product management with auto-generated SKUs
- Bulk product import via Excel (upsert logic: update if SKU exists, create new if not)
- Manual and bulk sales registration
- Price override with optional master price update
- Bulk import with column preview + error report download

### Phase 2 — Cash Flow & Analytics
- Expense module with categories, business unit tagging, receipt photo uploads
- Sales dashboards: bar charts with drill-down (Year → Month → Week → Day)
- Heatmaps: sales density by day-of-week and hour-of-day
- Period-over-period comparisons
- Cash flow summary card: Total Sales − Total Expenses

### Phase 3 — Stock & Recipe Management
- Ingredient inventory with minimum stock / reorder alerts
- Scalable recipes linking products to ingredients
- Automatic stock deduction on every sale
- Waste/shrinkage quick-entry
- Blind audit: physical count vs. theoretical stock

---

## Business Rules

### Product Management
- SKU is system-generated; cannot be manually set on import
- `created_at` and `updated_at` auto-populate
- Business unit is required (Bakery | Bistro)

### Sales
- Selecting a product auto-fills price
- If price is edited: prompt "Would you like to update the master price in the database?"
- Payment methods: Cash, Transfer, Debit, Credit
- Date defaults to today but is editable
- Client name is optional

### Bulk Import — Sales
- Preview column matching before processing
- On error: "Successfully loaded [X] sales, but [Y] failed due to errors in columns: [Column Names]"
- Provide downloadable error-only table for correction

### Bulk Import — Products
- If SKU exists → update product fields
- If SKU is new in the file → ignore file SKU, generate system SKU

---

## Visual Identity

| Token | Value |
|---|---|
| Background | Off-white / light cream `#FAF8F3` |
| Primary | Light pastel green `#C8E6C9` |
| Secondary | Forest green `#2E7D32` (text, icons, primary actions) |
| Accent | Soft gold / ochre `#D4A017` (dividers, borders, highlights) |
| Typography | Serif for headers · Sans-serif for forms and data |
| Border radius | Large (≥ 12px) |
| Transitions | Smooth on all interactive elements |

**Design Principle:** Minimalist, warm, mobile-first.

---

## Data Relationships (Core)

```
Ingredients
    └── RecipeItems (quantity per ingredient)
            └── Recipes → Products
                            └── SaleItems
                                    └── Sales
```

Every confirmed sale automatically deducts proportional ingredient stock via the recipe link.

---

## Out of Scope (v1)
- Multi-location beyond Bakery + Bistro
- Customer-facing ordering or loyalty programs
- Accounting/tax integrations
- Multi-currency
