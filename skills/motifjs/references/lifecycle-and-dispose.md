# Lifecycle, visibility and disposal

## Hook order

| # | Hook | When | Typical use |
|---|---|---|---|
| 1 | `onInitializing(sender, e)` | constructor, after the element is assigned and props are applied (`this.props` is set; subclass fields such as `state = reactive(...)` are not initialized yet) | rarely |
| 2 | `onInitialized` | constructor, right after `onInitializing` | derive config from props |
| 3 | `onConfig` | **props ready, not in DOM** (subclasses: start of `build()`, after class fields; plain `new Component(tag, { onConfig })`: constructor) | fetch data, set up bindings/watchers |
| 4 | `onConfigured` | config phase done (start of build) | |
| 5 | `onBuilding` | DOM construction starts | |
| 6 | `initializeComponent(sender)` | right before `view()`; the compiler puts the code generated from JSX here | `sender.bindings.*` calls when building without JSX |
| 6b | `oninitializeComponent(sender, e)` | right after all `initializeComponent` code (class method, tag value, compiler-generated), before `view()`; runs in order: class method, then tag props; on a function component tag it applies to the returned root | your own setup that must run after the compiler-generated setup (attributes, events, bindings and JSX children already exist) |
| 7 | `onBuilt` | component + children built (element may still be inside a detached fragment) | wiring that does not need layout |
| 8 | `onMounted` | once, in a later task after the component is placed under a root that is in a document: the `run()` host or a `new Component(existingElement)` wrapper, every component on the way visible (also when the tree already was in the page); the screen may already be painted | only work that needs the element in the page and visible: focus, measure, a library that needs the element in the document; anything else goes in `onBuilt` |
| – | `onActivated` / `onDeactivated` | an already built component is removed from the DOM without disposal (`onDeactivated`) and put back (`onActivated`): `keepAlive` route return, `controls.detach`/`silentDetach` then `controls.add` (incl. moving to another parent), `motif.hide()`/`motif.show()` (incl. `x-wait`/`x-display`), Virtualization rows scrolled out/in. Not on first appearance. Propagates to the visible subtree, parent first; a hidden child is activated by its own `motif.show()` | refresh data, pause timers |
| – | `onVisibilityChanged` | when `motif.show()/hide()/toggle()` (incl. `x-wait`/`x-display`) changes visibility; it runs before `isVisible` is updated (on hide after the leave animation), so inside it `isVisible` still holds the previous value and `e.visible` (`VisibilityChangedEventArgs`) the new one | |
| – | `onDisposing` | teardown begins | manual cleanup |
| – | `onDisposed` | teardown finished | |

1–2 run in the constructor. `onConfig` runs in the constructor for plain `Component` instances, but
for **subclasses** (`class X extends Component`) it is deferred to the start of `build()` so that
class fields (`state = reactive(...)`, injected services) are already initialized — it still runs
before `onConfigured`. 4–7 run inside `build()`; `onMounted` fires when the component is placed under a
root that is in a document (same mechanism as `x:mounted`; a pending hook dies with the component) and runs
in a later task on the root's window: the queue is processed in 8 ms slices, so many heavy mounts (wrapped widgets) do not block the page; the browser
paints and handles input between slices. Children run before their parent; a component disposed before
its turn is skipped. Every hook receives
`(sender, e: { cancel: boolean })`. Hooks may be `async`. A subclass that is never built never
receives `onConfig`. A hook that throws, or returns a promise that rejects, is reported as `MotifError`
`MJX122` (`console.error` and `errorHandler.addListener` listeners, in production too) and the component
keeps building; the same applies to `initializeComponent`, `oninitializeComponent`, an options-object `ctor`,
a `ref` callback on a plain tag or class component tag (the component is still created) and listeners
added in code with `this.motif.on('x:built' | 'x:configured' | 'x:disposed' | 'x:mounted' | …, fn)`
(`The component x:built hook threw.`). `isInitialized` is `false` during `onInitializing` and `true` from
`onInitialized` on.

Define them as class methods **or** pass as JSX props (`onConfig`, `onBuilt`, `onDisposing`, …
or `x-config`, `x-built`, `x:built`, …). Both run; prop handlers are additive. Several spellings of
the same hook on one tag (`onBuilt={a} x-built={b} x:built={c}`) are merged and all run once,
in source order, on plain DOM tags and component tags alike.

