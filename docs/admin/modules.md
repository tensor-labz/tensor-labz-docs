# Content Modules

Modules are the content sections of the site. Each module maps to a Supabase table and drives the admin list table and CRUD form.

---

## Module list

| Module ID  | Label        | Table      | Title field    | Description field | Image field |
| ---------- | ------------ | ---------- | -------------- | ----------------- | ----------- |
| `hero`     | Hero Slides  | `hero`     | `title`        | `subtitle`        | `img`       |
| `services` | Services     | `services` | `title`        | `description`     | `imageURL`  |
| `projects` | Projects     | `projects` | `title`        | `description`     | `imageURL`  |
| `about`    | About Us     | `about`    | `components`   | `value`           | —           |
| `contact`  | Contact Info | `contact`  | `title`        | `value`           | —           |
| `social`   | Social Links | `social`   | `social_media` | `value`           | —           |

---

## ModuleConfig structure

Defined in `src/features/admin/config/modules.tsx`:

```typescript
interface FieldConfig {
  key: string;
  label: string;
  type: FieldType; // see field types below
  placeholder?: string;
  required?: boolean;
  span?: 'full' | 'half'; // form grid column span
  options?: string[];      // for static radio/select types
  relation?: RelationConfig; // for dynamic FK select — fetches options from Supabase
}

interface ModuleConfig {
  id: string;              // matches Supabase table name
  label: string;           // display name in sidebar
  icon: IconType;          // react-icons icon
  imageField?: string;     // key of the image column shown as thumbnail in table
  titleField: string;      // primary text column in table
  descriptionField?: string; // secondary text column in table
  tableColumns?: string[]; // extra field keys shown in CrudTable beyond title + description
  fields: FieldConfig[];   // static default field definitions (used by AdminCrudForm)
}
```

---

## Field types

| Type         | Renders               | Notes                               |
| ------------ | --------------------- | ----------------------------------- |
| `text`       | `<input type="text">` |                                     |
| `textarea`   | `<textarea>`          | 3 rows, resizable                   |
| `url`        | `<input type="url">`  | Validates URL format                |
| `image`      | Upload/URL tabs       | S3 upload via Lambda, drag & drop   |
| `images`     | Gallery grid          | Multiple images, S3 upload + URL    |
| `toggle`     | Animated switch       | Stores `true`/`false`               |
| `checkbox`   | HTML checkbox         | Stores `true`/`false`               |
| `tags`       | Text input            | Comma-separated string stored as-is |
| `multiinput` | Array of inputs       | Stored as `text[]`, add/remove rows |
| `richtext`   | React Quill editor    | WYSIWYG HTML editor                 |
| `radio`      | Radio buttons         | Requires `options: string[]`        |
| `select`     | `<select>` dropdown   | Static: requires `options: string[]`. Dynamic FK: set `relation` instead |

---

## Default field definitions

### Hero (`hero`)

| Key        | Label    | Type    | Span | Required |
| ---------- | -------- | ------- | ---- | -------- |
| `img`      | Image    | `image` | full | —        |
| `title`    | Title    | `text`  | full | ✅       |
| `subtitle` | Subtitle | `text`  | full | —        |

### Services (`services`)

| Key            | Label             | Type       | Span | Required |
| -------------- | ----------------- | ---------- | ---- | -------- |
| `imageURL`     | Image             | `image`    | full | —        |
| `title`        | Title             | `text`     | half | ✅       |
| `slug`         | Slug              | `text`     | half | ✅       |
| `description`  | Description       | `textarea` | full | —        |
| `icon`         | Icon Class        | `text`     | half | —        |
| `show_in_home` | Show on Home Page | `toggle`   | half | —        |

### Projects (`projects`)

| Key           | Label             | Type       | Span | Required |
| ------------- | ----------------- | ---------- | ---- | -------- |
| `imageURL`    | Image             | `image`    | full | —        |
| `title`       | Title             | `text`     | half | ✅       |
| `slug`        | Slug              | `text`     | half | ✅       |
| `service_id`  | Service           | `select`   | half | —        |
| `tags`        | Tags              | `tags`     | half | —        |
| `description` | Short Description | `textarea` | full | —        |
| `content`     | Full Content      | `richtext` | full | —        |
| `vedio_demo`  | Video Demo URL    | `url`      | full | —        |
| `extraImages` | Extra Images      | `images`   | full | —        |
| `is_top`      | Featured Project  | `toggle`   | half | —        |

### About (`about`)

| Key          | Label       | Type       | Span | Required |
| ------------ | ----------- | ---------- | ---- | -------- |
| `components` | Title       | `text`     | full | ✅       |
| `value`      | Description | `textarea` | full | ✅       |

### Contact (`contact`)

| Key       | Label        | Type   | Span | Required |
| --------- | ------------ | ------ | ---- | -------- |
| `contact` | Contact Type | `text` | half | ✅       |
| `title`   | Label        | `text` | half | ✅       |
| `value`   | Value        | `text` | full | ✅       |

### Social Links (`social`)

| Key            | Label    | Type   | Span | Required |
| -------------- | -------- | ------ | ---- | -------- |
| `social_media` | Platform | `text` | half | ✅       |
| `value`        | URL      | `url`  | half | ✅       |

---

## tableColumns — extra table columns

`tableColumns` lists field keys that appear in `CrudTable` between the title and description columns when no `table_config` Supabase row exists for the module.

```typescript
// modules.tsx — projects module
{
  id: 'projects',
  titleField: 'title',
  descriptionField: 'description',
  tableColumns: ['service_id'],   // ← adds a "Service" column (resolved to label)
  ...
}
```

The column header is taken from the matching `FieldConfig.label` in `fields[]`. If the key has no matching field definition, the key itself is used as the header.

This is a code-level default. Once you save a `table_config` row for the module via **Admin → Settings → Tables**, the Supabase config takes over and `tableColumns` is ignored.

---

## Relation fields (FK select)

A `select` field can be backed by a live Supabase table instead of a static `options` list. Set the `relation` key instead of `options`:

```typescript
{
  key: 'service_id',
  label: 'Service',
  type: 'select',
  span: 'half',
  relation: {
    table: 'services',     // Supabase table to query
    labelField: 'title',   // field shown in the dropdown
    valueField: 'id',      // field stored as the value (default: 'id')
  },
}
```

### RelationConfig interface

```typescript
interface RelationConfig {
  table: string;       // Supabase table to fetch options from
  labelField: string;  // field to display in the dropdown
  valueField?: string; // field to store as value — defaults to 'id'
}
```

### How it works

**In `AdminCrudForm`** — `RelationSelect` fetches all rows from `relation.table` on mount and renders a native `<select>` dropdown. The stored value is the numeric ID (or `null` for blank).

**In `CrudTable`** — on module load, the table fetches all rows from each relation table and builds an `id → label` lookup map. Stored integer IDs are transparently resolved to their display label in the column cell, for both the Supabase colConfig path and the static `tableColumns` fallback.

### Supabase migration

The `projects` table uses a foreign key instead of a text slug:

```sql
-- add the FK column
alter table projects
  add column if not exists service_id bigint references services(id);

-- backfill from the old text slug (optional, if migrating existing data)
update projects p
set service_id = s.id
from services s
where s.slug = p.service;

-- drop the old column once backfill is verified
alter table projects drop column if exists service;
```

---

## Dynamic override

The static `fields` array is only the default. If a row exists in `form_config` for a module, `AdminCrudForm` uses that instead. See [Dynamic Form Config](form-config.md).
