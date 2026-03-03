# 06 — Architecture Diagrams

## Diagram 1: Full System Architecture

```mermaid
graph TB
    subgraph Angular["🖥️ Angular Frontend"]
        LP[Listing Page<br/>Infinite Scroll]
        EP[Edit Page<br/>Form]
        NS[NgRx Store<br/>items / cursor / scrollY]
        IS[IntersectionObserver<br/>Scroll Trigger]
    end

    subgraph NodeJS["⚙️ Node.js Backend"]
        LA[List API<br/>GET /api/items]
        UA[Update API<br/>PUT /api/items/:id]
        CE[Cursor Encoder<br/>base64url]
        SV[Items Service]
    end

    subgraph PostgreSQL["🗄️ PostgreSQL"]
        IT[(items table)]
        HT[(headers table)]
        CI[Composite Index<br/>updated_at, id]
        TR[Trigger<br/>recalculate_header]
    end

    LP -->|Load / Load More| LA
    LP -->|Navigate to edit| EP
    EP -->|PUT + full response| UA
    UA -->|patchItemInList| NS
    NS -->|Restore scroll| LP
    IS -->|Trigger load more| LA

    LA --> CE
    CE --> SV
    SV -->|Keyset query| IT
    UA -->|UPDATE RETURNING *| IT
    IT --> TR
    TR -->|Recalculate| HT
    IT --> CI
```

---

## Diagram 2: Edit → Back Flow (Sequence)

```mermaid
sequenceDiagram
    actor User
    participant Angular as Angular<br/>(Listing Page)
    participant Store as NgRx Store
    participant EditPage as Angular<br/>(Edit Page)
    participant NodeJS as Node.js API
    participant PG as PostgreSQL

    User->>Angular: Scrolls to item #95
    Angular->>Store: saveScrollPosition(scrollY=2400)
    Angular->>EditPage: navigate('/items/95/edit')

    User->>EditPage: Edits form fields
    User->>EditPage: Clicks Save

    EditPage->>NodeJS: PUT /api/items/95 { name, quantity... }
    NodeJS->>PG: UPDATE items SET ... WHERE id=95 RETURNING *
    PG-->>NodeJS: { id:95, name:"Updated", quantity:15, updated_at: NOW() }
    NodeJS-->>EditPage: 200 OK { id:95, name:"Updated", ... }

    EditPage->>Store: dispatch(patchItemInList({ item }))
    Note over Store: items.map(i => i.id===95 ? item : i)
    EditPage->>Angular: navigate('/items')

    Angular->>Store: select(scrollY)
    Store-->>Angular: 2400
    Angular->>Angular: requestAnimationFrame(scrollTo(2400))

    Note over Angular: ✅ Item #95 visible and highlighted<br/>✅ Scroll at position 2400<br/>✅ No API call made
```

---

## Diagram 3: Cursor Pagination Flow

```mermaid
sequenceDiagram
    participant Angular
    participant NodeJS
    participant PG as PostgreSQL

    Angular->>NodeJS: GET /api/items?limit=40
    NodeJS->>PG: SELECT ... ORDER BY updated_at DESC, id DESC LIMIT 41
    PG-->>NodeJS: 41 rows (hasMore = true)
    NodeJS-->>Angular: { items[0..39], cursor: "eyJ1cGRhdGVkX...", hasMore: true }

    Note over Angular: User scrolls to bottom

    Angular->>NodeJS: GET /api/items?cursor=eyJ1cGRhdGVkX...&limit=40
    NodeJS->>PG: SELECT ... WHERE (updated_at,id) < ($1,$2) LIMIT 41
    PG-->>Angular: { items[40..79], cursor: "newCursor", hasMore: true }

    Note over Angular: User edits item #55 (page 2)
    Note over Angular: Comes back → NO re-fetch, cursor intact ✅
```

---

## Diagram 4: Header Table Sync

```mermaid
sequenceDiagram
    participant NodeJS
    participant PG as PostgreSQL
    participant Items as items table
    participant Trigger as DB Trigger
    participant Headers as headers table

    NodeJS->>Items: UPDATE items SET quantity=15 WHERE id=95 RETURNING *
    Items-->>NodeJS: { id:95, quantity:15, header_id:3, ... }
    Items->>Trigger: AFTER UPDATE fires
    Trigger->>Headers: UPDATE headers SET total_quantity=SUM(quantity), last_updated_at=NOW() WHERE id=3
    Headers-->>Trigger: OK

    Note over NodeJS: Angular re-fetches header summary
    NodeJS->>Headers: SELECT * FROM headers WHERE id=3
    Headers-->>NodeJS: { total_quantity: 1005, item_count: 200, last_updated_at: NOW() }
```

---

## Diagram 5: NgRx State Machine

```mermaid
stateDiagram-v2
    [*] --> Empty : Component Init

    Empty --> Loading : dispatch(loadItems)
    Loading --> Loaded : loadItemsSuccess
    Loading --> Error : loadItemsFailed

    Loaded --> LoadingMore : dispatch(loadMoreItems)
    LoadingMore --> Loaded : loadMoreSuccess (append)
    LoadingMore --> Loaded : loadMoreFailed (keep existing)

    Loaded --> Loaded : patchItemInList (map + replace)
    Loaded --> Loaded : saveScrollPosition
    Loaded --> Loaded : setRecentlyEdited → clearRecentlyEdited (3s)

    note right of Loaded
        items: Item[]
        cursor: string | null
        hasMore: boolean
        scrollY: number
        recentlyEditedId: number | null
    end note
```

---

## Diagram 6: Strategy Selection Flowchart

```mermaid
flowchart TD
    A[User edits item<br/>on detail page] --> B{Does PUT response<br/>return full item?}

    B -->|NO - fix this first!| C[Update API to use<br/>RETURNING * in PostgreSQL]
    B -->|YES| D[dispatch patchItemInList]

    D --> E{Was scroll position<br/>saved before navigate?}

    E -->|NO| F[Add saveScrollPosition<br/>before router.navigate]
    E -->|YES| G[Restore scroll with<br/>requestAnimationFrame]

    G --> H{Dataset > 1000 rows?}

    H -->|YES| I[Switch to cursor<br/>keyset pagination]
    H -->|NO| J[Offset pagination OK<br/>for small datasets]

    I --> K{Real-time collaborative?}
    J --> K

    K -->|YES| L[Add WebSocket/SSE<br/>for live updates]
    K -->|NO| M[✅ Standard Practice Complete]

    L --> M
```
