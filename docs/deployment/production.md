# Production Deployment — AWS Amplify

Production deploys automatically when code is pushed to the `main` branch.

| Item          | Value                   |
| ------------- | ----------------------- |
| Trigger       | Push to `main` branch   |
| Build command | `npm run build`         |
| Output        | `dist/`                 |
| CDN           | AWS CloudFront (global) |

---

## Branch strategy

```mermaid
gitGraph
   commit id: "initial setup"
   branch staging
   checkout staging
   branch feat/hero
   checkout feat/hero
   commit id: "feat: hero section"
   checkout staging
   merge feat/hero id: "PR: feat → staging"
   branch fix/mobile-nav
   checkout fix/mobile-nav
   commit id: "fix: mobile nav"
   checkout staging
   merge fix/mobile-nav id: "PR: fix → staging"
   branch main
   checkout main
   merge staging id: "PR: staging → main 🚀"
```

---

## Deploy pipeline

```mermaid
flowchart LR
    subgraph Dev["Developer machine"]
        Code["code changes\non a feature branch"]
    end
    subgraph GitHub
        FeatB["feature branch\nfix/* · feat/* · refactor/*"]
        StagingB["staging branch"]
        MainB["main branch"]
    end
    subgraph Hosting
        Firebase["Firebase Hosting\ntensor-labz-website.web.app\n(staging QA)"]
        Amplify["AWS Amplify\ntensorlabz.com\n(production)"]
    end

    Code -- "push" --> FeatB
    FeatB -- "PR merge" --> StagingB
    StagingB -- "webhook" --> Firebase
    StagingB -- "PR merge (after QA)" --> MainB
    MainB -- "webhook" --> Amplify
```

---

## Amplify environment variables

Add these in **Amplify Console → App → Environment variables** before the first build:

```
VITE_SUPABASE_URL
VITE_SUPABASE_ANON_KEY
VITE_IMAGE_LAMBDA_URL
VITE_CDN_URL
VITE_S3_BUCKET
VITE_S3_REGION
```

Update via CLI:

```bash
aws --profile tensor amplify update-app \
  --app-id <AMPLIFY_APP_ID> \
  --environment-variables \
    VITE_SUPABASE_URL=https://rsxbmgusdiilcajuoxmk.supabase.co,\
    VITE_SUPABASE_ANON_KEY=sb_publishable_...,\
    VITE_IMAGE_LAMBDA_URL=https://ewf03ybvmc.execute-api.eu-north-1.amazonaws.com,\
    VITE_CDN_URL=https://tensor-labz-store.s3.eu-north-1.amazonaws.com,\
    VITE_S3_BUCKET=tensor-labz-store,\
    VITE_S3_REGION=eu-north-1
```

After changing env vars, trigger a new Amplify build to pick them up.

---

## `amplify.yml` (build spec)

Create at repo root if Amplify cannot auto-detect the build:

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

---

## SPA routing fix

Add a rewrite rule in **Amplify Console → Rewrites and redirects**:

| Source | Target        | Type            |
| ------ | ------------- | --------------- |
| `/<*>` | `/index.html` | `200 (Rewrite)` |

Without this, refreshing any `/admin/*` URL returns 404.

---

## Custom domain (`tensor-labs.tech`)

```mermaid
sequenceDiagram
    participant You
    participant Amplify
    participant DNS as DNS Provider
    participant ACM as AWS ACM

    You->>Amplify: Add domain tensor-labs.tech
    Amplify-->>You: CNAME records to add
    You->>DNS: Add CNAME records
    Amplify->>ACM: Request SSL certificate
    ACM-->>Amplify: Certificate issued
    Note over DNS,Amplify: DNS propagation (up to 24h)
    Amplify-->>You: Domain active ✓
```
