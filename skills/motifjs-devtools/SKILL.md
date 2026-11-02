---
name: motifjs-devtools
description: "Give a MotifJS package (component library, data layer, store, router add-on) its own section in the MotifJS devtools (@motifx-community/devtools) with registerDevtools from @motifx/core/devtools: sources, commands, events and panels described as JSON widgets; for package authors, not for app developers who only use the panel."
license: MIT
metadata:
  author: MotifJS
  version: "1.0.0"
  homepage: https://motifjs.com
---

# MotifJS devtools panels for packages

The MotifJS devtools (`@motifx-community/devtools`, in-page panel and Chrome/Edge/Firefox extension) can show a
section of a package: a sidebar entry with widgets, sources it reads, commands it runs, events that change rows as
they happen, lanes on the Performance timeline and a way to point the user at something.

**A panel is data, not code.** The package registers plain JSON metadata (which widgets, which fields, which
sources); the panel draws it with widgets it ships itself. Nothing of the package runs in the panel and nothing of
the panel runs in the page. There is no component, no HTML, no CSS and no callback in the metadata.

For the app side of MotifJS (components, reactivity, the core devtools contract) see the `motifjs` skill and its
`references/api-cheatsheet.md`.

## Before you start

- The core must have the extension methods of the devtools contract (`readExtensionPanels`): `@motifx/core`
  1.2.0-rc.24 or later. On an older core only `describe()` shows, as a value tree.
- Devtools are on only after `app.useDevelopment(true)`. Registration costs nothing while they are off;
  `describe`, `inspect`, `sources`, `commands` and a `panels` function run only when the panel asks.
- `@motifx-community/devtools` is a type-only import (`PanelMetadata`, `Widget`, every widget type). Nothing of it
  is loaded at runtime, so the package does not depend on the panel.

## Minimum: describe only

```ts
import { registerDevtools } from '@motifx/core/devtools';

const devtools = registerDevtools('my-store', {
  describe: () => ({ items: store.items.length, pending: store.pending }),  // plain JSON-safe copy
});
```

This alone gives a panel named after the package with `describe()` as a value tree, read again on every event of
the package and on every reactivity flush while it is open. Add `inspect: (component) => ...` to show what the
package knows about a component in the Extensions tab of Elements.

## Full shape

```ts
import { registerDevtools } from '@motifx/core/devtools';
import type { PanelMetadata } from '@motifx-community/devtools';

const panel: PanelMetadata = {
  id: 'orders', title: 'Orders', icon: 'list', order: 100,
  capabilities: { read: true, subscribe: true, write: true },
  layout: { type: 'table', source: 'orders', key: 'id', select: 'orderId',
            columns: [{ field: 'id' }, { field: 'total', format: 'number' }],
            open: { route: 'order', params: { id: { field: 'id' } } } },
  routes: [{ id: 'order', title: { field: 'id' }, source: 'order', params: { id: { param: 'id' } },
             layout: { type: 'detail', source: 'order', params: { id: { param: 'id' } }, sections: ['lines', 'status'] } }],
  lanes: [{ id: 'saves', label: 'Saves', event: 'order:saved', itemDuration: 'duration', itemTitle: 'id' }],
};

const devtools = registerDevtools('orders', {
  describe: () => ({ count: orders.length }),
  sources: {
    orders: { read: () => orders.map(toRow), key: 'id', events: { 'order:saved': 'upsert', 'order:removed': 'remove' }, max: 5000 },
    order: { read: ({ id }) => toDetail(byId.get(id)) },
  },
  commands: { remove: ({ id }) => removeOrder(id) },
  panels: [panel],
});
// devtools.emit(type, payload) / devtools.changed(source?) / devtools.reveal({ panel, route?, params?, state? }) / devtools.dispose()
// devtools.pathOf(component) gives a component's path for the `component` format
```

Widgets: `stack`, `toolbar`, `split`, `tabs`, `text`, `counter`, `badge`, `json`, `code`, `button`, `filter`,
`table`, `tree`, `detail`, `chart`, `log`, `timeline`. Their fields are in `references/extensions.md`; the exact
types are in `@motifx-community/devtools` (`schema.ts`).

## Rules

