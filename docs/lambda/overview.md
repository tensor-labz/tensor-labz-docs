# Lambda Image Service — Overview

The **tensor-labz-image-handler** Lambda is a Node.js 20 service that acts as a secure proxy between the admin panel and AWS S3.

Repository: [ThanuMahee12/tensor-labz-image-lambda](https://github.com/ThanuMahee12/tensor-labz-image-lambda)

---

## Infrastructure

| Resource      | Value                                                     |
| ------------- | --------------------------------------------------------- |
| Function name | `tensor-labz-image-handler`                               |
| Runtime       | Node.js 20.x                                              |
| Region        | `eu-north-1`                                              |
| Memory        | 256 MB                                                    |
| Timeout       | 15 seconds                                                |
| API Gateway   | HTTP API — `ewf03ybvmc`                                   |
| Base URL      | `https://ewf03ybvmc.execute-api.eu-north-1.amazonaws.com` |
| S3 bucket     | `tensor-labz-store`                                       |
| IAM role      | `tensor-labz-lambda-exec`                                 |

---

## New image upload

```mermaid
sequenceDiagram
    actor Admin as Admin (Browser)
    participant Lambda
    participant S3
    participant Supabase

    Admin->>Supabase: auth.getSession()
    Supabase-->>Admin: access_token (JWT)

    Admin->>Lambda: POST /image/upload-url<br/>{ filename, contentType, folder }<br/>Authorization: Bearer JWT
    Lambda->>Lambda: verifyToken(JWT)<br/>using SUPABASE_JWT_SECRET

    alt JWT invalid
        Lambda-->>Admin: 401 Unauthorized
    end

    Lambda->>S3: generate presigned PUT URL (TTL 15 min)
    S3-->>Lambda: signed URL
    Lambda-->>Admin: { uploadUrl, publicUrl, key }

    Admin->>S3: PUT binary file directly to uploadUrl
    S3-->>Admin: 200 OK (file stored)

    Admin->>Supabase: from('table').upsert({ imageurl: publicUrl })
    Supabase-->>Admin: saved ✓
```

---

## Replace existing image

```mermaid
sequenceDiagram
    actor Admin as Admin (Browser)
    participant Lambda
    participant S3

    Admin->>Lambda: POST /image/replace<br/>{ oldKey, filename, contentType, folder }

    Lambda->>S3: DeleteObjects([oldKey])
    Note over Lambda,S3: failure here is non-blocking

    Lambda->>S3: generate new presigned PUT URL
    S3-->>Lambda: new signed URL
    Lambda-->>Admin: { uploadUrl, publicUrl, key }

    Admin->>S3: PUT new binary file directly
```

---

## Delete image(s)

```mermaid
sequenceDiagram
    actor Admin as Admin (Browser)
    participant Lambda
    participant S3

    Admin->>Lambda: DELETE /image<br/>{ keys: ["key1.jpg", "key2.jpg"] }
    Lambda->>S3: DeleteObjects(keys)
    S3-->>Lambda: deleted count
    Lambda-->>Admin: { deleted: 2 }
```

---

## S3 folder structure

```mermaid
graph TD
    Bucket["tensor-labz-store (S3 bucket)"]
    Bucket --> Hero["Home/Hero/"]
    Bucket --> Insights["Insights/"]
    Bucket --> Projects["projects/"]
    Bucket --> Other["other modules/"]

    Hero --> H1["&lt;uuid&gt;.jpg"]
    Hero --> H2["&lt;uuid&gt;.webp"]

    Insights --> I1["&lt;uuid&gt;.jpg"]
    Insights --> I2["&lt;uuid&gt;.png"]

    Projects --> P1["my-project-slug/"]
    Projects --> P2["another-project/"]
    P1 --> PA["&lt;uuid&gt;.jpg  (cover)"]
    P1 --> PB["&lt;uuid&gt;.jpg  (extra)"]
```

The folder is determined by `moduleFolder()` in `src/lib/imageUpload.ts`:

```typescript
export function moduleFolder(moduleId: string, slug?: string): string {
  if (moduleId === 'hero') return 'Home/Hero';
  if (moduleId === 'services') return 'Insights';
  if (moduleId === 'projects') return `projects/${slug ?? 'draft'}`;
  return moduleId;
}
```

---

## Key source files

```
src/
  index.ts    ← main handler — routes POST/DELETE to auth + s3
  auth.ts     ← verifyToken() — validates Supabase JWT (HS256)
  s3.ts       ← createPresignedUrl(), deleteObjects(), urlToKey()
  types.ts    ← TypeScript interfaces for all request/response bodies
```

---

## Security model

```mermaid
flowchart LR
    Admin["Admin\n(browser)"] -- "Bearer JWT" --> GW["API Gateway"]
    GW --> Fn["Lambda\nhandler"]
    Fn -- "jwt.verify(token, secret)" --> Check{Valid?}
    Check -- "No" --> Err["401 Unauthorized"]
    Check -- "Yes" --> S3["S3 operations\npresign / delete"]
```

- All three endpoints require a valid `Authorization: Bearer <jwt>` header
- JWT is verified using `SUPABASE_JWT_SECRET` (HS256 algorithm)
- Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/svg+xml`
- Presigned URLs expire after **15 minutes**
- CORS origin controlled by `ALLOWED_ORIGIN` env var
