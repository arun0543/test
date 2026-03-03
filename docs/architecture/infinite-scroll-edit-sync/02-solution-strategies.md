# 02 — Solution Strategies

## Overview of All Strategies

| # | Strategy | Complexity | Best For |
|---|----------|-----------|----------|
| 1 | Client-side cache patch | Low | All apps |
| 2 | Scroll position restore | Low | All apps |
| 3 | Cursor/keyset pagination | Medium | Large datasets |
| 4 | Optimistic UI | Medium | High-frequency edits |
| 5 | Pin edited item to top | Low | Chronological lists |
| 6 | WebSocket/SSE push | High | Real-time collaborative |

---

## Strategy 1: Client-Side Cache Patch ⭐ RECOMMENDED

**Problem it solves:** After edit, user returns to listing — edited item not visible.

**How it works:**
- `PUT /items/:id` returns the **full updated item** in response body
- Angular uses a store action `patchItemInList` to **update only that item** in the existing in-memory list
- No new API call. No page reset.

```
PUT /items/95
        │
        ▼
Node.js: UPDATE items SET ... WHERE id=95 RETURNING *
        │
        ▼
Returns: { id: 95, name: "Updated", quantity: 15, ... }
        │
        ▼
Angular: store.dispatch(patchItemInList({ item: updatedItem }))
        │
        ▼
items = items.map(i => i.id === 95 ? updatedItem : i)
        │
        ▼
✅ User sees updated item in exact same position
```

**When to use:** Always. This is the baseline for all SaaS listing pages.

---

## Strategy 2: Scroll Position Restore ⭐ RECOMMENDED

**Problem it solves:** User returns to listing page — scroll resets to top.

**How it works:**
- Before navigating away, save `window.scrollY` to NgRx Store
- On listing component init, check if stored scroll position exists
- After data loads, use `requestAnimationFrame` to restore scroll

```
User clicks item row
        │
        ▼
Angular saves scrollY = 2400 to store
        │
        ▼
Navigates to /items/95/edit
        │
        ▼
User edits → navigates back
        │
        ▼
Listing component ngOnInit:
  - Data already in store (no re-fetch needed)
  - scrollY = 2400 found in store
        │
        ▼
requestAnimationFrame(() => window.scrollTo(0, 2400))
        │
        ▼
✅ User lands exactly where they left off
```

**When to use:** Any app where user navigates away and returns to a list.

---

## Strategy 3: Cursor / Keyset Pagination ⭐ RECOMMENDED FOR LARGE DATA

**Problem it solves:** Offset pagination causes duplicate/missing rows when data changes.

**How it works:**
- Instead of `OFFSET N`, use the last item's values as the cursor
- Cursor encodes: `{ updated_at: "2025-01-15T10:30:00Z", id: 94 }`
- Next page query: `WHERE (updated_at, id) < (cursor.updated_at, cursor.id)`

```
First load:
  GET /items?limit=40
  Cursor returned: eyJ1cGRhdGVkX2F0IjoiMjAyNS0wMS0xNVQxMDozMDowMFoiLCAiaWQiOjQwfQ==

Second load:
  GET /items?cursor=eyJ1cGRhdGVkX2F0Ij...&limit=40
  → Stable regardless of inserts/deletes/edits
```

**When to use:** Any dataset > 1,000 rows, or where inserts/deletes occur during browsing.

---

## Strategy 4: Optimistic UI

**Problem it solves:** Edit feels slow — user waits for API confirmation.

**How it works:**
- Update the in-memory store **immediately** (before API responds)
- Show a "saving..." indicator
- On success: confirm. On failure: rollback + show error.

```
User clicks Save
        │
        ▼
Angular: store.dispatch(patchItemOptimistic({ item: localUpdate }))
UI updates instantly ← user sees change immediately
        │
        ▼ (async)
PUT /items/95 call in progress...
        │
  ┌─────┴──────┐
SUCCESS       FAILURE
  │               │
Confirm        store.dispatch(rollbackItem({ id: 95 }))
               Show error toast
```

**When to use:** Toggle fields (status, active/inactive), quick edits. Not recommended for complex multi-field forms.

---

## Strategy 5: Pin Edited Item to Top (Temporary)

**Problem it solves:** Edited item is on page 3 — user can't find it after returning.

**How it works:**
- After edit, store `recentlyEditedId` in NgRx store
- On listing init, if `recentlyEditedId` exists, prepend that item to the list temporarily
- Highlight it for 3 seconds, then remove the pin

```
Angular store:
  items: [ ...normalList ],
  recentlyEditedId: 95

Listing component renders:
  [item #95 - HIGHLIGHTED "Recently edited"]  ← pinned to top
  [item #1]
  [item #2]
  ...
```

**When to use:** Chronological lists sorted by `created_at` where edited items would naturally be far from top.

---

## Strategy 6: WebSocket / SSE Push Updates

**Problem it solves:** Multiple users editing the same list — one user's edits don't appear on another's screen.

**How it works:**
- Node.js broadcasts item update events via WebSocket or Server-Sent Events
- All connected Angular clients receive the event and dispatch `patchItemInList`

```
User A edits item #95
        │
        ▼
Node.js: UPDATE ... RETURNING *
Node.js: ws.broadcast({ type: 'ITEM_UPDATED', item: updatedItem })
        │
        ▼
User B's Angular receives: { type: 'ITEM_UPDATED', item: { id: 95, ... } }
store.dispatch(patchItemInList({ item }))
        │
        ▼
✅ User B sees item #95 updated in real-time
```

**When to use:** Collaborative SaaS (shared dashboards, team task lists). Requires infrastructure investment.
