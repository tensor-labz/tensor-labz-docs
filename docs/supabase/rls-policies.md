# Row Level Security (RLS) Policies

Supabase RLS controls who can read and write each table. Run all policies after enabling RLS.

---

## Policy pattern

| Table          | Public read | Authenticated write |
| -------------- | ----------- | ------------------- |
| `hero`         | ✅          | ✅                  |
| `services`     | ✅          | ✅                  |
| `projects`     | ✅          | ✅                  |
| `about`        | ✅          | ✅                  |
| `contact`      | ✅          | ✅                  |
| `social`       | ✅          | ✅                  |
| `table_config` | ✅          | ✅                  |
| `form_config`  | ✅          | ✅                  |

- **Public read** — the React frontend reads these tables without auth (anonymous browsing).
- **Authenticated write** — only logged-in admin users can insert, update, delete.

---

## Full SQL

```sql
-- ── hero ──────────────────────────────────────
create policy "public read hero"
  on hero for select using (true);

create policy "admin write hero"
  on hero for all using (auth.role() = 'authenticated');

-- ── services ──────────────────────────────────
create policy "public read services"
  on services for select using (true);

create policy "admin write services"
  on services for all using (auth.role() = 'authenticated');

-- ── projects ──────────────────────────────────
create policy "public read projects"
  on projects for select using (true);

create policy "admin write projects"
  on projects for all using (auth.role() = 'authenticated');

-- ── about ─────────────────────────────────────
create policy "public read about"
  on about for select using (true);

create policy "admin write about"
  on about for all using (auth.role() = 'authenticated');

-- ── contact ───────────────────────────────────
create policy "public read contact"
  on contact for select using (true);

create policy "admin write contact"
  on contact for all using (auth.role() = 'authenticated');

-- ── social ────────────────────────────────────
create policy "public read social"
  on social for select using (true);

create policy "admin write social"
  on social for all using (auth.role() = 'authenticated');

-- ── table_config ──────────────────────────────
create policy "public read table_config"
  on table_config for select using (true);

create policy "admin write table_config"
  on table_config for all using (auth.role() = 'authenticated');

-- ── form_config ───────────────────────────────
create policy "public read form_config"
  on form_config for select using (true);

create policy "admin write form_config"
  on form_config for all using (auth.role() = 'authenticated');
```

---

## Applying all at once (copy-paste)

```sql
do $$
declare
  t text;
begin
  foreach t in array array[
    'hero','services','projects','about','contact','social','table_config','form_config'
  ] loop
    execute format(
      'create policy "public read %1$s" on %1$s for select using (true);
       create policy "admin write %1$s" on %1$s for all using (auth.role() = ''authenticated'');',
      t
    );
  end loop;
end
$$;
```

---

## Anon key vs Service role key

| Key                     | Used by                  | Can bypass RLS?     |
| ----------------------- | ------------------------ | ------------------- |
| `anon` / `publishable`  | Frontend browser         | No — subject to RLS |
| `service_role` (secret) | Server-side scripts only | Yes — skips RLS     |

Never expose the `service_role` key in the frontend bundle. The `anon` key is safe to commit in `.env.example`.
