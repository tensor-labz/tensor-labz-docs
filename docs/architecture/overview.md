# System Architecture Overview

---

## Full system diagram

```mermaid
flowchart TB
    subgraph Browser["Browser"]
        direction TB
        SPA["React 18 SPA\nVite · TypeScript · Tailwind\nRedux Toolkit"]
        subgraph Pages["Public Pages"]
            P1["/ Hero"]
            P2["/services · /services/:slug"]
            P3["/project/:slug"]
            P5["/posts · /posts/:slug"]
            P4["/about · /contact"]
        end
        subgraph AdminPanel["Admin Panel  /admin/*"]
            Table["CrudTable\nreact-data-table-component\nsearch · sort · pagination"]
            Form["AdminCrudForm\ndynamic field renderer"]
            Settings["AdminSettings\nTables · Forms config"]
            AdminStore["Redux adminSlice\ncreateEntityAdapter\nfetchRecords · createRecord\nupdateRecord · deleteRecord"]
        end
    end

    subgraph SupabaseCloud["Supabase (Cloud)"]
        PG["PostgreSQL\nhero · services · projects\nabout · contact · social"]
        Config["Config Tables\ntable_config · form_config"]
        SupaAuth["Supabase Auth\nJWT HS256\nemail + password"]
    end

    subgraph AWSCloud["AWS eu-north-1"]
        APIGW["API Gateway\nHTTP API\newf03ybvmc"]
        Fn["Lambda\ntensor-labz-image-handler\nNode.js 20"]
        Bucket["S3 Bucket\ntensor-labz-store\nHome/Hero\nInsights\nprojects/{slug}"]
    end

    Pages      -- "supabase.from().select()" --> PG
    AdminPanel -- "adminSlice thunks\nfetch · create · update · delete" --> PG
    AdminPanel -- "read/write config" --> Config
    AdminPanel -- "signInWithPassword()" --> SupaAuth
    SupaAuth   -- "JWT session" --> AdminPanel
    AdminPanel -- "POST + Bearer JWT" --> APIGW
    APIGW      --> Fn
    Fn         -- "verify JWT" --> SupaAuth
    Fn         -- "presigned PUT URL" --> APIGW
    APIGW      -- "{ uploadUrl, publicUrl, key }" --> AdminPanel
    AdminPanel -- "PUT binary (direct)" --> Bucket
```

---

## Public page data flow

```mermaid
sequenceDiagram
    participant User
    participant Redux
    participant Supabase

    User->>Redux: navigate to /services
    Redux->>Redux: dispatch fetchServices() thunk
    Note over Redux: status === 'idle'
    Redux->>Supabase: from('services').select('*').order('id')
    Supabase-->>Redux: rows[]
    Redux->>Redux: store in servicesSlice
    Redux-->>User: component re-renders with data
```

---

## Image upload flow

```mermaid
sequenceDiagram
    participant Admin as Admin (Browser)
    participant Supabase
    participant Lambda
    participant S3

    Admin->>Supabase: auth.getSession()
    Supabase-->>Admin: access_token (JWT)

    Admin->>Lambda: POST /image/upload-url<br/>{ filename, contentType, folder }<br/>Authorization: Bearer JWT
    Lambda->>Lambda: verifyToken(jwt)<br/>using SUPABASE_JWT_SECRET
    Lambda->>S3: generate presigned PUT URL (15 min TTL)
    S3-->>Lambda: signed URL
    Lambda-->>Admin: { uploadUrl, publicUrl, key }

    Admin->>S3: PUT <uploadUrl>  (binary file, no Lambda involved)
    S3-->>Admin: 200 OK

    Admin->>Supabase: from('projects').upsert({ imageurl: publicUrl })
    Supabase-->>Admin: saved ✓
```

---

## Image replace flow (edit existing record)

