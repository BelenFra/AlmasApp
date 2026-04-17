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

### Gestión de Productos

> Campos del formulario: **SKU** · **Nombre** · **Unidad de negocio** · **Precio** · **Unidad de medida** · **Descripción** · **Comentarios**

| Regla | Detalle |
|---|---|
| Formato de SKU | Generado por el sistema. Pastelería: `BKR-XXXX`. Bodegón: `BST-XXXX`. Secuencial. |
| Inmutabilidad del SKU | El SKU no puede modificarse luego de la creación. No se acepta desde importaciones Excel. |
| Fechas automáticas | **Fecha de creación** y **Última edición** se generan automáticamente; no son editables por el usuario. |
| Unidad de negocio | Obligatoria en cada producto. No puede modificarse luego de la creación. |
| Baja lógica | Los productos nunca se eliminan físicamente. Se desactivan mediante `is_active = false`. |

### Registro de Ventas

> Campos del formulario: **Producto** · **Precio unitario** · **Cantidad** · **Método de pago** · **Nombre del cliente** · **Monto abonado** · **Vuelto** · **Fecha** · **Unidad de negocio**

| Regla | Detalle |
|---|---|
| Precio automático | Al seleccionar un producto, el **precio unitario** se completa automáticamente desde el catálogo maestro. |
| Edición de precio | El usuario puede editar el precio. El sistema muestra el alerta: *"¿Desea actualizar el precio maestro en la base de datos?"* — **Sí** actualiza el producto; **No** registra la venta con el precio personalizado únicamente. |
| Métodos de pago | Efectivo, Transferencia, Débito, Crédito, QR. Uno por venta. |
| Vuelto | Se calcula automáticamente: `monto_abonado − total_venta`. |
| Nombre del cliente | Opcional. |
| Fecha | Por defecto es el día de hoy. Editable (permite ingresar fechas anteriores para correcciones). |
| Unidad de negocio | Heredada de los productos de la venta; todos deben pertenecer a la misma unidad. |

### Importación Masiva — Productos

| Escenario | Comportamiento |
|---|---|
| El SKU del archivo coincide con un producto existente | Actualiza todos los campos excepto SKU, unidad de negocio y fecha de creación. |
| El SKU del archivo no existe en el sistema | Ignora el SKU del archivo; genera un nuevo SKU del sistema. |
| Falta un campo obligatorio | Omite la fila; la incluye en el reporte de errores. |

### Importación Masiva — Ventas

| Paso | Comportamiento |
|---|---|
| Pre-importación | Muestra un modal de previsualización de columnas; el usuario debe confirmar antes de procesar. |
| Éxito total | *"Se cargaron [X] ventas exitosamente."* |
| Éxito parcial | *"Se cargaron [X] ventas exitosamente, pero [Y] fallaron por errores en las columnas: [Nombres de Columnas]."* |
| Salida de errores | Tabla descargable con únicamente las filas con errores para su corrección y re-importación. |

### Descuento de Stock (Fase 3)

- El descuento se activa al confirmar la venta, no al agregar un ítem
- Cantidad descontada = `cantidad_vendida × cantidad_receta / rendimiento_receta`
- Si el producto no tiene receta asociada, no se realiza ningún descuento
- Todos los descuentos quedan registrados en `stock_movements` con `reference_id = sale_id`

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

## 11. Glosario

| Término | Etiqueta en pantalla | Definición |
|---|---|---|
| **SKU** | SKU | Código único generado por el sistema para identificar cada producto. |
| **Unidad de negocio** | Unidad de negocio | Una de las dos unidades operativas: Pastelería (`BAKERY`) o Bodegón (`BISTRO`). |
| **Upsert** | — | Insertar o actualizar. En la importación de productos: actualiza si el SKU existe, crea si es nuevo. |
| **Precio maestro** | Precio | El precio canónico almacenado en el producto, usado para completar automáticamente las ventas. |
| **Rendimiento de receta** | Rendimiento | Cantidad de unidades vendibles que produce un lote de receta (ej. una torta = 1 unidad). |
| **Punto de reorden** | Stock mínimo | El umbral `min_stock` a partir del cual se activa la alerta de stock bajo. |
| **Auditoría ciega** | Auditoría de inventario | El propietario ingresa conteos físicos sin ver los valores teóricos esperados, para evitar sesgo. |
| **Stock teórico** | Stock teórico | Cantidad calculada por el sistema en base a compras menos descuentos por ventas menos mermas. |
| **Flujo de caja** | Flujo de caja | Total Ventas − Total Gastos para un período y/o unidad de negocio determinados. |
| **Merma** | Merma / Desperdicio | Pérdida de ingredientes por rotura, vencimiento u otros motivos no relacionados con ventas. |
