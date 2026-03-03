# 04 — Backend: Node.js Implementation

## API Contract

```
GET  /api/items?cursor=<base64>&limit=40    → { items, cursor, hasMore }
PUT  /api/items/:id                         → updatedItem (FULL object)  ← CRITICAL
GET  /api/items/:id                         → item
GET  /api/headers/:id                       → header summary
```

> **The contract that enables cache patching:** `PUT` must always return the **complete updated item** — never just `{ success: true }`.

---

## Cursor Encoder

```typescript
// utils/cursor.ts
interface CursorPayload {
  updated_at: string;
  id: number;
}

export function encodeCursor(updated_at: string, id: number): string {
  const payload: CursorPayload = { updated_at, id };
  return Buffer.from(JSON.stringify(payload)).toString('base64url');
}

export function decodeCursor(cursor: string): CursorPayload {
  try {
    const decoded = Buffer.from(cursor, 'base64url').toString('utf-8');
    return JSON.parse(decoded) as CursorPayload;
  } catch {
    throw new Error('Invalid pagination cursor');
  }
}
}
```

---

## Items Controller

```typescript
// controllers/items.controller.ts
import { Request, Response } from 'express';
import { ItemsService } from '../services/items.service';

export class ItemsController {
  constructor(private itemsService: ItemsService) {}

  // GET /api/items
  async list(req: Request, res: Response): Promise<void> {
    const cursor = req.query.cursor as string | undefined;
    const limit = Math.min(parseInt(req.query.limit as string) || 40, 100);

    const result = await this.itemsService.listItems({ cursor, limit });
    res.json(result);
  }

  // PUT /api/items/:id
  async update(req: Request, res: Response): Promise<void> {
    const id = parseInt(req.params.id);
    const updates = req.body;

    const updatedItem = await this.itemsService.updateItem(id, updates);

    if (!updatedItem) {
      res.status(404).json({ error: 'Item not found' });
      return;
    }

    // ⭐ Always return full updated item for client-side cache patching
    res.json(updatedItem);
  }
}
```

---

## Items Service

```typescript
// services/items.service.ts
import { ItemsRepository } from '../repositories/items.repository';
import { decodeCursor, encodeCursor } from '../utils/cursor';

export class ItemsService {
  constructor(private repo: ItemsRepository) {}

  async listItems(params: { cursor?: string; limit: number }) {
    const { cursor, limit } = params;

    let cursorData: { updated_at: string; id: number } | undefined;

    if (cursor) {
      cursorData = decodeCursor(cursor);
    }

    const items = await this.repo.findWithCursor({ cursorData, limit: limit + 1 });

    const hasMore = items.length > limit;
    const pageItems = hasMore ? items.slice(0, limit) : items;

    const lastItem = pageItems[pageItems.length - 1];
    const nextCursor = lastItem
      ? encodeCursor(lastItem.updated_at, lastItem.id)
      : null;

    return {
      items: pageItems,
      cursor: nextCursor,
      hasMore,
    };
  }

  async updateItem(id: number, updates: Partial<Item>): Promise<Item | null> {
    return this.repo.updateAndReturn(id, updates);
  }
}
```

---

## Items Repository

```typescript
// repositories/items.repository.ts
import { Pool } from 'pg';

export class ItemsRepository {
  constructor(private pool: Pool) {}

  // Cursor-based keyset pagination
  async findWithCursor(params: {
    cursorData?: { updated_at: string; id: number };
    limit: number;
  }): Promise<Item[]> {
    const { cursorData, limit } = params;

    if (cursorData) {
      const { rows } = await this.pool.query<Item>(
        `SELECT id, name, status, quantity, updated_at
         FROM items
         WHERE (updated_at, id) < ($1, $2)
         ORDER BY updated_at DESC, id DESC
         LIMIT $3`,
        [cursorData.updated_at, cursorData.id, limit]
      );
      return rows;
    }

    // First page — no cursor
    const { rows } = await this.pool.query<Item>(
      `SELECT id, name, status, quantity, updated_at, header_id
       FROM items
       ORDER BY updated_at DESC, id DESC
       LIMIT $1`,
      [limit]
    );
    return rows;
  }

  // ⭐ UPDATE with RETURNING * — sends full item back to frontend
  async updateAndReturn(id: number, updates: Partial<Item>): Promise<Item | null> {
    const setClauses = Object.keys(updates)
      .map((key, i) => `${key} = $${i + 2}`)
      .join(', ');

    const values = [id, ...Object.values(updates)];

    const { rows } = await this.pool.query<Item>(
      `UPDATE items
       SET ${setClauses}, updated_at = NOW()
       WHERE id = $1
       RETURNING *`,
      values
    );

    return rows[0] ?? null;
  }
}
```

---

## Error Handling Middleware

```typescript
// middleware/error.middleware.ts
export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err.message === 'Invalid pagination cursor') {
    return res.status(400).json({ error: 'Invalid cursor. Please refresh and try again.' });
  }

  console.error(err);
  res.status(500).json({ error: 'Internal server error' });
}
```

---

## Express App Setup

```typescript
// app.ts
import express from 'express';
import { Pool } from 'pg';
import { ItemsRepository } from './repositories/items.repository';
import { ItemsService } from './services/items.service';
import { ItemsController } from './controllers/items.controller';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const repo = new ItemsRepository(pool);
const service = new ItemsService(repo);
const controller = new ItemsController(service);

const app = express();
app.use(express.json());

app.get('/api/items', (req, res) => controller.list(req, res));
app.put('/api/items/:id', (req, res) => controller.update(req, res));

export default app;
```