```mermaid
sequenceDiagram
    participant Admin as Admin (Browser)
    participant Lambda
    participant S3

    Admin->>Lambda: POST /image/replace<br/>{ oldKey, filename, contentType, folder }
    Lambda->>S3: DeleteObjects([oldKey])
    Note over Lambda,S3: Delete failure is non-blocking
    Lambda->>S3: generate new presigned PUT URL
    S3-->>Lambda: signed URL
    Lambda-->>Admin: { uploadUrl, publicUrl, key }

    Admin->>S3: PUT <uploadUrl>  (new binary file)
```

---

## Frontend page composition

The list pages (posts, services) are built from one reusable, data-driven set in `src/shared/components/` — there are no per-page card/list/pagination/hero components.

```mermaid
flowchart TB
    Page["page (Posts / Services)\nmaps data → props"] --> CP["ContentPage"]
    CP --> CH["ContentHeader\neyebrow · title · search · glow\n(sticky)"]
    CP --> SB["sidebar (optional)\ne.g. ServiceFilterDrawer"]
    CP --> CL["ContentList\nloading / empty slots"]
    CL --> CC["ContentCard ×N\nid · title · imgUrl · tags\ndescription · link · created_at"]
    CP --> PG["ContentPagination\npage · totalPages · onPageChange"]
```

| Component          | Role                                                                  |
| ------------------ | -------------------------------------------------------------------- |
| `ContentPage`      | Composes header + optional sidebar + list + pagination from props    |
| `ContentHeader`    | Sticky page header (search, glow); publishes height as `--page-header-h` |
| `ContentCard`      | Normalized content card; uniform height with image fill              |
| `ContentList`      | Vertical card list with loading/empty states                        |
| `ContentPagination`| Controlled prev/next + numbered paging                              |

- **Services page** is URL-driven: the active service comes from the route `:slug` (loads the correct list immediately on entry/refresh/back-forward), search uses `?q`, paging uses `?page`, and the filter is a left slide-in drawer in `ContentPage`'s `sidebar` slot. `/services` shows all; `/services/:slug` filters — no `/services/all` redirect.
- `ServiceCard` (home page service-category card) is separate from these content components.

### Detail pages

The single-item pages (post detail, project detail) share one composition the same way the list pages share `ContentPage`:

```mermaid
flowchart TB
    DP["page (PostDetail / Project)\nmaps data → props"] --> DPC["DetailsPage"]
    DPC --> Hero["hero: cover display frame\n+ metadata column\n(kicker · title · tags · meta)"]
    DPC --> Badge["titleBadge (optional)\ne.g. project crown"]
    DPC --> Act["actions (optional)\ne.g. Get-Support / contact"]
    DPC --> Body["rich-text body (DOMPurify)"]
    DPC --> MG["MediaGallery (children)\nvideo players + image viewer\n+ thumbnail strip"]
    DPC --> Share["social share row"]
```

| Component     | Role                                                                       |
| ------------- | -------------------------------------------------------------------------- |
| `DetailsPage` | Detail composition: cover frame + metadata hero, body, share; optional `titleBadge` / `actions` / media `children` slots |
| `MediaGallery`| Generic media set (`MediaItem[]`) — videos + image viewer with thumbnails  |

- **`PostDetail`** feeds post data + `additional_media` into `MediaGallery`; cover supports image / YouTube / video (player embedded).
- **`Project`** reuses `DetailsPage` with the crown via `titleBadge`, Get-Support/contact via `actions`, and `extraImages` via `MediaGallery`.
- `coverThumbnail(url, type)` (`postService`) turns a YouTube cover link into a still thumbnail for list cards.

---

## Key design decisions

| Decision                              | Reason                                                           |
| ------------------------------------- | ---------------------------------------------------------------- |
| Supabase over Firestore               | Relational PostgreSQL — better for structured CMS with joins     |
| Lambda for S3 (not direct SDK)        | AWS credentials never in browser bundle                          |
| Supabase JWT for Lambda auth          | Single auth system — no second token service                     |
| Presigned PUT (browser → S3 direct)   | Lambda never handles binary — no 6 MB API Gateway limit          |
| Redux Toolkit                         | Consistent async state, memoized selectors, avoids prop drilling |
| Dynamic table/form config in Supabase | Schema changes without code deployments                          |