```tsx
class Chart extends Component<HTMLDivElement, { data: number[] }> {
  chart?: ExternalChart;
  onBuilt() {                                     // the element and its content exist here
    this.chart = new ExternalChart(this.element, this.props.data);
    this.motif.setDisposable(() => this.chart?.destroy());
  }
}
```

## Visibility

```ts
await this.motif.hide();     // plays leave transition, then hides
await this.motif.show();     // enter transition
this.motif.toggle();
this.isVisible;
```
`this.motif.options.hideStrategy` (set through the `options` JSX prop or in code): `'placeholder'` swaps
in a `<!--h-->` comment trace (fast), `'detach'` removes from DOM without a trace, `'auto'` (default) is
`'placeholder'` everywhere, list rows included. A component that waits from the start is not built but its trace is
already in place.

A fragment component hides like an element: each visible child swaps in its own trace and plays its own leave, the
fragment is notified first, and the children keep their own `isVisible`. A child added while the fragment is hidden
(new, already built, or moved from another parent) stays hidden; its place is its position in `controls`, and it goes
there when the fragment is shown, with its own `hideStrategy` ignored. A child that is hidden itself stays hidden.

`x-display={() => bool}` is the declarative form.

## Wait state

```tsx
<Virtualization x-wait={() => state.items.length === 0} … />
```
While the predicate is `true` the component is not inserted; when it turns `false` it is built and
inserted. If it starts `true`, `onConfig`/`onConfigured` run but `onBuilding`/`onBuilt`/`onMounted` wait for
the release. Returning to the wait state hides the component without disposing it (`onDeactivated`).
Imperative: `this.isWait = true/false`.

## Disposal

```ts
await this.dispose();                                // deep, with leave transition
await this.dispose({ skipLeaveTransition: true });
await this.disposeAsync({ deep: true });
await this.motif.clear();                            // dispose children only (no-op with motif.options.disableDisposal)
this.controls.remove(child);                         // dispose one child
this.isDisposed;                                     // most methods become no-ops afterwards
```

`IDisposeOptions`: `skipLeaveTransition` makes `dispose()` skip the leave animation (`disposeAsync()`
does not read it: it stops running animations and removes the element without a leave transition, and it
is the path children take when their parent is disposed, so an animated child never holds its parent or a
conditional branch back). `deep` (default `true`, passed on to the children) makes `disposeAsync()` clear the
instance's own fields afterwards (`element`, `props`, `class`, `attr`, `motif.options`, every own
property); `disposeAsync({ deep: false })` keeps them. `dispose()` clears them whatever `deep` says.

### What is cleaned automatically
DOM listeners from `motif.on()`/JSX, all bindings and watchers, children (recursively), DOM nodes,
internal references, and subscriptions opened through `this.context.on(...)` /
`this.context.onRouterChanged(...)`. **You never clean up things you created through JSX.**
Only those two `context` members are scoped to the component: `this.context.onLifecycle(...)` goes
straight to the application and stays subscribed after dispose.

### What you must register
Timers, observers, sockets, external subscriptions, raw `effect()` stops, app-event unsubscribes
opened on `Application.main` / a captured `app` (not through `this.context`), and every
`onLifecycle(...)` unsubscribe (also through `this.context`):

```ts
onConfig() {
  const id = setInterval(() => this.tick(), 1000);
  this.motif.setDisposable(() => clearInterval(id));

  this.motif.setDisposable(this.context.onLifecycle(({ state }) => { if (state === 'resumed') this.refresh(); }));

  const stop = effect(() => console.log(store.x));
  this.motif.setDisposable(stop);

  this.motif.register(disposableCore.toDisposable(() => ws.close()));   // IDisposable form
}
```

`DisposableStore` groups many disposables: `store.add(d); store.dispose(); store.isDisposed`.

### Dispose-safe async

```ts
this.using(fetchData(), (data) => { this.state.items = data; });   // callback skipped if disposed; a throwing callback is reported as MJX129
this.doWork(fetchData())                                           // Promise<T | MotifError>
  .then((data) => { if (data instanceof Error) return; this.state.items = data; })  // MJX108 once disposed
  .catch((error) => errorHandler.report(error));                  // rejections while alive; once disposed the rejection is the cause of MJX108
```

