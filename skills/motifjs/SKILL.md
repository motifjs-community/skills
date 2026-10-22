---
name: motifjs
description: "Build, change or debug MotifJS apps (@motifx/core, @motifx/compiler), in JSX or in plain TS/JS with motifComponent. Use when code imports @motifx/core or the user mentions MotifJS. MotifJS is not React: read this before writing its code."
license: MIT
metadata:
  author: MotifJS
  version: "1.0.0"
  homepage: https://motifjs.com
---

# MotifJS

MotifJS is a UI framework for JavaScript and TypeScript (JSX/TSX) for long-lived, state-heavy browser apps.
There is **no virtual DOM, no diffing, no re-render loop and no SSR**. The `@motifx/compiler` compiler
turns **every JSX element into a live component object** (`_mc(tag, setup)`) and builds the tree in code
with `sender.controls.add(...)`; there are no HTML template strings. Each element has its own `controls`,
`class`, `attr`, `bindings`, `motif.on` and lifecycle, and every dynamic expression becomes a fine-grained
*binding* on that component that patches only the text node / attribute / list slot that depends on the
changed reactive field.

Packages: `@motifx/core` (runtime: components, reactivity, bindings, router, DI, transitions,
virtualization) and `@motifx/compiler` (JSX compiler exposed as a Vite plugin — also usable in a plain
Rollup build — plus the `motif-lint` and `motif-explain` CLIs).

## Thinking in Motif (read before designing a feature)

Code that uses every API below correctly can still be structured like a re-rendering framework.
The structure below is the default; `references/thinking-in-motif.md` explains each point and shows
one page written both ways.

1. **A component is a long-lived object** that owns real DOM nodes, in all three forms (class, function, Options API). `view()` hands it a template once; from then on reactivity and `controls` manage it. There is no re-render to design around.
2. **Model and view are separate files.** Domain services hold reactive state and operations and are registered in DI (`singleton`). A page whose own state outgrows a handful of fields, or that coordinates several services, gets a page model (`scoped`: one instance per navigation, shared by the page and its children). The view binds and calls; it carries no fetch, try/catch or business rule. Write the model first, then the view.
3. **Communication is chosen by scope.** Props are the normal way to hand data from parent to child: the item, the variant, the slot content, a live reactive object. App-wide concerns (session, API, settings, the page model) are resolved from DI by the component that needs them; they are not threaded down through props. Operations are called on the model; a callback prop reports a choice from a small reusable part. The component tree is a live object graph: `parent`, `controls`, `siblings`, `$(selector).fromComponent()` and `ref` reach any component in every direction, and its public fields and methods are used like any object's.
4. **Identity drives the data model.** One reactive object per entity for the life of the screen: fill it in place when data arrives, `push`/`splice` lists. Replacing an object or array is a deliberate "rebuild everything bound to it".
5. **Rebuild or fill in place is a design decision per page.** The same thing changing state (data arriving, a page number, a sign-in) is filled in place behind `x-display`/`x-wait`; a different thing, or a tree whose shape depends on the data or on route params, is rebuilt. Both are correct in their place.
6. **Boundaries follow state, reuse and rows,** not size. Split out a part when it has its own state and lifetime, when it is reused, or when it is a list row; never down to a `<div>`. Three or three thousand lines are both fine. Pages and model-bound views are classes, small parts are functions, the Options API carries object-based code.
7. **`controls.add` inserts content that has no state to bind to.** Everything bound to a model is written in JSX. In a project without JSX the whole view is the same calls written by hand, `motifComponent(tag, (c) => { c.class.add(…); c.bindings.text(…); c.controls.add(…); })`; `references/imperative.md` gives the shapes and the `.ts`-only pitfalls.
8. **Lifetime is yours.** Resources you open are registered with `this.motif.setDisposable`; work that finishes after an `await` goes through `this.using` / `this.doWork` or a scoped model, not a hand-written `isDisposed` check after every `await`.

## Habits to unlearn

