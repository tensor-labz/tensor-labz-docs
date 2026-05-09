# Tensor Labz — Developer Documentation

Welcome to the complete developer documentation for the **Tensor Labz** website platform.

---

## System at a glance

```mermaid
flowchart TB
    subgraph FE ["Frontend (Browser)"]
        direction LR
        Public["Public Pages\n/ · /services · /projects\n/about · /contact"]
        Admin["Admin Panel\n/admin/*\n(Supabase Auth protected)"]
    end

    subgraph SUPA ["Supabase (PostgreSQL)"]
        DB["Content Tables\nhero · services · projects\nabout · about_media · contact\nsocial · company_info"]
        Config["Config Tables\ntable_config · form_config"]
        Auth["Supabase Auth\nJWT HS256"]
    end

    subgraph AWS ["AWS (eu-north-1)"]
        Lambda["Lambda\ntensor-labz-image-handler\nPresigned URL generator"]
        S3["S3 Bucket\ntensor-labz-store\nHome/Hero · Insights\nprojects/{slug} · about-us"]
    end

    subgraph DEPLOY ["Hosting"]
        Firebase["Firebase Hosting\nStaging"]
        Amplify["AWS Amplify\nProduction"]
    end

    Public  -- "read data" --> DB
    Admin   -- "CRUD" --> DB
    Admin   -- "read/write config" --> Config
    Admin   -- "login" --> Auth
    Auth    -- "JWT" --> Admin
    Admin   -- "upload request + JWT" --> Lambda
    Lambda  -- "presigned PUT URL" --> Admin
    Admin   -- "PUT binary (direct)" --> S3
    FE      -. "built & served by" .-> Firebase
    FE      -. "built & served by" .-> Amplify
```

---

## Tech stack

| Layer          | Technology                                      |
| -------------- | ----------------------------------------------- |
| Frontend       | React 18 + TypeScript + Vite + Tailwind CSS     |
| Animations     | Framer Motion (`motion/react`) + Three.js       |
| State          | Redux Toolkit                                   |
| CMS / Database | Supabase (PostgreSQL)                           |
| Auth           | Supabase Auth (JWT HS256)                       |
| Image Storage  | AWS S3 (`tensor-labz-store`, eu-north-1)        |
| Image Service  | AWS Lambda + API Gateway                        |
| Staging        | Firebase Hosting                                |
| Production     | AWS Amplify (auto-deploy from `main`)           |

---

## Quick links

| Resource             | Link |
| -------------------- | ---- |
| Frontend repo        | [tensor-labz/tensor-labz-website](https://github.com/tensor-labz/tensor-labz-website) |
| Lambda repo          | [tensor-labz/tensor-labz-image-lambda](https://github.com/tensor-labz/tensor-labz-image-lambda) |
| Supabase dashboard   | [rsxbmgusdiilcajuoxmk.supabase.co](https://supabase.com/dashboard/project/rsxbmgusdiilcajuoxmk) |
| AWS Console          | [eu-north-1 Lambda](https://eu-north-1.console.aws.amazon.com/lambda/home?region=eu-north-1) |
| Staging URL          | [tensor-labz-website.web.app](https://tensor-labz-website.web.app) |

---

## Branch strategy

```
feat/* ──► dev ──► staging ──► main
                     │              │
               Firebase          AWS Amplify
               (staging QA)      (production)
```

All development happens on `feat/*` branches → `dev`. When ready for QA, merge `dev → staging` (deploys to Firebase). When approved, merge `staging → main` (Amplify auto-deploys to production).

---

## Documentation sections

- **[Setup](setup/prerequisites.md)** — install tools, configure env vars, run locally
- **[Architecture](architecture/overview.md)** — full system diagram and data flow
- **[Supabase](supabase/schema.md)** — database schema, RLS, auth
- **[Lambda](lambda/overview.md)** — image service: how it works, how to deploy
- **[Admin Panel](admin/overview.md)** — CMS admin, modules, dynamic config
- **[Guides](guides/s3-upload-urls.md)** — S3 upload & URL workflow, admin CRUD operations
- **[Features](features/about-us.md)** — feature-level docs (About Us page, media gallery)
- **[Deployment](deployment/staging.md)** — staging (Firebase) and production (Amplify)
