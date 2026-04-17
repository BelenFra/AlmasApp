# Alma App — Environment Setup Guide

Complete this guide **before** starting Week 1 of the [Implementation Plan](./IMPLEMENTATION_PLAN.md). Every external account, tool, and configuration decision is made here once so the implementation can proceed without interruption.

---

## Table of Contents

1. [Accounts to Create](#1-accounts-to-create)
2. [Local Tooling](#2-local-tooling)
3. [VS Code Setup](#3-vs-code-setup)
4. [Repository Configuration](#4-repository-configuration)
5. [Supabase Project](#5-supabase-project)
6. [Vercel Project](#6-vercel-project)
7. [Sentry Project](#7-sentry-project)
8. [Environment Variables](#8-environment-variables)
9. [Git Conventions](#9-git-conventions)
10. [Pre-flight Checklist](#10-pre-flight-checklist)

---

## 1. Accounts to Create

Create these accounts before anything else. All have free tiers sufficient for development and early production.

| Service | URL | Purpose | Free Tier Limit |
|---|---|---|---|
| **GitHub** | github.com | Source control, CI/CD | Unlimited public repos |
| **Supabase** | supabase.com | PostgreSQL + Auth + Storage | 500 MB DB, 1 GB storage |
| **Vercel** | vercel.com | Frontend + API hosting | Unlimited personal projects |
| **Sentry** | sentry.io | Error tracking | 5,000 errors/month |

> Sign up for Vercel and Supabase with your GitHub account to simplify OAuth connections later.

---

## 2. Local Tooling

### Required

Install in this order:

#### Node.js (v20 LTS)
```bash
# Recommended: install via nvm to manage versions
# macOS / Linux
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20
nvm use 20

# Windows: download installer from https://nodejs.org (LTS version)
```

Verify:
```bash
node --version   # should print v20.x.x
```

#### pnpm (v9+)
```bash
npm install -g pnpm@latest
pnpm --version   # should print 9.x.x
```

#### Git (v2.40+)
```bash
# macOS
brew install git

# Windows: download from https://git-scm.com/download/win
git --version
```

Configure your identity (required for commits):
```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### Optional but Recommended

| Tool | Install | Purpose |
|---|---|---|
| **Prisma CLI** (global) | `pnpm add -g prisma` | Run migrations outside project |
| **Supabase CLI** | [docs.supabase.com/guides/cli](https://supabase.com/docs/guides/cli) | Local DB + seed management |

---

## 3. VS Code Setup

Install [Visual Studio Code](https://code.visualstudio.com/) and then add these extensions:

### Required Extensions

```
dbaeumer.vscode-eslint          ESLint
esbenp.prettier-vscode          Prettier
bradlc.vscode-tailwindcss       Tailwind CSS IntelliSense
prisma.prisma                   Prisma schema formatting + syntax
```

Install all at once via terminal:
```bash
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension bradlc.vscode-tailwindcss
code --install-extension prisma.prisma
```

### Recommended Extensions

```
christian-kohler.path-intellisense   Path autocomplete
usernamehw.errorlens                 Inline error display
eamodio.gitlens                      Git blame + history
```

### Workspace Settings

Create `.vscode/settings.json` in the project root:

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"]
  ],
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

---

## 4. Repository Configuration

### Clone and Initialize

```bash
git clone https://github.com/BelenFra/AlmasApp.git
cd AlmasApp
pnpm install
```

### Branch Protection (GitHub)

Go to **GitHub → Settings → Branches → Add rule** for the `main` branch:

- [x] Require a pull request before merging
- [x] Require status checks to pass (add `lint` and `typecheck` once CI is set up)
- [x] Do not allow bypassing the above settings

### GitHub Actions

The CI workflow (`.github/workflows/ci.yml`) runs on every PR. Create it during Week 1:

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test --if-present
```

---

## 5. Supabase Project

### Create Two Projects

Create separate projects for dev and production to avoid accidents:

| Project | Name suggestion | Use |
|---|---|---|
| Development | `almasapp-dev` | Local development and testing |
| Production | `almasapp-prod` | Live app only |

### Steps (repeat for each project)

1. Go to [supabase.com/dashboard](https://supabase.com/dashboard) → **New project**
2. Choose a strong database password — save it in a password manager
3. Select the closest region (e.g. South America for Argentina)
4. After creation: go to **Settings → Database** and copy:
   - **Connection string (pooled)** → `DATABASE_URL`
   - **Connection string (direct)** → `DIRECT_URL`
5. Go to **Settings → API** and copy:
   - **Project URL** → `NEXT_PUBLIC_SUPABASE_URL`
   - **anon public** key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - **service_role** key → `SUPABASE_SERVICE_ROLE_KEY` (never expose client-side)

### Enable Row-Level Security

After running migrations, enable RLS on all tables from the Supabase dashboard or via SQL:

```sql
ALTER TABLE products        ENABLE ROW LEVEL SECURITY;
ALTER TABLE sales           ENABLE ROW LEVEL SECURITY;
ALTER TABLE sale_items      ENABLE ROW LEVEL SECURITY;
ALTER TABLE expenses        ENABLE ROW LEVEL SECURITY;
ALTER TABLE ingredients     ENABLE ROW LEVEL SECURITY;
ALTER TABLE recipes         ENABLE ROW LEVEL SECURITY;
ALTER TABLE recipe_items    ENABLE ROW LEVEL SECURITY;
ALTER TABLE stock_movements ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_sessions  ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_items     ENABLE ROW LEVEL SECURITY;
```

### Storage Bucket

For Phase 2 receipt uploads, create a bucket in **Storage → New bucket**:

- Name: `receipts`
- Public: **No** (private, accessed via signed URLs)
- File size limit: 5 MB
- Allowed MIME types: `image/jpeg, image/png, image/webp`

---

## 6. Vercel Project

### Connect Repository

1. Go to [vercel.com/new](https://vercel.com/new)
2. Import the `AlmasApp` GitHub repository
3. Framework: **Next.js** (auto-detected)
4. Root directory: `.` (project root)

### Configure Environments

Vercel maintains three environments: **Production**, **Preview**, and **Development**.

- **Production** → uses `almasapp-prod` Supabase project
- **Preview** (auto-deploy on PRs) → uses `almasapp-dev` Supabase project

Add all environment variables from [Section 8](#8-environment-variables) to both environments via **Settings → Environment Variables**.

---

## 7. Sentry Project

1. Go to [sentry.io](https://sentry.io) → **Create Project** → select **Next.js**
2. Copy the **DSN** → `SENTRY_DSN`
3. Install the SDK during Week 1 setup:
   ```bash
   pnpm add @sentry/nextjs
   npx @sentry/wizard@latest -i nextjs
   ```

---

## 8. Environment Variables

### `.env.example` (commit this file)

```env
# ── Supabase ────────────────────────────────────────────────────────────────
DATABASE_URL=postgresql://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres?pgbouncer=true
DIRECT_URL=postgresql://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:5432/postgres

NEXT_PUBLIC_SUPABASE_URL=https://[project-ref].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# ── Auth (NextAuth.js) ───────────────────────────────────────────────────────
NEXTAUTH_SECRET=                    # openssl rand -base64 32
NEXTAUTH_URL=http://localhost:3000  # change to prod URL in Vercel

# ── Monitoring ───────────────────────────────────────────────────────────────
SENTRY_DSN=https://[key]@o[org].ingest.sentry.io/[project]
NEXT_PUBLIC_SENTRY_DSN=https://[key]@o[org].ingest.sentry.io/[project]
```

### `.env.local` (never commit — add to `.gitignore`)

Copy `.env.example` to `.env.local` and fill in real values:

```bash
cp .env.example .env.local
```

Generate `NEXTAUTH_SECRET`:
```bash
openssl rand -base64 32
```

---

## 9. Git Conventions

### Branch Naming

```
feature/<short-description>     New functionality
fix/<short-description>         Bug fixes
chore/<short-description>       Config, deps, tooling
docs/<short-description>        Documentation only
```

Examples:
```
feature/product-bulk-import
fix/sku-generation-collision
chore/setup-eslint-config
```

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body]
```

| Type | When to use |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Tooling, deps, config |
| `docs` | Documentation |
| `test` | Adding or updating tests |
| `refactor` | Code restructure without behavior change |

Examples:
```
feat(products): add SKU auto-generation on create
fix(sales): correct change calculation for partial payments
chore: configure Prettier and ESLint
```

### Pull Request Flow

```
feature-branch  →  PR with review  →  main  →  auto-deploy to Vercel
```

- PRs require at least one reviewer (owner or lead dev)
- CI must pass before merge
- Squash merge to keep `main` history clean

---

## 10. Pre-flight Checklist

Complete all items before starting Week 1 of the implementation plan:

### Accounts
- [ ] GitHub account created and repository cloned
- [ ] Supabase `almasapp-dev` project created
- [ ] Supabase `almasapp-prod` project created
- [ ] Vercel project connected to GitHub repository
- [ ] Sentry project created

### Local Environment
- [ ] Node.js v20 installed (`node --version`)
- [ ] pnpm v9+ installed (`pnpm --version`)
- [ ] Git v2.40+ installed and user identity configured
- [ ] VS Code with all required extensions installed
- [ ] `.vscode/settings.json` created

### Repository
- [ ] `.env.local` created with all variables filled in
- [ ] `.env.example` committed (no real secrets)
- [ ] `.gitignore` includes `.env.local` and `.env`
- [ ] Branch protection enabled on `main`
- [ ] `pnpm install` runs without errors

### Supabase
- [ ] Both Supabase projects (dev + prod) are live
- [ ] All connection strings and API keys copied to `.env.local`
- [ ] Storage bucket `receipts` created in dev project
- [ ] RLS will be enabled after first migration (Week 1 task)

### Vercel
- [ ] Environment variables added for both Production and Preview environments
- [ ] Test deployment triggered (push to main → check Vercel dashboard)

Once all boxes are checked, proceed to **[Week 1 of the Implementation Plan](./IMPLEMENTATION_PLAN.md#phase-1--mvp-weeks-15)**.
