# Site Control

Site Control is the admin panel page for managing all public-facing company information from one place. Access it at `/admin/site-control`.

---

## Company Info tab

All fields are saved to the `company_info` table in Supabase (row `id = 1`). Click **Save** in the top-right to persist changes.

| Field | Column | Description |
| ----- | ------ | ----------- |
| Company Name | `name` | Shown in footer copyright, page title, About Us heading |
| Tagline | `tagline` | Short headline — shown in mobile nav drawer footer |
| Description | `description` | Public company description in footer brand column and Contact Us quote |
| Available Hours | `available_hours` | Business hours shown in footer (e.g. `Mon – Fri: 8:00 AM – 6:00 PM`) |
| Logo — Light mode | `logo_url` | Used on light theme — upload via drag & drop or paste S3 URL |
| Logo — Dark mode | `logo_url_dark` | Used on dark theme — upload via drag & drop or paste S3 URL |
| Who We Are | `who_we_are` | Paragraph shown on the About Us page |
| Vision | `vision` | Vision statement shown on About Us page |
| Mission | `mission` | Mission statement shown on About Us page |

### Logo theme logic

The header and mobile nav automatically pick the right logo:

- Both logos set → light theme uses `logo_url`, dark theme uses `logo_url_dark`
- Only one logo set → used for both themes
- Neither set → falls back to the bundled local asset (`src/assets/images/logo.png`)

### Uploading a logo directly to S3

If you prefer not to use the drag & drop uploader, upload the file manually and paste the URL:

```bash
aws --profile tensor s3 cp logo.png \
  s3://tensor-labz-store/assets/upload/logo.png \
  --acl public-read --content-type image/png
```

Public URL:
```
https://tensor-labz-store.s3.eu-north-1.amazonaws.com/assets/upload/logo.png
```

Paste this URL into the **Logo — Light mode** or **Logo — Dark mode** field using the **S3 / URL** tab.

---

## Contact Details section

Manages rows in the `contact` table. Each row has:

| Column | Description |
| ------ | ----------- |
| `contact` | Type key — `email`, `phone`, `whatsapp`, `address`, `map`, etc. |
| `title` | Display label — e.g. `Corporate Email`, `Mobile` |
| `value` | The actual contact value — e.g. `tensoragri@gmail.com` |
| `link` | Optional explicit URL — overrides auto-derived href |

### Auto-derived links

When you save a row and leave the **Link** field empty, it is auto-filled based on the `contact` type:

| Type | Auto-derived link |
| ---- | ----------------- |
| `email` / `mail` | `mailto:<value>` |
| `whatsapp` | `https://wa.me/<digits only>` |
| `phone` / `tel` / `mobile` | `tel:<value>` |
| `address` / `location` / `map` | `https://maps.google.com/search?q=<value>` |
| anything else | none |

You can override this by typing a custom URL in the **Link** field.

### Google Maps embed URL

To display an interactive embedded map on the Contact Us page, store the Google Maps embed URL in the address row's `link` field.

**Step-by-step:**

