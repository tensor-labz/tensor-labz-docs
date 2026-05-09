# Admin CRUD Operations

This guide walks through every content management task in the admin panel — creating, reading, updating, and deleting records for each module.

---

## Accessing the admin panel

1. Navigate to `/admin` (e.g. `https://tensorlabz.com/admin`)
2. You'll be redirected to `/login` if not signed in
3. Enter your Supabase admin email and password
4. On success you land on the **Overview** dashboard

!!! tip "Forgotten password"
    Use the Supabase dashboard → **Authentication → Users** to reset a user's password, or use the "Forgot password" flow on the login page if it is configured.

---

## Dashboard overview

The overview page shows live counts for every module pulled directly from Supabase, plus featured/active counts.

```
/admin                 → Dashboard (charts + counts)
/admin/hero            → Hero slides
/admin/services        → Services
/admin/projects        → Projects
/admin/about           → About Us key facts
/admin/contact         → Contact details
/admin/social          → Social links
/admin/settings        → Settings (general, appearance, tables, forms)
```

---

## Creating a record

1. Click a module name in the sidebar (e.g. **Projects**)
2. Click the **+ New** button in the top-right of the table
3. Fill in the form fields
4. Click **Save** — the record is inserted into Supabase and appears in the table immediately

### Image fields

Image fields have two tabs:

=== "Upload File"
    Drag and drop a file, or click to browse. The file is uploaded to S3 via Lambda and the public URL is saved automatically. Supported formats: JPG, PNG, WebP, GIF.

=== "S3 / URL"
    Paste a URL directly — either an S3 public URL you uploaded manually or any other image URL.

    ```
    https://tensor-labz-store.s3.eu-north-1.amazonaws.com/Home/Hero/slide1.jpg
    ```

    See the [S3 Upload Guide](s3-upload-urls.md) for how to get these URLs.

---

## Editing a record

1. Click any row in the module table
2. The edit form opens at `/admin/:module/:id`
3. Make your changes
4. Click **Save** — the row is updated in Supabase

### Replacing an image

- Switch to the **Upload File** tab and drop a new file — the old S3 object is deleted automatically
- Or switch to the **S3 / URL** tab and paste a new URL

---

## Deleting a record

1. Open the record (click the row)
2. Scroll to the bottom of the form
3. Click the **Delete** button (red)
4. Confirm the prompt

!!! warning "Image cleanup"
    When you delete a record, all S3 images attached to it (cover image + `extraImages` array) are deleted from the bucket automatically. This cannot be undone.

---

## Module-by-module guide

### Hero Slides

**Table:** `hero` | **Path:** `/admin/hero`

Controls the full-screen slideshow on the home page.

| Field | Type | Notes |
| ----- | ---- | ----- |
| Image | `image` | Full-screen background — use landscape images (1920 × 1080 recommended) |
| Title | `text` | Large headline text on the slide |
| Subtitle | `text` | Smaller supporting text below the headline |

**Ordering:** Slides display in database insertion order. To reorder, use the Supabase SQL editor:

```sql
-- swap the order of two slides by updating IDs
update hero set id = 999 where id = 1; -- temp
update hero set id = 1   where id = 2;
update hero set id = 2   where id = 999;
```

---

### Services

**Table:** `services` | **Path:** `/admin/services`

Lists what the company offers. Services with `show_in_home = true` appear on the home page.

| Field | Type | Notes |
| ----- | ---- | ----- |
| Image | `image` | Service thumbnail |
| Title | `text` | Service name |
| Slug | `text` | URL-safe identifier — used in `/services/:slug`. Must be unique, no spaces |
| Description | `textarea` | Short summary |
| Icon Class | `text` | CSS icon class or emoji |
| Show on Home | `toggle` | Enable to feature on home page |

!!! important "Slugs are permanent"
    Changing a slug will break any bookmarked URLs for that service's detail page. Only change slugs before the page is publicly shared.

---

### Projects

**Table:** `projects` | **Path:** `/admin/projects`

Portfolio items. Featured projects (`is_top = true`) appear highlighted on the home page.

| Field | Type | Notes |
| ----- | ---- | ----- |
| Image | `image` | Cover image (landscape recommended) |
| Title | `text` | Project name |
| Slug | `text` | URL-safe identifier — used in `/projects/:slug` |
| Service | `select` | Linked service (FK to `services`) |
| Tags | `tags` | Comma-separated keywords |
| Short Description | `textarea` | Summary shown in cards |
| Full Content | `richtext` | HTML body of the project detail page |
| Video Demo URL | `url` | YouTube embed URL e.g. `https://www.youtube.com/embed/...` |
| Extra Images | `images` | Gallery of additional images |
| Featured Project | `toggle` | Show on home page highlights |