1. **Metadata is JSON.** Literals, arrays and objects only. Values that depend on something use `{ state: 'key' }`
   (the panel's state), `{ param: 'id' }` (a parameter of the open route) and, in a row's `open`, `{ field: 'id' }`
   (a field of that row). No expressions, no templates, no functions. Fields are dotted paths: `summary.duration`.
2. **Sources return plain JSON-safe data.** Map to new plain objects (`orders.map(toRow)`); never hand over a
   reactive store, a class instance, a component, a DOM node or a cyclic value. The bridge copies the answer and
   marks what JSON cannot carry.
3. **A list source is a top-level array with `key`.** Events change rows only when the source's value is an array;
   `upsert` and `remove` find the row by `key`. Put the list at the top, not under a field.
4. **Events:** `upsert` replaces the row with the same key or adds it, `append` adds, `remove` takes the row with
   the payload's key out, `refresh` reads the source again. `max` keeps the newest rows (5000 unless given, 50 000
   at most). Emit the same row shape the source returns.
5. **Commands run only from a panel with `capabilities.write`.** Without it buttons stay disabled and
   `callExtension` is refused. `read` lets a panel read sources, `subscribe` lets events reach it; all three are
   false unless given. A thrown error becomes the notice under the panel title; a returned promise is awaited.
6. **Ids:** panel, route and lane ids are letters a-z, digits and dashes, at most 40; panel ids are unique in the
   package. Icons come from the panel's list; an unknown name gets the puzzle piece.
7. **Routes** open from a button `navigate`, from a row's `open` (double click in `table`, `tree`, `log`,
   `timeline`) or from the page with `reveal`. Their widgets read parameters with `{ param }`.
8. **`reveal` does not steal focus.** It writes `state` and opens `route` (or closes the open route when none is
   named). With the section open it applies at once; with the panel app shown and the user not typing in it, the
   app switches to the section; otherwise nothing moves, the section gets a mark in the sidebar and the target
   applies when the user opens it. Use it for something worth a look (a failure), not for every change.
9. **Stay under the limits** below; past them data is cut, dropped or refused.
10. **Store rules hold by design.** The browser stores forbid running code from outside the extension. Nothing a
    package sends is evaluated, written as HTML (`innerHTML`) or loaded from an address. Do not try to work around
    it with HTML strings or image URLs; the look comes from tones (`accent`, `info`, `success`, `warning`,
    `danger`, `muted`) and formats only.
11. **History packages** use `runOperation(info, fn)` and `onWrite(fn)` from `@motifx/core/devtools` to know which
    store writes belong to which piece of work. Nothing survives an `await`; resume by hand. See
    `references/example-actions.md`.

## Limits (defaults)

| What | Limit | Past it |
|---|---|---|
| events | 1000 per package per second | dropped, the panel shows "N events dropped" |
| one event | 64 KB after copying, depth 6 | replaced by its size |
| panel metadata | 256 KB, 2000 objects, depth 32 per package | refused whole, the sidebar shows the problem |
| one source answer | 1 MB | a list is cut ("N more rows"), a single value goes shallower |
| a read or a command | 10 s; 4 in flight per package | `timeout`, `busy` |
| count | 32 packages, 8 panels per package | extra packages counted; more panels: `too-many-panels` |
| rows in the panel | `max` of the source (5000, at most 50 000) | the oldest go |

The panel reads from the page only while it is shown and the section is open, at most every 250 ms.

## Testing checklist

- Turn devtools on (`app.useDevelopment(true)`) and open the section in the in-page panel or the extension.
- Emit after the mutation is done, with the final row (`emit('order:saved', toRow(order))`), never before.
- Every source and event payload is a plain copy; changing the store afterwards must not change what was sent.
- When the data changed in a way no event describes, call `devtools.changed('source')` (or `changed()` for all).
- Check a panel without `write`: buttons disabled, nothing changes the app.
- Check `reveal` with the panel closed: a mark in the sidebar, applied when opened.
- Check a big store: lists cut with "N more rows", the panel stays responsive.
- On a command error, the notice under the title says what went wrong in words the user understands.

## Reference map

| Need | File |
|---|---|
| Every widget and its fields, panel state, params, routes, reveal, events and lanes, limits, store rules | `references/extensions.md` |
| A whole package: action history with `runOperation` + `onWrite`, sources, commands, split layout with timeline, log and detail, a route, a lane, reveal on failure | `references/example-actions.md` |
| The core contract (`registerDevtools` handle, operations, write records, devtools events) | `motifjs` skill, `references/api-cheatsheet.md` |

When unsure about a metadata field, the ground truth is the `PanelMetadata` type exported by
`@motifx-community/devtools` and `registerDevtools` in `node_modules/@motifx/core/dist/devtools.d.ts`.