| Habit | MotifJS |
|---|---|
| `useState`, hooks, re-render on change | `state = reactive({...})`; mutate fields (`this.state.count++`); the bound DOM node updates, nothing re-renders. |
| `render()` runs many times | `view()` runs **once**; a value computed outside a `{() => ...}` getter is frozen. |
| `className`, `onClick`, `htmlFor` | DOM names: `class`, `onclick`, `oninput`, `for`. |
| `{cond ? <A/> : <B/>}` and `items.map(...)` re-evaluated per render | Compiled to `bindings.ternary` / `bindings.list`: a branch is rebuilt when the value changes, rows follow the item object. Give a stable `key` anyway. |
| `useEffect` + cleanup | `this.bindings.watch(fn)` (auto-disposed) or `effect(fn)`, whose stop function you register. |
| A JSX element is a DOM node | Every element is a component instance; its node is `s.element`. Classes, attributes, events and children are calls on it (`s.class.add`, `s.attr.add`, `s.motif.on`, `s.controls.add`), and the same calls work without JSX. Cost and capability are counted per component. |
| `ref` gives a DOM node | `ref` / `x-ref` give the component instance, exactly once; the node is `s.element`. `ref` is never in `props`. |
| Context / Redux / Pinia | Built-in DI (`builder.services.addSingleton(...)`, `this.getService(T)`, `inject(T)`) with reactive service state. |
| `<Link>`, `useNavigate`, `<Outlet>` | `<a rel="router" href="/x">`, `<RouterLink to="/x" el="a">`, `this.context.navigate('/x')`, `<RouterView />` inside the layout. |
| Unmount is free | JSX-created children and `this.context.on(...)` subscriptions are disposed for you; timers, sockets and `effect` stops go through `this.motif.setDisposable(fn)`. |

## Minimal correct project

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import compiler from '@motifx/compiler';
export default defineConfig({
  plugins: [compiler()],
  resolve: { extensions: ['.tsx', '.ts', '.jsx', '.js'] },
});
```

```jsonc
// tsconfig.json (relevant keys)
{ "compilerOptions": {
    "jsx": "preserve", "target": "ES2021", "module": "esnext", "moduleResolution": "bundler",
    "useDefineForClassFields": true, "strict": true, "lib": ["ESNext", "DOM", "DOM.Iterable"] } }
```

```tsx
// src/index.tsx
import { Application, Component, reactive } from '@motifx/core';

class Counter extends Component<HTMLDivElement> {
  state = reactive({ count: 0 });
  onConfigured() { this.class.add('counter'); }
  view() {
    return (
      <>
        <p>Count: {this.state.count}</p>
        <button onclick={() => this.state.count++}>+1</button>
      </>
    );
  }
}