1. Open [Google Maps](https://maps.google.com) and search for your location
2. Click **Share** (the share icon)
3. Select the **Embed a map** tab
4. Click **Copy HTML** — you will see something like:
   ```html
   <iframe src="https://www.google.com/maps/embed?pb=!1m17..." allowfullscreen></iframe>
   ```
5. Copy **only the URL** inside `src="..."`:
   ```
   https://www.google.com/maps/embed?pb=!1m17!1m12...
   ```
6. In Site Control, edit the **Address** contact row and paste the URL into the **Link** field
7. Click **Save**

Or update directly via SQL:

```sql
UPDATE contact
SET link = 'https://www.google.com/maps/embed?pb=!1m17...'
WHERE contact = 'address';
```

!!! tip
    The embed URL always starts with `https://www.google.com/maps/embed?pb=`.
    Do not use the regular maps.google.com share link — it will not embed correctly.

---

## Social Media section

Manages rows in the `social` table. Each row has:

| Column | Description |
| ------ | ----------- |
| `social_media` | Platform name — e.g. `Facebook`, `LinkedIn`, `WhatsApp` |
| `value` | Full URL to the profile |

The footer automatically maps platform names to icons. Supported platforms:

| Platform name (case-insensitive) | Icon |
| -------------------------------- | ---- |
| `whatsapp` | WhatsApp |
| `facebook` | Facebook |
| `linkedin` | LinkedIn |
| `instagram` | Instagram |
| `tiktok` | TikTok |
| `youtube` | YouTube |
| `twitter` / `x` | Twitter/X |
| anything else | Globe |

---

## Database schema

```sql
-- company_info (one row, id = 1)
CREATE TABLE company_info (
  id               serial PRIMARY KEY,
  name             text,
  tagline          text,
  description      text,
  logo_url         text,
  logo_url_dark    text,
  available_hours  text,
  who_we_are       text,
  vision           text,
  mission          text,
  map_url          text   -- reserved, use contact.link for embed URL
);

-- contact
CREATE TABLE contact (
  id       serial PRIMARY KEY,
  contact  text,   -- type key
  title    text,
  value    text,
  link     text    -- optional explicit href / embed URL
);

-- social
CREATE TABLE social (
  id           serial PRIMARY KEY,
  social_media text,
  value        text
);
```

### RLS policies required

```sql
-- company_info: public read, authenticated write
CREATE POLICY "public read company_info" ON company_info
  FOR SELECT USING (true);

CREATE POLICY "auth write company_info" ON company_info
  FOR ALL TO authenticated USING (true) WITH CHECK (true);

-- contact: public read, authenticated write
CREATE POLICY "public read contact" ON contact
  FOR SELECT USING (true);

CREATE POLICY "auth write contact" ON contact
  FOR ALL TO authenticated USING (true) WITH CHECK (true);

-- social: public read, authenticated write
CREATE POLICY "public read social" ON social
  FOR SELECT USING (true);

CREATE POLICY "auth write social" ON social
  FOR ALL TO authenticated USING (true) WITH CHECK (true);
```

---

## Seed data

Run this once to populate the initial company info and contact rows:

```sql
-- Add columns if missing (safe to re-run)
ALTER TABLE company_info ADD COLUMN IF NOT EXISTS who_we_are     text;
ALTER TABLE company_info ADD COLUMN IF NOT EXISTS logo_url_dark  text;
ALTER TABLE company_info ADD COLUMN IF NOT EXISTS available_hours text;
ALTER TABLE contact      ADD COLUMN IF NOT EXISTS link           text;

-- Seed company_info
INSERT INTO company_info (id, name, tagline, description, logo_url, available_hours)
VALUES (
  1,
  'Tensor Labs',
  'Creating Sustainable Impact Through Technology',
  'Empowering creators and problem-solvers through research, innovation, and practical application.',
  'https://tensor-labz-store.s3.eu-north-1.amazonaws.com/assets/upload/logo.png',
  'Mon – Fri: 8:00 AM – 6:00 PM'
)
ON CONFLICT (id) DO UPDATE SET
  name            = EXCLUDED.name,
  tagline         = EXCLUDED.tagline,
  description     = EXCLUDED.description,
  logo_url        = EXCLUDED.logo_url,
  available_hours = EXCLUDED.available_hours;

-- Seed contact rows
INSERT INTO contact (contact, title, value, link) VALUES
  ('email',    'Corporate Email', 'tensoragri@gmail.com', 'mailto:tensoragri@gmail.com'),
  ('whatsapp', 'WhatsApp',        '94705359369',          'https://wa.me/94705359369'),
  ('address',  'Location',        'Jaffna, Sri Lanka',    'https://www.google.com/maps/embed?pb=YOUR_EMBED_CODE'),
  ('phoneNo',  'Mobile',          '+94770484739',         'tel:+94770484739')
ON CONFLICT DO NOTHING;
```
