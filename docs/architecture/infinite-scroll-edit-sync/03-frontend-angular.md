# 03 — Frontend: Angular Implementation

## NgRx State Design

```typescript
// state/items.state.ts
export interface ItemsState {
  items: Item[];
  cursor: string | null;        // For keyset pagination
  hasMore: boolean;
  loading: boolean;
  scrollY: number;              // Saved scroll position
  recentlyEditedId: number | null;
}

export const initialState: ItemsState = {
  items: [],
  cursor: null,
  hasMore: true,
  loading: false,
  scrollY: 0,
  recentlyEditedId: null,
};
```

---

## NgRx Actions

```typescript
// state/items.actions.ts
export const ItemsActions = createActionGroup({
  source: 'Items',
  events: {
    'Load Items': emptyProps(),
    'Load Items Success': props<{ items: Item[], cursor: string | null, hasMore: boolean }>(),
    'Load More Items': emptyProps(),
    'Patch Item In List': props<{ item: Item }>(),     // ← KEY ACTION
    'Save Scroll Position': props<{ scrollY: number }>(),
    'Set Recently Edited': props<{ id: number }>(),
    'Clear Recently Edited': emptyProps(),
  }
});
```

---

## NgRx Reducer: The Critical Patch

```typescript
// state/items.reducer.ts
export const itemsReducer = createReducer(
  initialState,

  on(ItemsActions.loadItemsSuccess, (state, { items, cursor, hasMore }) => ({
    ...state,
    items,
    cursor,
    hasMore,
    loading: false,
  })),

  // ⭐ KEY: Patch only the edited item — no re-fetch
  on(ItemsActions.patchItemInList, (state, { item }) => ({
    ...state,
    items: state.items.map(i => i.id === item.id ? item : i),
    recentlyEditedId: item.id,
  })),

  on(ItemsActions.saveScrollPosition, (state, { scrollY }) => ({
    ...state,
    scrollY,
  })),

  on(ItemsActions.clearRecentlyEdited, (state) => ({
    ...state,
    recentlyEditedId: null,
  })),
);
```

---

## NgRx Effects: Load & Load More

```typescript
// state/items.effects.ts
@Injectable()
export class ItemsEffects {

  loadItems$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ItemsActions.loadItems),
      exhaustMap(() =>
        this.itemsService.getItems({ limit: 40 }).pipe(
          map(({ items, cursor, hasMore }) =>
            ItemsActions.loadItemsSuccess({ items, cursor, hasMore })
          )
        )
      )
    )
  );

  loadMore$ = createEffect(() =>
    this.actions$.pipe(
      ofType(ItemsActions.loadMoreItems),
      withLatestFrom(this.store.select(selectCursor)),
      exhaustMap(([, cursor]) =>
        this.itemsService.getItems({ cursor, limit: 40 }).pipe(
          map(({ items, cursor: newCursor, hasMore }) =>
            ItemsActions.loadItemsSuccess({
              items: [...existingItems, ...items],  // APPEND
              cursor: newCursor,
              hasMore
            })
          )
        )
      )
    )
  );
}
```

---

## Listing Component

```typescript
// listing.component.ts
@Component({
  selector: 'app-listing',
  templateUrl: './listing.component.html',
})
export class ListingComponent implements OnInit, AfterViewInit, OnDestroy {

  items$ = this.store.select(selectItems);
  hasMore$ = this.store.select(selectHasMore);
  recentlyEditedId$ = this.store.select(selectRecentlyEditedId);

  private scrollObserver!: IntersectionObserver;

  constructor(
    private store: Store,
    private router: Router,
  ) {}

  ngOnInit(): void {
    // Only load if store is empty (don't re-fetch on back navigation)
    this.store.select(selectItems).pipe(
      take(1)
    ).subscribe(items => {
      if (items.length === 0) {
        this.store.dispatch(ItemsActions.loadItems());
      }
    });

    // Clear highlight after 3 seconds
    this.recentlyEditedId$.pipe(
      filter(id => id !== null),
      delay(3000),
      take(1)
    ).subscribe(() => {
      this.store.dispatch(ItemsActions.clearRecentlyEdited());
    });
  }

  ngAfterViewInit(): void {
    // Restore scroll position after view renders
    this.store.select(selectScrollY).pipe(take(1)).subscribe(scrollY => {
      if (scrollY > 0) {
        requestAnimationFrame(() => window.scrollTo({ top: scrollY, behavior: 'instant' }));
      }
    });

    // Setup infinite scroll trigger
    this.setupScrollObserver();
  }

  private setupScrollObserver(): void {
    const sentinel = document.querySelector('#scroll-sentinel');
    this.scrollObserver = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        this.store.dispatch(ItemsActions.loadMoreItems());
      }
    }, { threshold: 0.1 });

    if (sentinel) this.scrollObserver.observe(sentinel);
  }

  onRowClick(item: Item): void {
    // ⭐ Save scroll position before navigating away
    this.store.dispatch(ItemsActions.saveScrollPosition({ scrollY: window.scrollY }));
    this.router.navigate(['/items', item.id, 'edit']);
  }

  trackById(index: number, item: Item): number {
    return item.id;
  }

  ngOnDestroy(): void {
    this.scrollObserver?.disconnect();
  }
}
```

---

## Listing Template

```html
<!-- listing.component.html -->
<div class="listing-container">

  <!-- Header Summary Table -->
  <div class="header-summary">
    <app-header-summary [header]="header$ | async"></app-header-summary>
  </div>

  <!-- Items List -->
  <div class="items-list">
    <div
      *ngFor="let item of items$ | async; trackBy: trackById"
      class="item-row"
      [class.recently-edited]="(recentlyEditedId$ | async) === item.id"
      (click)="onRowClick(item)"
    >
      <span class="item-id">{{ item.id }}</span>
      <span class="item-name">{{ item.name }}</span>
      <span class="item-status">{{ item.status }}</span>
      <span class="item-updated">{{ item.updated_at | date:'short' }}</span>
    </div>
  </div>

  <!-- Infinite Scroll Sentinel -->
  <div id="scroll-sentinel" *ngIf="hasMore$ | async">
    <div class="loading-indicator">Loading more...</div>
  </div>

</div>
```

---

## CSS: Edit Highlight Animation

```scss
/* listing.component.scss */
.item-row {
  transition: background-color 0.3s ease;
  cursor: pointer;

  &:hover {
    background-color: #f5f5f5;
  }

  &.recently-edited {
    animation: editHighlight 3s ease-out forwards;
  }
}

@keyframes editHighlight {
  0%   { background-color: #e8f5e9; }  /* Green flash */
  20%  { background-color: #c8e6c9; }
  100% { background-color: transparent; }
}
```

---

## Edit Component: The Critical PUT

```typescript
// edit.component.ts
export class EditComponent {

  onSave(formData: Partial<Item>): void {
    this.itemsService.updateItem(this.itemId, formData).subscribe({
      next: (updatedItem) => {
        // ⭐ Patch the in-memory list with the returned item
        this.store.dispatch(ItemsActions.patchItemInList({ item: updatedItem }));
        this.router.navigate(['/items']);  // Go back — scroll will be restored
      },
      error: (err) => {
        this.showError('Failed to save. Please try again.');
      }
    });
  }
}
```
