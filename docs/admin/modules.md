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
  options?: string[]; // for radio/select types
}

interface ModuleConfig {
  id: string; // matches Supabase table name
  label: string; // display name in sidebar
  icon: IconType; // react-icons icon
  imageField?: string; // key of the image shown in list table
  titleField: string; // primary text in list table
  descriptionField?: string; // secondary text in list table
  fields: FieldConfig[]; // static default field definitions
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
| `select`     | `<select>` dropdown   | Requires `options: string[]`        |

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
| `service`     | Service Slug      | `text`     | half | —        |
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

## Dynamic override

The static `fields` array is only the default. If a row exists in `form_config` for a module, `AdminCrudForm` uses that instead. See [Dynamic Form Config](form-config.md).