const app = Application.CreateBuilder().build();
app.run('#app', new Counter());
```

`app.run(host)` with no root component mounts a default `<RouterView>`; combine with
`app.useRouter({...})` for routed apps (see `references/routing.md`).

## Rules that prevent most bugs

1. **Reactive text must be a reactive field or a getter.** `{state.name}` and `{() => a + b}` update; `{localVar}` and `{someFn()}` are one-shot. Template literals containing reactive reads are compiled reactively.
2. **Attributes take a value or a getter function.** `disabled={() => state.busy}`, `class={() => ...}`, `style={() => ({...})}`. A plain object/string is static, except `class={{ name: cond }}`: each key is a class toggled by its condition; literal conditions apply once, any other value is wrapped in a getter and stays live (also inside an array, `class={["card", { active: state.on }]}`). On a plain tag literal values are kept (`tabindex={0}` → `"0"`); `false`/`null`/`undefined` remove the attribute and `true` writes `""`; on `aria-*`/`data-*` a boolean is written as text (`aria-expanded={false}` → `"false"`). An expression reading a variable (`a + b`, `!state.open`) is wrapped in a getter and stays live. Method-like props on a plain tag call the element method instead of setting an attribute (`focus`, `showModal={() => state.open}`, `scrollIntoView`, `setSelectionRange`… — see `references/jsx-and-reactivity.md`). `{...props}` on a DOM tag applies every key; on a component tag the common attributes (`class`, `style`, `id`, `tabindex`, `role`, `aria-*`, `data-*` — not `title`) fall through to the component's own element (a class's declared element, an Options object's `el`, the root a function component returns; a fragment component has none and drops them) and everything stays in `this.props`.
3. **Lists need object items and a stable `key`.** Prefer arrays of objects over arrays of primitives; rows are matched by the item object (primitives by value/position), so object identity drives reuse. `key` does not affect reuse; it identifies items and a duplicate raises `MJX202`.
4. **Show/hide with `x-wait`, not `{cond && <X/>}`.** `<Panel x-wait={() => !state.open} />` waits *while* the function returns `true` (`x-display` is the inverse). If it starts hidden the element is not built until the predicate turns `false` (the subtree, `onBuilding`, `onBuilt` and `onMounted` wait; `onConfig`/`onConfigured` still run), and later toggles only hide/show, so the instance and its state survive. `{cond && <X/>}` builds a new instance on every show and disposes it on every hide; pick it only when that dispose is the point. `&&` and ternary branches are rebuilt only when the condition value changes, so a component prop written as a plain expression inside a branch (`<Badge count={state.n}/>`) is a snapshot from when the branch was built; pass a getter (`count={() => state.n}`) and read it with `Bind<T>` / `read()` to keep it live.
5. **In a block-bodied arrow in a JSX child position, statements before a `switch` or before `return a ? b : c` are kept**: they run in the discriminant/condition getter and the branches see the locals (a branch reads them when it is built, so a change that keeps the condition value does not reach it; write live values inside the branch as getters); `case` bodies, fall-through and the statements after the `switch` run as written. A shape the compiler cannot lower (a non-literal `case` label, a `return` or JSX before the `switch`, a `break` that leaves it) runs as `bindings.method` and warns `MJX016` (no branch is kept). Only an arrow **with parameters** still hoists the condition out of the closure (`MJX001`, `ReferenceError`); use an expression-bodied arrow there. The compiler also warns about three more silent mistakes (`MJX002` camelCase DOM event name on a component tag, `MJX003` `.map()` item without `key`, `MJX004` short-circuit array predicate in a reactive getter) — see `references/pitfalls.md`. A type-aware check, `npx motif-lint`, adds `MJX005` (a ternary passed to a component prop whose declared type is not `Bind<T>`). When unsure what a JSX expression compiles to, run `npx motif-explain file.tsx`: it prints, per expression, the real lowered call, its reactivity class and dependency surface (`references/setup.md`).
6. **The component's own element:** a class component gets it from `class X extends Component<HTMLDivElement>` (the Motif Vite plugin injects `static elementTag = 'div'`, in `.ts` and `.tsx` files alike), `static elementTag` or `super('div')` (with no constructor the declared type is used; a tag passed to `super` wins over it); an Options object gets it from `el`. `view()` returns the content placed *inside* that element, never the element itself: a `div` component whose `view()` returns `<div>…</div>` renders `<div><div>…</div></div>`. Configure the component's own element in `onConfigured` (`this.class.add(...)`, `this.attr.add(...)`, `this.motif.on(...)`) and return only the content from `view()`. A class with no element declared, or an Options object with `el: undefined`/`null`, is a *fragment* (comment markers): `class`, `id` and the other attributes written on its tag reach nothing, silently, DOM event props on its tag listen on the content elements, and whatever `view()` returns are its children. A function component is different: the root it returns is the component's element.
7. **Props typing:** `class X extends Component<HTMLDivElement, Props>`; read via `this.props`; JSX children arrive as `this.childs` (array of components).
8. **Lifecycle:** fetch data in `onConfig` (props ready, not in DOM yet; in subclasses it runs at the start of `build()`, after class fields such as `state = reactive(...)` exist); touch the live DOM (focus, measure) in `onMounted` — `onBuilt` can fire while the element is still inside a detached fragment (ternary/list branches); clean external resources in `onDisposing` or via `this.motif.setDisposable`.
9. **Event handlers** may be `(e) => ...` or `(sender, e) => ...`; MotifJS picks by arity. Returning `{ cancel: true }` prevents default + stops propagation; any other return value (including `false`) is ignored. Modifiers: `onclick:once`, `onsubmit:prevent`, `:stop`, `:capture`, `:passive`, `:self`, `:trusted` — one per JSX attribute (JSX syntax allows a single `:`); chain them imperatively: `this.motif.on('click:once:prevent', fn)`.
10. **Two-way input binding:** `<input x-model={() => state.q} />` — compiled to a getter/setter pair that reads the owner object again on every write, so it survives the object being replaced or a list being reordered. Long forms: `value={() => state.q}` + `oninput={(e) => state.q = e.target.value}`, or `onConfig={(s) => s.bindings.model(state, 'q')}`.
11. **Never keep `effect()` return values unregistered** inside components; use `this.bindings.watch` instead.
12. **Async after dispose:** wrap with `this.using(promise, cb)` (callback skipped once disposed) or `await this.doWork(promise)` (resolves to an `Error` instance instead of the value once disposed).
13. **Add imperatively any time:** `this.controls.add(<div/>)` inserts into the DOM immediately; there is no flush to await.
14. **Errors do not propagate.** A throw (or rejected promise) in a lifecycle hook (methods, JSX props, `ref` callbacks and `motif.on('x:built' | 'x:mounted' | …)` listeners) is reported as `MotifError` `MJX122`, in an event handler (DOM, `motif.on`, `app.on`/`app.fire`, `onRouterChanged`) as `MJX123`, in an effect/binding function as `MJX208`; the component keeps working. Reports go to `console.error` (unless `app.useLogging(false)`) and to `errorHandler.addListener(fn)` listeners, in production too; `error.code` holds the code and `error.cause` the original error.

## Two more component styles (all compile to the same core)

```tsx
import { motifOptions } from "@motifx/core";
// Function component: closure state, JSX return, lifecycle via props on the root element
export function Greeting(props: { name: string }) {
  const s = reactive({ likes: 0 });
  return <div onBuilt={(c) => console.log('built', c.element)}>
    Hi {props.name} <button onclick={() => s.likes++}>({s.likes})</button>
  </div>;
}