Both look at `isDisposed` when the result arrives. The flag turns true when disposal finishes, not when `dispose()` is called, so a result that arrives while the component is being disposed is still handled (its bindings are already stopped). A hidden, detached or moved component is not disposed; its result is handled. Without `onrejected`, `using` drops a rejection silently. `using` returns nothing; use `doWork` to await, chain or return the work. A `finally` after `doWork` runs in both cases, so do not touch the component there. Neither cancels the request: to cancel it, pass an `AbortController` signal to `fetch` and register `controller.abort()` with `motif.setDisposable`; that runs after `isDisposed` is true, so `using` drops the `AbortError` and `doWork` resolves to MJX108 with it as `cause`. Pick `using` when the result only goes into this component; pick `doWork` to chain steps, combine requests (`doWork(Promise.all([...]))`) or return the work to a caller.

### Leak monitoring (dev)

```ts
app.useReactiveMonitor({ enabled: true, threshold: 200, name: 'app' });
```
Warns (`MJX507`) when the number of effects tracking one reactive field reaches the threshold (200 by
default when enabled), which usually means an effect or binding was never released. Off by default; `configureReactivityLeakMonitor({...})` is the same switch
without an `Application`.

### Counting live components (tests)

Every component is a `Disposable`: its constructor calls `trackDisposable`, disposal calls
`markAsDisposed`. Plug a tracker into `disposableCore.disposableTracker` to count live instances in a
create/dispose loop:

```ts
import { disposableCore, ComponentBase } from '@motifx/core';

let alive = 0;
disposableCore.disposableTracker = {
  trackDisposable(x) { if (x instanceof ComponentBase) alive++; },
  markAsDisposed(x) { if (x instanceof ComponentBase) alive--; },
  setParent() {}, markAsSingleton() {},
};
```
`markAsDisposed` arrives once per object; calling `dispose()` again does not report it again.
Reactive dependency maps are available from `import { debugGetDeps, debugGetDepMap } from '@motifx/core/devtools'`.
With `app.useDevelopment(true)`, `__MOTIF_DEVTOOLS__.getCounters().components.notInTree` counts living components
that are not in the tree; if it does not drop after leaving a page, those components are still held somewhere.

## Wrapping third-party libraries (charts, editors, maps, drag and drop)

1. Create the library instance in `onBuilt` (class, Options) or `initializeComponent` (plain tag): the
   element exists then, even before it is in `document`. Use `onMounted` only when the library needs the
   element in the document (it measures, reads layout or wants a live DOM); `onBuilt` may run while the
   element is still in a detached fragment, which misleads those libraries. Many `onMounted` hooks at
   once are run in slices, so their construction does not block the page.
2. Register its `destroy()` with `this.motif.setDisposable(...)` right where you create it; do the same
   for timers, `ResizeObserver`, sockets.
3. Feed reactive data through `this.bindings.watch`: take a plain snapshot in the tracked part and hand
   it to the library inside `untracked(...)`, so the library's own reads do not become dependencies:
   ```ts
   this.bindings.watch(() => {
     const snapshot = store.points.map((p) => ({ ...p }));   // tracked, deep reads included
     untracked(() => this.lib?.setData(snapshot));           // library reads are not tracked
   });
   ```
4. An exception thrown from `onBuilt` (or any other lifecycle hook) is reported as `MJX122` (`console.error` + `errorHandler`
   listeners, in production too) and the component stays in place without the library; wrap construction
   in `try/catch` when the wrapper must show a fallback or keep running without it.

If the library moves DOM nodes owned by a list binding (drag and drop), undo its DOM change, apply the
same change to the model and let the list binding redraw. Do not use `options` as a data prop name
(it is read as component options); use `chartOptions`, `sortableOptions`… A wrapper shipped as its own
package can declare its own element without JSX (`static elementTag = 'canvas'`); with no `view()`, JSX
children are appended to that element. Keep one copy of `@motifx/core` and of the wrapped library in the app
(`resolve: { dedupe: ['@motifx/core', 'chart.js'] }` in Vite): two copies split registries and reactivity
engines silently.
