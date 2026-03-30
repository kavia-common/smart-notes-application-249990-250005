# Notes DB schema + seed (PostgreSQL)

This container uses PostgreSQL on port `5000` with database `myapp`.

Connection string (source of truth): `db_connection.txt`

## Tables

- `notes`
  - `id` (bigserial PK)
  - `title` (text, required)
  - `content` (text, default empty string)
  - `is_archived` (boolean, default false)
  - `created_at` / `updated_at` (timestamptz)
  - Trigger updates `updated_at` automatically on UPDATE

- `tags`
  - `id` (bigserial PK)
  - `name` (text, unique)
  - `created_at` (timestamptz)

- `note_tags` (many-to-many)
  - composite PK (`note_id`, `tag_id`)
  - `note_id` -> `notes(id)` ON DELETE CASCADE
  - `tag_id` -> `tags(id)` ON DELETE CASCADE
  - `created_at`

## Indexes

- `idx_notes_created_at` on `notes(created_at DESC)` (fast ordering)
- `idx_tags_name` on `tags(name)` (fast lookup)
- `idx_note_tags_tag_id` on `note_tags(tag_id)` (fast filter by tag)
- Trigram search indexes (requires `pg_trgm` extension):
  - `idx_notes_title_trgm` on `notes` using GIN trigram ops
  - `idx_notes_content_trgm` on `notes` using GIN trigram ops

These improve performance for `ILIKE '%query%'` search on `title`/`content`.

## Minimal seed data

Tags:
- `work`
- `personal`
- `ideas`

Notes:
- `Welcome to Smart Notes`
- `Shopping list`
- `Project ideas`

Tag links:
- `Shopping list` -> `personal`
- `Project ideas` -> `ideas`, `work`

## How to apply (container convention)

Use the exact `psql` command from `db_connection.txt` and run SQL statements one at a time, e.g.:

```bash
CONN=$(cat db_connection.txt)

# Extension
$CONN -c "CREATE EXTENSION IF NOT EXISTS pg_trgm;"

# DDL
$CONN -c "CREATE TABLE IF NOT EXISTS notes (...);"
$CONN -c "CREATE TABLE IF NOT EXISTS tags (...);"
$CONN -c "CREATE TABLE IF NOT EXISTS note_tags (...);"

# Seed
$CONN -c "INSERT INTO tags (name) VALUES ('work') ON CONFLICT (name) DO NOTHING;"
...
```

Notes:
- When creating the `set_updated_at()` trigger function, use `\\$\\$` escaping if you run via shell `-c` to avoid `$$` expansion issues.
