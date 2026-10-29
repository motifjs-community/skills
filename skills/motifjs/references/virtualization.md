# Virtualization (large lists)

Use `Virtualization<T>` for thousands of rows whose heights are fixed or computable in advance, or for
paged/infinite data. For < ~100 rows or heights known only after rendering use a normal `.map`.

```tsx
import { Component, reactive, Virtualization } from '@motifx/core';

interface Todo { id: number; title: string; completed: boolean }

export default class TodoList extends Component<HTMLDivElement> {
  state = reactive({ todos: [] as Todo[] });

  onConfig() {
    this.doWork(fetch('/api/todos').then(r => r.json())).then((todos) => {
      if (todos instanceof Error) return;
      this.state.todos = todos;
    });
  }

  view() {
    return <Virtualization<Todo>
      x-wait={() => this.state.todos.length === 0}
      itemHeight={48}
      pageSize={40}
      dataRequest={async ({ page, pageSize }) => {
        const start = page * pageSize, end = start + pageSize;
        return { items: this.state.todos.slice(start, end), totalCount: this.state.todos.length, hasMore: end < this.state.todos.length };
      }}
      itemTemplate={(todo, i) => (                 // called once per row; the component is reused while scrolling
        <div class="row">
          <input type="checkbox" checked={() => todo.completed} onchange={(e) => todo.completed = e.target.checked} />
          <span>{todo.title}</span>
        </div>
      )}
      mainTemplate={(content, st) => (
        <div class="list">
          <header>{() => st.data.length} loaded</header>
          <div class="scroll"><content></content></div>      {/* rows go here */}
        </div>
      )}
      loadingTemplate={(st) => <div class="spinner" />}
      emptyTemplate={() => <div>No rows</div>}
      errorTemplate={(err) => <div class="error">{err.message}</div>}
      autoLoad autoLoadThreshold={200}
    />;
  }
}
```

## Props (`VirtualizationProps<T>`)

| Prop | Type | Notes |
|---|---|---|
| `dataRequest` | `({ page, pageSize, scrollTop }) => Promise<{ items, totalCount, hasMore }>` | **required**; `page` is 0-based; `totalCount` fills `state.totalCount` and, while `hasMore` is true, sizes the scroll area: items not loaded yet take space below the rows (exact with a fixed `itemHeight`, averaged from the loaded rows with a function), scrolling into that space requests the pages in order up to it, a `totalCount` smaller than the loaded count is ignored, and `filter` applies to loaded items only; `hasMore: false` stops further page requests |
| `itemTemplate` | `(item, index) => JSX` | **required**; receives the item exactly as it was given (see "Data is used as given") |
| `itemHeight` | `number \| (item, index) => number` | **required**; a number for fixed rows, a function for known per-row heights (offsets are summed once per data change, lookup is a binary search). Heights are not tracked: when they change while the data stays the same (a row grows, a density setting, a new fixed number), call `updateHeights()` |
| `pageSize` | `number` | default 50 |
| `mainTemplate` | `(content, state) => JSX` | wrap the list; place `<content></content>` or `{content}` where rows go (see the placement rule below) |
| `loadingTemplate`, `emptyTemplate`, `errorTemplate` | | `errorTemplate(error)` shows failures of the first load, `refresh()` and `autoRefresh` reloads (without it they are reported as `MJX207`); a failing next-page request (`autoLoad`) is reported as `MJX207` and stored in `getState().error`, the loaded rows stay, no error view is shown, it is retried on the next scroll and a successful load clears `error` |
| `filter` | `(items: T[]) => T[]` | applied to the loaded data on every load (not reactive); spacers/indices use the filtered length; when it removes every loaded item, `emptyTemplate` is shown (nothing without it) and no further page is requested, so filter in `dataRequest` when later pages may match |
| `overscan` | `number` | extra rows rendered above **and** below the viewport; default = visible row count |
| `cacheSize` | `number` | max off-screen row components kept alive for reuse (LRU); default `max(200, 3 x window)` |
| `overscanPages`, `pageBuffer`, `renderMode` | | reserved; `'page'` mode is not implemented |
| `autoLoad`, `autoLoadThreshold` | | load next page when scrolled within `threshold` px (default 100) of the **loaded** data's bottom, and also when the rendered window comes within `overscan` rows of the loaded end (e.g. the first page does not fill the viewport); `autoLoad` defaults to on. With a fixed `itemHeight` the bottom is the **unfiltered** `data.length x itemHeight`, while the spacers use the filtered length; with a `filter` that hides rows the threshold check measures past the real content end and the window check (filtered length) is what loads the next page. With a height function both use the filtered rows |
| `autoRefresh` | `boolean` | default on: when reactive data read synchronously inside `dataRequest` changes, the loaded range (page 0 to the current page) is requested again in one call; rows of items that are still in the response are kept, the others are disposed; `false` → reload only through `refresh()`. Reads after an `await` in an inline `dataRequest` are tracked too (the compiler instruments it). The watch is set up by the first load and `refresh()`; `setData()` stops it. More than 30 reloads within one second stop the watch (a `dataRequest` that writes the data it reads); dev mode warns `MJX206` |
| `watch` | `() => unknown` | extra dependencies for `autoRefresh`: reactive fields read here also trigger a reload (fallback when the reads cannot be seen). A throw is reported as `MJX205` and the request still runs |
| `className`, `style` | | container styling |

`VirtualizationState<T>`: `{ isLoading, isInitialized, currentPage, totalCount, hasMore, error?, data }`.
The fields are reactive, so a getter in `mainTemplate` stays current; reading `state.data` re-runs when
the loaded or `setData` array changes, but its items are not wrapped.

