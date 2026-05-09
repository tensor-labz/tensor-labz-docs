# Admin Panel — Overview

The admin panel is a full CMS for managing all website content. It lives at `/admin/*` and is protected by Supabase Auth.

---

## Route map

```mermaid
flowchart LR
    Root["/admin"] --> Overview["AdminOverview\nDashboard + charts"]
    Root --> Users["/admin/users\nAdminUsers"]
    Root --> Billing["/admin/billing\nAdminBilling"]
    Root --> Settings["/admin/settings\nAdminSettings"]
    Root --> Module["/admin/:module\nAdminDataTable\nlist + search + sort"]
    Module --> Record["/admin/:module/:id\nAdminCrudForm\nedit record"]
    Module --> New["/admin/:module/new\nAdminCrudForm\ncreate record"]

    style Overview fill:#6366f1,color:#fff
    style Users fill:#6366f1,color:#fff
    style Billing fill:#6366f1,color:#fff
    style Settings fill:#6366f1,color:#fff
    style Module fill:#0891b2,color:#fff
    style Record fill:#0891b2,color:#fff
    style New fill:#22c55e,color:#fff
```

!!! important "Route ordering"
`users`, `billing`, and `settings` must be declared **before** the `:module` catch-all in `AdminDashboard.tsx`. Without this, React Router matches them as module names.

---

## Auth guard flow

```mermaid
flowchart TD
    Load(["Page load"]) --> Listen["providers.tsx\nsupabase.auth.onAuthStateChange"]
    Listen --> HasSession{Session\nexists?}
    HasSession -->|"Yes"| SetUser["setUser(user)\ninitialized = true"]
    HasSession -->|"No"| SetNull["setUser(null)\ninitialized = true"]
    SetUser --> PR["ProtectedRoute renders"]
    SetNull --> PR
    PR --> IsInit{initialized\n= true?}
    IsInit -->|"No"| Blank["render null\n(blank, no flash)"]
    IsInit -->|"Yes"| IsAuth{user\nnot null?}
    IsAuth -->|"Yes"| AdminUI["✅ Render /admin/* children"]
    IsAuth -->|"No"| Login["🔀 Navigate to /login\n?from=original URL"]

    style SetUser fill:#22c55e,color:#fff
    style SetNull fill:#f97316,color:#fff
    style AdminUI fill:#3b82f6,color:#fff
    style Login  fill:#ef4444,color:#fff
```

---

## Settings tabs

```mermaid
flowchart LR
    S["/admin/settings"] --> G["⚙ General\nSite name, emails"]
    S --> A["🎨 Appearance\nTheme, accent colour"]
    S --> I["🔌 Integrations\nSupabase, AWS config"]
    S --> Sec["🔒 Security\nPassword, 2FA, sessions"]
    S --> T["📊 Tables\nColumn config → table_config"]
    S --> F["📋 Forms\nField builder → form_config"]
```

---

## Key source files

```
src/
  pages/
    AdminDashboard.tsx       ← layout shell (sidebar + route switch)
    Login.tsx                ← email + password sign-in
  components/
    ProtectedRoute.tsx       ← auth guard
  features/admin/
    config/
      modules.tsx            ← MODULES array — all module + field definitions
    components/
      AdminSidebar.tsx       ← desktop sticky + mobile drawer nav
      AdminOverview.tsx      ← dashboard charts (Recharts)
      AdminDataTable.tsx     ← list table + search + sort + column config
      AdminCrudForm.tsx      ← dynamic form with all 12 field types
      AdminSettings.tsx      ← settings tabs + Tables/Forms builders
  lib/
    supabase.ts              ← Supabase client singleton
    imageUpload.ts           ← upload / replace / delete + moduleFolder()
  store/
    authSlice.ts             ← login / signOut thunks + user state
  app/
    providers.tsx            ← onAuthStateChange listener
```
