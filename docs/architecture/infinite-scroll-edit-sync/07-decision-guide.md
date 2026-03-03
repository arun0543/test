# 07 — Decision Guide: When to Use What

## Quick Decision Tree

```
Does your listing page use infinite scroll or pagination?
        │
        ▼
       YES
        │
    ┌───┴────────────────────────────────────────────────┐
    │                                                    │
Can users edit items and go back?            Read-only listing?
        │                                                │
       YES                               → No special handling needed
        │
    ┌───┴────────────────────────────────────────────────────────────────────┐
    │                                                                        │
    ▼                                                                        ▼
How large is the dataset?                    Is it real-time / collaborative?
    │                                                        │
    ├── < 1,000 rows                              YES → Strategy 6 (WebSocket/SSE)
    │     → Strategy 1 + 2                        NO → Strategy 1 + 2 + 3
    │       (cache patch + scroll restore)
    │
    ├── 1,000 – 100,000 rows
    │     → Strategy 1 + 2 + 3
    │       (cache patch + scroll restore + cursor pagination)
    │
    └── > 100,000 rows
          → Strategy 1 + 2 + 3 + 4
            (add optimistic UI for fastest perceived performance)
```

---

## Feature-by-Feature Guide

### When to use each Angular feature

| Feature | Use When | Skip When |
|---------|----------|-----------| 
| NgRx Store for items list | Always — even simple apps | Never skip this |
| Scroll position save/restore | User navigates away and returns | Single-page no navigation |
| patchItemInList reducer | User can edit items | Read-only listing |
| recentlyEditedId highlight | Edit is non-obvious to user | Real-time auto-refresh |
| IntersectionObserver | Infinite scroll | Traditional pagination |
| Optimistic UI | High-frequency edits (toggles, status) | Complex multi-field forms |

### When to use each Node.js feature

| Feature | Use When | Skip When |
|---------|----------|-----------| 
| Cursor/keyset pagination | > 1,000 rows, sorted by date | Very small datasets |
| RETURNING * on UPDATE | Always — enables cache patching | Never skip this |
| Optimistic locking (updated_at check) | Multiple users can edit same records | Single-user apps |
| Cursor encoding (base64url) | Exposing cursors to frontend | Internal only APIs |

### When to use each PostgreSQL feature

| Feature | Use When | Skip When |
|---------|----------|-----------| 
| Composite cursor index (updated_at, id) | Cursor pagination | Offset-only pagination |
| RETURNING clause on UPDATE | Always | Never skip this |
| Trigger for header recalculation | Header shows aggregate of items | Header is independent |
| Optimistic lock (updated_at WHERE) | Concurrent multi-user edits | Single-user workflows |

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Re-fetching the whole list after edit

```typescript
// BAD
this.editService.updateItem(id, data).subscribe(() => {
  this.store.dispatch(ItemsActions.loadItems());  // Resets to page 1!
});

// GOOD
this.editService.updateItem(id, data).subscribe(updatedItem => {
  this.store.dispatch(ItemsActions.patchItemInList({ item: updatedItem }));
});
```

### ❌ Anti-Pattern 2: PUT returns only success/failure

```typescript
// BAD API response
res.json({ success: true });  // Angular can't patch the list!

// GOOD API response
res.json(updatedItem);  // Full item object via RETURNING *
```

### ❌ Anti-Pattern 3: Using OFFSET for large datasets

```sql
-- BAD: Unstable, slow on large tables
SELECT * FROM items ORDER BY updated_at DESC LIMIT 40 OFFSET 2000;

-- GOOD: Stable, uses index efficiently
SELECT * FROM items
WHERE (updated_at, id) < ($1, $2)
ORDER BY updated_at DESC, id DESC
LIMIT 40;
```

### ❌ Anti-Pattern 4: No scroll position memory

```typescript
// BAD: Navigate away and lose scroll position
this.router.navigate(['/items', id, 'edit']);

// GOOD: Save first, then navigate
this.store.dispatch(ItemsActions.saveScrollPosition({ scrollY: window.scrollY }));
this.router.navigate(['/items', id, 'edit']);
```

### ❌ Anti-Pattern 5: Destroying list state on navigation

```typescript
// BAD: No route reuse — component is destroyed and recreated
// Angular default behaviour wipes in-memory list

// GOOD: Use RouteReuseStrategy or keep list in NgRx store (not component state)
// Store state survives navigation; component reads from store on init
```

---

## Implementation Checklist

### Backend (Node.js + PostgreSQL)

- [ ] PUT /items/:id returns full updated item via RETURNING *
- [ ] Pagination uses cursor/keyset, not OFFSET
- [ ] Cursor is encoded as base64url (opaque to frontend)
- [ ] Header table has trigger or is recalculated on item update
- [ ] UPDATE uses optimistic locking for concurrent edit protection
- [ ] Composite index on (updated_at DESC, id DESC) exists

### Frontend (Angular)

- [ ] Items list is stored in NgRx Store, not component state
- [ ] loadMoreItems appends to list (not replaces)
- [ ] patchItemInList reducer updates only the edited item
- [ ] Scroll position saved to store before navigation
- [ ] Scroll position restored with requestAnimationFrame on return
- [ ] Recently edited item highlighted with CSS animation
- [ ] trackBy: trackById used on *ngFor for performance
- [ ] IntersectionObserver used for scroll trigger (not scroll events)
- [ ] List not re-loaded if store already has data (check length > 0)

---

## Summary: The Complete Standard Practice

```
┌──────────────────────────────────────────────────────────────┐
│           STANDARD PRACTICE FOR SAAS LISTING PAGES          │
│                                                              │
│  1. CURSOR PAGINATION (DB + API)                             │
│     Stable ordering — edits don't shift pages               │
│                                                              │
│  2. RETURNING * on UPDATE (DB)                               │
│     Full item returned — enables in-memory patching          │
│                                                              │
│  3. patchItemInList (Angular NgRx)                           │
│     Single item updated — no re-fetch, instant feedback      │
│                                                              │
│  4. SCROLL POSITION SAVE + RESTORE (Angular)                 │
│     User returns to exact position — seamless UX             │
│                                                              │
│  5. RECENTLY EDITED HIGHLIGHT (Angular CSS)                  │
│     Visual confirmation — user knows edit was saved          │
│                                                              │
│  6. HEADER TABLE TRIGGER (PostgreSQL)                        │
│     Summary always accurate — no stale aggregates           │
└──────────────────────────────────────────────────────────────┘
```