## Data is used as given

The list does not turn the data into reactive objects. Items returned by `dataRequest` or passed to
`setData` reach `itemTemplate`, `itemHeight` and `filter` unchanged, and the array passed to `setData`
is the one `state.data` and `getState().data` return. Reactive data stays reactive, plain data stays plain.

- A row built from a plain object does not show later writes to that object. Pass the data through
  `reactive` when rows must follow changes (the example above reads `this.state.todos`, so its items
  are reactive), or re-run the row's bindings with `setState()` for plain data; pass a new array with
  `setData`.
- Next pages are not pushed into the first page's array: the list collects loaded items in its own
  array and leaves the arrays returned by `dataRequest` untouched.

Public fields: `wrapper` is the scroll container (its `scrollTop` drives the window). Without
`mainTemplate` it is the `Virtualization` element itself (`overflow-y: auto; height: 100%; position: relative; overflow-anchor: none`);
with `mainTemplate` it is the nearest ancestor of `content` that owns a real element, found when `content`
is built, and it receives `overflow-y: auto; position: relative; overflow-anchor: none` and the scroll listener, so give that
element a height. `contentWrapper` is the `content` component handed to `mainTemplate` (spacers + rows).

A `ResizeObserver` on the scroll container re-renders the visible rows when its height changes
(rotation, on-screen keyboard); it is skipped when `ResizeObserver` is missing and disconnected on
dispose. Heights that are only known after measuring (free-text feeds) are not supported.

## How it renders (what to expect in the DOM)

- The DOM holds only the window `[first, last)` = `[anchor - overscan, anchor + visible + overscan)`
  where `anchor = floor(scrollTop / itemHeight)`. Rows are plain siblings between two spacer
  `div`s (`.motif-virtualization-top-spacer` / `-bottom-spacer`); spacer heights are never negative
  and `top + rows + bottom = filteredLength x itemHeight`.
- A row leaving the window is **detached, not disposed**: the same component (its bindings, input
  state, item object) comes back when scrolled into view. No `<!--h-->` placeholders
  accumulate. Off-screen rows beyond `cacheSize` are disposed oldest-first.
- Every scroll event re-renders the window (also when `hasMore === false`); only the difference to
  the previous window is touched, so cost is O(window), not O(data).
- Rows are cached by item object identity (by index for primitive items); the same object may appear
  more than once in the data and every occurrence gets its own row.
- `setData()`, an `autoRefresh` reload and the arrival of the next page match rows by that identity: an
  object still in the new view keeps its row (on screen or cached) and the row moves to its new
  position; rows of objects that are gone are disposed (including those a `filter` drops when the next
  page arrives). A raw object and its reactive proxy find the same row; a primitive keeps its row
  when the same value stays at the same index. Fresh objects on every load (a new server list) rebuild
  the rows even with the same `id`. The `index` a kept row got from `itemTemplate` is the position when
  it was built and is not updated. `refresh()` disposes every cached row.
- When the data changes (`setData()`, `autoRefresh`, next page) the first row visible on screen that
  is still in the new data keeps its place on screen: items added or removed above (also next-page
  items that a sorting `filter` places above) shift `scrollTop` by their height, a new order takes the
  scroll to that row's new position. A primitive value is looked up at the position nearest to its old
  index. If no visible row survives, the surviving rendered row nearest to the screen is used, overscan
  rows above first, then those below; if none survives, `scrollTop` is kept as is. A row sticking out above the screen that becomes shorter stays partly
  visible. At `scrollTop` 0 the list stays at the top, so an item added on top is visible. Call
  `scrollToIndex(0)` after a re-sort to go back to the start. Positions are read from the rendered
  rows, so an array changed in place (`splice`, `sort`) and passed again still anchors correctly. The
  scroll container gets `overflow-anchor: none`: the list, not the browser, keeps the position; the
  browser's own scroll anchoring can move `scrollTop` while the list renders and leave a blank area.
- The first render waits for layout: if `clientHeight` is 0 (not yet in `document`) it is retried
  in `onMounted`. In jsdom tests fake `clientHeight`/`scrollTop` with `Object.defineProperty` and
  dispatch `new Event('scroll')` (see `packages/tests/src/components/virtualization.test.ts`).
- `refresh()` reloads page 0 through `dataRequest` and disposes every cached row (returns a
  `Promise`); `setData(items)` replaces the data without a request (`hasMore` becomes `false`, rows of items that
  stay are kept);
  `updateHeights()` rereads every height and renders the window when heights change while the data
  stays the same (rows kept, `filter` not rerun, nothing requested, the top visible row keeps its
  place); `scrollToIndex(i)` scrolls and renders synchronously; `getState()` returns a shallow copy of
  the state.
- A row that leaves the window receives `onDeactivated`; a cached row coming back receives
  `onActivated`.
- Paged mode: the scrollbar represents **loaded** pages only (the loaded rows after `filter` x row height), giving an
  infinite-scroll feel; it grows as pages arrive.

## Placement rule for `content`

Both forms place the rows component inside `mainTemplate`:

- `<content></content>`: the compiler turns **any** lowercase tag name that is not an HTML/SVG tag name
  into a variable reference, whether or not such a variable exists (an undeclared name fails at runtime),
  and a component instance is placed as is. Trap: a parameter named like an HTML tag (`header`, `main`,
  `section`, `slot`) produces that DOM element instead; keep the name `content` or another non-HTML name.
  A name with `-` stays a custom element.
- `{content}`: a bare identifier or member expression child is checked once, when the child is created: if
  it holds a component (or a non-empty array of components) it is placed; any other value becomes a
  text binding.
