# 05 — Database: PostgreSQL Implementation

## Schema Design

```sql
-- Header/summary table (aggregate view)
CREATE TABLE headers (
  id              SERIAL PRIMARY KEY,
  title           VARCHAR(255) NOT NULL,
  total_quantity  INTEGER DEFAULT 0,
  item_count      INTEGER DEFAULT 0,
  last_updated_at TIMESTAMPTZ DEFAULT NOW(),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Items table (the list data)
CREATE TABLE items (
  id          SERIAL PRIMARY KEY,
  header_id   INTEGER REFERENCES headers(id) ON DELETE CASCADE,
  name        VARCHAR(255) NOT NULL,
  status      VARCHAR(50) NOT NULL DEFAULT 'active',
  quantity    INTEGER NOT NULL DEFAULT 0,
  notes       TEXT,
  updated_at  TIMESTAMPTZ DEFAULT NOW(),  -- ← Used for cursor pagination
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- ⭐ Composite index for cursor/keyset pagination
CREATE INDEX idx_items_cursor ON items (updated_at DESC, id DESC);

-- Index for header_id lookups
CREATE INDEX idx_items_header_id ON items (header_id);
```

---

## Cursor Keyset Query

```sql
-- First page (no cursor)
SELECT id, name, status, quantity, updated_at, header_id
FROM items
ORDER BY updated_at DESC, id DESC
LIMIT 40;

-- Subsequent pages (with cursor)
-- cursor encodes: { updated_at: '2025-01-15T10:30:00Z', id: 94 }
SELECT id, name, status, quantity, updated_at, header_id
FROM items
WHERE (updated_at, id) < ($1::timestamptz, $2::integer)
ORDER BY updated_at DESC, id DESC
LIMIT 40;

-- ✅ This is STABLE: edits to existing rows don't change their position
--    (unless updated_at changes — see "Pin Updated Items" strategy)
```

---

## UPDATE with RETURNING *

```sql
-- ⭐ Always RETURNING * so Node.js can send full item to Angular
UPDATE items
SET
  name       = $2,
  status     = $3,
  quantity   = $4,
  notes      = $5,
  updated_at = NOW()
WHERE id = $1
RETURNING *;
```

---

## Header Table Auto-Recalculation Trigger

```sql
-- Function to recalculate header summary
CREATE OR REPLACE FUNCTION recalculate_header()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE headers
  SET
    total_quantity  = (
      SELECT COALESCE(SUM(quantity), 0)
      FROM items
      WHERE header_id = NEW.header_id
    ),
    item_count      = (
      SELECT COUNT(*)
      FROM items
      WHERE header_id = NEW.header_id
    ),
    last_updated_at = NOW()
  WHERE id = NEW.header_id;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger fires after any INSERT or UPDATE on items
CREATE TRIGGER trg_recalculate_header
AFTER INSERT OR UPDATE ON items
FOR EACH ROW
EXECUTE FUNCTION recalculate_header();
```

---

## Optimistic Locking (Concurrent Edit Protection)

```sql
-- Prevent overwriting a newer update with an older one
UPDATE items
SET
  name       = $2,
  status     = $3,
  quantity   = $4,
  updated_at = NOW()
WHERE
  id         = $1
  AND updated_at = $5  -- ← client sends the updated_at it last saw
RETURNING *;

-- If 0 rows returned → someone else updated this item → return 409 Conflict
```

---

## Migration Script

```sql
-- migrations/001_initial_schema.sql

BEGIN;

CREATE TABLE IF NOT EXISTS headers (
  id              SERIAL PRIMARY KEY,
  title           VARCHAR(255) NOT NULL,
  total_quantity  INTEGER DEFAULT 0,
  item_count      INTEGER DEFAULT 0,
  last_updated_at TIMESTAMPTZ DEFAULT NOW(),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS items (
  id          SERIAL PRIMARY KEY,
  header_id   INTEGER REFERENCES headers(id) ON DELETE CASCADE,
  name        VARCHAR(255) NOT NULL,
  status      VARCHAR(50) NOT NULL DEFAULT 'active',
  quantity    INTEGER NOT NULL DEFAULT 0,
  notes       TEXT,
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_items_cursor    ON items (updated_at DESC, id DESC);
CREATE INDEX IF NOT EXISTS idx_items_header_id ON items (header_id);

COMMIT;
```
