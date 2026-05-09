# About Us Page

The About Us page (`/about`) is a fully dynamic, data-driven page that pulls content from Supabase and renders a polished multi-section layout with Three.js animations and a media gallery.

---

## Page structure

| Section | Data source | Renders when |
| ------- | ----------- | ------------ |
| **Hero** | `company_info.name`, `company_info.description` | Always |
| **Vision & Mission** | `company_info.vision`, `company_info.mission` | Both fields non-empty |
| **Our Story** | `company_info.who_we_are`, `company_info.tagline` | `who_we_are` non-empty |
| **Media Gallery** | `about_media` table | Table has ≥ 1 row |
| **Key Facts** | `about` table | Table has ≥ 1 row |

---

## Hero section

```
src/pages/AboutUs.tsx
src/features/about/components/HeroParticles.tsx   ← Three.js background
```

The hero occupies the full viewport (`min-h-screen`). It contains:

- **Three.js particle constellation** — lazy-loaded (`React.lazy`) so it doesn't block the initial paint.
- **Radial gradient overlay** — fades the particles toward the edges so they don't compete with the text.
- **Animated title & description** — Framer Motion `animate` (not `whileInView`) so they play immediately.
- **Bouncing scroll indicator** — appears after 1 s, loops with `y: [0, 5, 0]`.

### HeroParticles internals

| Constant | Value | Effect |
| -------- | ----- | ------ |
| `COUNT` | 90 | Number of particles |
| `CONNECT_DIST` | 1.4 | Max distance to draw a connecting line |
| `SPEED` | 0.0015 | Particle drift speed per frame |
| Particle opacity | 0.55 | Subtle enough not to overpower text |
| Line opacity | 0.09 | Very faint connecting web |

Mouse parallax: as the cursor moves, the camera drifts toward `(mx, -my)` with `lerp(0.04)` smoothing.

The renderer is initialised inside `requestAnimationFrame(init)` to guarantee the mount div has non-zero dimensions before `getBoundingClientRect()` is called. `setSize(w, h, false)` is used to prevent Three.js from overriding the canvas CSS size.

---

## Vision & Mission section

The first scroll section. Both cards animate in from opposite sides (`x: ±20 → 0`) using `whileInView` with `viewport: { once: true }`. The section is conditionally rendered — if `company_info.vision` and `company_info.mission` are both empty, the section is hidden entirely.

---

## Our Story section

Two-column layout: left column has the `who_we_are` text; right column has a glassmorphism quote card with `company_info.tagline`.

---

## Media Gallery

```
src/features/about/components/MediaGallery.tsx
src/features/about/hooks/useAboutMedia.ts
```

### Layout

A single **16:9 featured player** (achieved with `paddingBottom: 56.25%` + `position: absolute inset-0`) with:

- **`<iframe>`** for videos (S3 `.mp4` or YouTube embed URL)
- **`<img>`** for images
- **Crossfade transition** — `AnimatePresence mode="wait"` animates between items
- **Prev / Next arrows** — absolutely positioned, accent color on hover
- **Dot indicators** — top-right corner, active dot expands width
- **Thumbnail strip** — scrollable row of 110 × 62 px thumbnails below the frame

### Auto-rotation

The gallery automatically advances to a **randomly selected item** every **5 seconds**:

- Pauses when the user hovers over the featured frame (`onMouseEnter/Leave`)
- Resets the 5 s timer whenever the user manually navigates (prev/next or thumbnail click)
- Never selects the same item twice in a row

```typescript
timerRef.current = setInterval(() => {
  setActive((cur) => {
    let next: number;
    do { next = Math.floor(Math.random() * items.length); }
    while (next === cur && items.length > 1);
    return next;
  });
}, 5000);
```

### YouTube thumbnails

For `type = 'video'` rows with YouTube embed URLs, the thumbnail strip auto-extracts the preview image:

```typescript
function ytThumb(url: string): string | null {
  const m = url.match(/(?:embed\/|v=|youtu\.be\/|shorts\/)([A-Za-z0-9_-]{11})/);
  return m ? `https://img.youtube.com/vi/${m[1]}/mqdefault.jpg` : null;
}
```

S3 video rows show a play icon placeholder instead (no server-side thumbnail generation).

### `useAboutMedia` hook

```typescript
// src/features/about/hooks/useAboutMedia.ts

export interface MediaItem {
  id: number;
  type: 'video' | 'image';
  url: string;
  title: string;
  sort_order: number;
}
```

Uses a **module-level cache** (`let _cache: MediaItem[] | null = null`) so the Supabase fetch runs at most once per page load, regardless of how many times the hook is mounted.

---

## Database setup

### 1. Create the `about_media` table

```sql
create table about_media (
  id         serial primary key,
  type       text    not null default 'image',
  url        text    not null,
  title      text    default '',
  sort_order integer default 0
);

alter table about_media enable row level security;

create policy "public read about_media"
  on about_media for select using (true);

create policy "auth write about_media"
  on about_media for all to authenticated
  using (true) with check (true);
```

### 2. Insert sample rows

```sql
insert into about_media (type, url, title, sort_order) values
  ('video', 'https://tensor-labz-store.s3.eu-north-1.amazonaws.com/about-us/aboutusBg.mp4',       'About Us',  1),
  ('image', 'https://tensor-labz-store.s3.eu-north-1.amazonaws.com/contact-us/ContactusBgLG.png', 'Our Story', 2);
```

### 3. Add a YouTube video

```sql
insert into about_media (type, url, title, sort_order) values
  ('video', 'https://www.youtube.com/embed/<VIDEO_ID>', 'Demo', 3);
```

### 4. Populate `company_info`

The Vision, Mission, Who We Are, and tagline fields are read from a single row in `company_info`:

```sql
insert into company_info (name, tagline, description, who_we_are, vision, mission)
values (
  'Tensor Labz',
  'Creating Sustainable Impact Through Technology',
  'Empowering creators and problem-solvers through research, innovation, and practical application.',
  'We are a technology research lab...',
  'To be the leading AI-driven agritech innovator in South Asia.',
  'Build tools that empower farmers and communities through accessible technology.'
);
```

---

## Key Facts section

Reads from the `about` table (rows with `components` as the title and `value` as the description). These are displayed as a 1–3 column responsive card grid at the bottom of the page.

```sql
insert into about (components, value) values
  ('Projects Delivered', '30+ completed across agritech and sustainability'),
  ('Team Size',          '12 engineers, researchers, and designers'),
  ('Countries Reached',  'Operating across Sri Lanka, India, and beyond');
```

---

## File map

```
src/
├── pages/
│   └── AboutUs.tsx                               ← Page layout and section orchestration
└── features/about/
    ├── components/
    │   ├── HeroParticles.tsx                     ← Three.js particle constellation
    │   └── MediaGallery.tsx                      ← 16:9 player + thumbnails + auto-rotate
    └── hooks/
        └── useAboutMedia.ts                      ← Supabase fetch with module-level cache
```