---

### About Us — Key Facts

**Table:** `about` | **Path:** `/admin/about`

The "By the Numbers" card grid on the About Us page. Each row becomes one card.

| Field | Type | Example |
| ----- | ---- | ------- |
| Title (`components`) | `text` | `Projects Delivered` |
| Description (`value`) | `textarea` | `30+ across agritech and sustainability` |

Add as many rows as needed — they render in a responsive 1–3 column grid.

---

### Contact Details

**Table:** `contact` | **Path:** `/admin/contact`

The contact info shown on the Contact Us page and footer.

| Field | Type | Notes |
| ----- | ---- | ----- |
| Contact Type (`contact`) | `text` | `email` / `phone` / `address` / `hours` |
| Label (`title`) | `text` | Display label e.g. `Email Address` |
| Value | `text` | The actual data e.g. `tensoragri@gmail.com` |
| Link | `url` | Optional URL override. For `address` rows, paste the Google Maps embed URL here |

**How to get a Google Maps embed URL:**

1. Go to [maps.google.com](https://maps.google.com) and search your location
2. Click **Share → Embed a map**
3. Copy the URL from inside `src="..."` — starts with `https://www.google.com/maps/embed?pb=...`
4. Paste it into the **Link** field of the address row

---

### Social Links

**Table:** `social` | **Path:** `/admin/social`

Platform links shown in the footer and contact page.

| Field | Type | Example |
| ----- | ---- | ------- |
| Platform (`social_media`) | `text` | `Facebook`, `LinkedIn`, `Instagram` |
| URL (`value`) | `url` | `https://www.facebook.com/tensorlabs.tech` |

Supported platforms (icons auto-matched by platform name):

`WhatsApp` · `Facebook` · `LinkedIn` · `Instagram` · `TikTok` · `YouTube` · `Twitter` · `GitHub`

---

## Company Info (Site Control)

**Path:** `/admin/settings → General`

Controls global site identity — name, tagline, description, logos, vision, mission, who-we-are text.

| Field | Used on |
| ----- | ------- |
| Name | Page titles, About Us hero, footer |
| Tagline | About Us quote card |
| Description | About Us hero subtitle |
| Logo URL (light) | Navbar in light mode |
| Logo URL (dark) | Navbar in dark mode |
| Who We Are | About Us — Our Story section |
| Vision | About Us — Vision card |
| Mission | About Us — Mission card |

To update the logo: [upload the file to S3](s3-upload-urls.md) first, then paste the URL here.

---

## Dynamic table & form config

The admin panel supports overriding the default column and field layouts per module without code changes.

### Table columns (what's visible in the list)

1. Go to **Admin → Settings → Tables**
2. Select a module
3. Toggle columns on/off, reorder them, change labels
4. Click **Save** — stored in `table_config` in Supabase

### Form fields (what appears in the edit form)

1. Go to **Admin → Settings → Forms**
2. Select a module
3. Add, remove, or reorder fields using the field builder
4. Click **Save** — stored in `form_config` in Supabase

!!! note "Code fallback"
    If no `table_config` / `form_config` row exists for a module, the admin panel falls back to the static definitions in `src/features/admin/config/modules.tsx`.

---

## Keyboard shortcuts

| Action | Shortcut |
| ------ | -------- |
| Save form | `Ctrl/Cmd + S` |
| Search in table | Click search box or `Ctrl/Cmd + F` |

---

## Troubleshooting

### Record won't save

- Check for required fields (marked with `*`) — leave none blank
- Check your Supabase session hasn't expired — refresh the page and log in again

### Image upload fails

- Confirm the Lambda endpoint is reachable: `VITE_IMAGE_LAMBDA_URL` in `.env`
- Check the Lambda function is deployed and running in AWS Console
- As a fallback, [upload the file manually to S3](s3-upload-urls.md) and paste the URL

### Table is empty but records exist in Supabase

- Check browser network tab — if the Supabase fetch returns a 401, your RLS policies may be misconfigured
- Confirm `public read` policies exist for the table — see [RLS Policies](../supabase/rls-policies.md)

### Deleted record still shows on the live site

The frontend uses a module-level cache that persists for the page session. A hard refresh (`Ctrl/Cmd + Shift + R`) clears the cache and fetches fresh data.
