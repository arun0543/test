# 01 — Problem Deep Dive

## 🔍 What Goes Wrong?

### Scenario Walkthrough

```
User loads listing page
        │
        ▼
API: GET /items?limit=40&offset=0
Returns items 1–40
        │
        ▼
User scrolls → API: GET /items?limit=40&offset=40
Returns items 41–80
        │
        ▼
User scrolls → API: GET /items?limit=40&offset=80
Returns items 81–120
        │
        ▼
User clicks item #95 → navigates to /items/95/edit
        │
        ▼
User edits item #95 → submits
        │
        ▼
User clicks ← Back to listing
        │
        ▼
❌ API: GET /items?limit=40&offset=0  ← PROBLEM: resets to page 1!
   Item #95 is not visible
   Scroll position is lost
   User has NO feedback that their edit worked
```

---

## 🧠 Root Causes

### 1. Offset-Based Pagination is Unstable

Offset pagination (`OFFSET 80`) counts rows from the beginning every time.

**Problem:** If any row is inserted, deleted, or reordered between requests, the offset shifts and you see **duplicate or missing rows**.

```sql
-- This query is UNSTABLE for pagination
SELECT * FROM items
ORDER BY created_at DESC
LIMIT 40 OFFSET 80;  -- What is "row 80" changes on every insert!
```

### 2. No In-Memory Cache Patching

When the user edits item #95 via `PUT /items/95`, the frontend typically:
- ❌ Discards the response
- ❌ Navigates back
- ❌ Re-fetches from scratch

What it **should** do:
- ✅ Receive the full updated item in the PUT response
- ✅ Patch only that item in the in-memory list
- ✅ Navigate back — user sees the change instantly

### 3. No Scroll Position Memory

Angular's default router behavior **destroys and recreates** components on navigation.
When the user goes back, the page starts at top with no scroll memory.

### 4. Header Table Out of Sync

Header tables (summary/aggregate data) may become stale after an edit:

```
items table:       item #95 updated (quantity: 10 → 15)
header table:      still shows total_quantity = 1000  ← stale!
```

---

## 💥 Impact on User Experience

| What Happens | User Perception |
|---|---|
| List resets to item 1–40 | "Did my edit even save?" |
| Scroll resets to top | "I have to scroll all the way down again" |
| No visual confirmation | "I'll click Save again" → duplicate edits |
| Header shows stale data | "The summary numbers are wrong" |

---

## ✅ What Good UX Looks Like

```
User edits item #95 → clicks Save
        │
        ▼
API returns updated item #95 in response body
        │
        ▼
Angular patches item #95 in the in-memory list (no re-fetch)
        │
        ▼
Angular navigates back, restores scroll to previous position
        │
        ▼
✅ User sees item #95 highlighted with updated data
✅ Header shows updated summary
✅ Scroll is exactly where they left off
```
