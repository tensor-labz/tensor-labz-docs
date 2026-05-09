# Lambda API Reference

Base URL: `https://ewf03ybvmc.execute-api.eu-north-1.amazonaws.com`

All endpoints require:

```
Authorization: Bearer <supabase-jwt>
Content-Type: application/json
```

---

## POST `/image/upload-url`

Generate a presigned S3 PUT URL for a **new** image upload.

### Request body

```json
{
  "filename": "cover.jpg",
  "contentType": "image/jpeg",
  "folder": "projects/my-project-slug"
}
```

| Field         | Type   | Required | Description                                 |
| ------------- | ------ | -------- | ------------------------------------------- |
| `filename`    | string | ✅       | Original filename (used for extension only) |
| `contentType` | string | ✅       | MIME type — must be an allowed image type   |
| `folder`      | string | ✅       | S3 folder path (no leading/trailing slash)  |

**Allowed `contentType` values:** `image/jpeg`, `image/png`, `image/webp`, `image/gif`, `image/svg+xml`

### Response `200`

```json
{
  "uploadUrl": "https://tensor-labz-store.s3.eu-north-1.amazonaws.com/projects/my-slug/550e8400-e29b-41d4-a716-446655440000.jpg?X-Amz-...",
  "publicUrl": "https://tensor-labz-store.s3.eu-north-1.amazonaws.com/projects/my-slug/550e8400-e29b-41d4-a716-446655440000.jpg",
  "key": "projects/my-slug/550e8400-e29b-41d4-a716-446655440000.jpg"
}
```

| Field       | Description                                                              |
| ----------- | ------------------------------------------------------------------------ |
| `uploadUrl` | Pre-signed PUT URL — valid for 15 minutes. Use this as the `PUT` target. |
| `publicUrl` | Final public URL to store in Supabase (never expires).                   |
| `key`       | S3 object key — save this if you need to delete the image later.         |

### Frontend usage

```typescript
const res = await fetch(`${LAMBDA_URL}/image/upload-url`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${token}`,
  },
  body: JSON.stringify({ filename: file.name, contentType: file.type, folder }),
});
const { uploadUrl, publicUrl } = await res.json();

// Upload directly to S3 (Lambda not involved)
await fetch(uploadUrl, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': file.type },
});

// Store publicUrl in Supabase
await supabase.from('projects').upsert({ imageurl: publicUrl });
```

---

## POST `/image/replace`

Delete an existing S3 object and generate a new presigned URL in one call.  
Use this when the user replaces an existing image in a form field.

### Request body

```json
{
  "oldKey": "projects/my-slug/550e8400.jpg",
  "filename": "new-cover.png",
  "contentType": "image/png",
  "folder": "projects/my-project-slug"
}
```

| Field         | Type   | Required | Description                   |
| ------------- | ------ | -------- | ----------------------------- |
| `oldKey`      | string | ✅       | S3 key of the image to delete |
| `filename`    | string | ✅       | New file name                 |
| `contentType` | string | ✅       | New file MIME type            |
| `folder`      | string | ✅       | S3 folder for the new image   |

### Response `200`

Same shape as `/image/upload-url`:

```json
{
  "uploadUrl": "https://...",
  "publicUrl": "https://...",
  "key": "projects/my-slug/<new-uuid>.png"
}
```

!!! note "Delete failure is non-blocking"
If deleting the old object fails (e.g., key doesn't exist), the Lambda logs a warning but still returns the new presigned URL. The upload is not blocked.

---

## DELETE `/image`

Delete one or more S3 objects. Used when a record is deleted or an image is removed from a gallery.

### Request body

```json
{
  "keys": [
    "projects/my-slug/cover.jpg",
    "projects/my-slug/gallery-1.jpg",
    "https://tensor-labz-store.s3.eu-north-1.amazonaws.com/projects/my-slug/gallery-2.jpg"
  ]
}
```

| Field  | Type     | Required | Description                                                  |
| ------ | -------- | -------- | ------------------------------------------------------------ |
| `keys` | string[] | ✅       | Array of S3 keys **or** full CDN/S3 URLs. Must be non-empty. |

Both raw S3 keys (`projects/slug/uuid.jpg`) and full URLs are accepted. Full URLs are converted to keys automatically.

### Response `200`

```json
{ "deleted": 3 }
```

### Frontend usage

```typescript
import { deleteImages } from '@/lib/imageUpload';

// On record delete — remove cover + all extra images
await deleteImages([record.imageurl, ...record.extraImages]);
```

---

## Error responses

All errors return JSON:

```json
{ "error": "Missing or malformed Authorization header" }
```

| Status | Cause                                                 |
| ------ | ----------------------------------------------------- |
| `400`  | Missing required field in request body                |
| `401`  | Missing, expired, or invalid JWT                      |
| `404`  | Unknown route (`METHOD PATH` not matched)             |
| `500`  | Unhandled server error (check Lambda CloudWatch logs) |

---

## CORS

Preflight `OPTIONS` requests return `204` with:

```
Access-Control-Allow-Origin:  <ALLOWED_ORIGIN env var>
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Methods: POST, DELETE, OPTIONS
```