// Options-API object: { el, data, view, ctor?(props) } — ctor runs once after creation, this = component
export const MessageBox = () => motifOptions({
  el: 'div',
  data: reactive({ msg: 'Hello' }),
  view() { return <span onclick={() => this.data.msg = 'Clicked'}>{this.data.msg}</span>; },
});
```

## Reference map (read on demand)

| Need | File |
|---|---|
| Design: component as object, model/view split, communication by scope, identity, rebuild vs fill, one page written both ways | `references/thinking-in-motif.md` |
| Build setup, file extensions, `@DEV-ONLY` markers, tsconfig | `references/setup.md` |
| Component anatomy: constructor forms, props/childs, controls, refs, context, elementTag | `references/components.md` |
| Views without JSX: `motifComponent` call shapes, text nodes, `preconfig`, fragments for branches and lists, class pages in `.ts`, reading compiled output | `references/imperative.md` |
| JSX expression forms, attributes, `x-*` props, reactive primitives, bindings API | `references/jsx-and-reactivity.md` |
| `x-wait`/`x-display`, `&&`, ternary, `.map`, `switch`, `Frame`, `Lazy`, `Transport` slots | `references/conditionals-lists-frames.md` |
| DOM/component/app events, forms, two-way binding | `references/events-and-forms.md` |
| Lifecycle order, show/hide, isWait, dispose, memory rules, wrapping third-party libraries | `references/lifecycle-and-dispose.md` |
| Routes, params, guards, `RouterView`, `RouterLink`, navigation | `references/routing.md` |
| DI: ServiceCollection, `@Injectable`, `inject`, `getService` | `references/di.md` |
| class/style helpers, CSS transitions, WAAPI | `references/styling-and-transitions.md` |
| `Virtualization` for big lists | `references/virtualization.md` |
| Phone-style page stack, swipe back, app lifecycle, Electron and `file://` apps | `references/mobile-and-electron.md` |
| Writing Jest/jsdom tests for MotifJS components | `references/testing.md` |
| Compact export list with signatures (incl. `Query`, error handling, `@motifx/core/devtools`) | `references/api-cheatsheet.md` |
| React-habit mistakes and known compiler/runtime quirks | `references/pitfalls.md` |

When unsure about a signature, the ground truth is `node_modules/@motifx/core/dist/index.d.ts`
(runtime types) and the sources shipped in `node_modules/@motifx/core/src/`. Do not invent
hooks (`useEffect`, `useRef`, `componentDidMount`) — they do not exist. (`onMounted` IS a MotifJS hook: fires once when the element reaches `document`.)

Project: https://motifjs.com · Source and issues: https://github.com/motifjsdev/motifjs
