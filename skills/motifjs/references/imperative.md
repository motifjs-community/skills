# Writing views without JSX (the imperative style)

Every JSX element compiles to one call, `motifComponent(tag, initializeComponent)`, and the tree is built with
`sender.controls.add(...)`. A project written in plain `.ts` / `.js` writes those calls by hand: the
compiler's output, typed in, the way a desktop UI toolkit is used. The runtime, the bindings, the
lifecycle and the rules of `SKILL.md` are the same; only the spelling changes. Where JSX is available
it stays the default (SKILL.md, Thinking in Motif, point 7); this file is for projects that have no
JSX, for libraries that must not depend on the compiler, for generated code, and for reading compiled
output.

## The call shapes

| JSX | Hand-written |
|---|---|
| `<div class="app">…</div>` | `motifComponent('div', (c) => { c.class.add('app'); c.controls.add(…); })` |
| `Starter` (literal text) | `motifComponent('text', (n) => n.setText('Starter'))` — a text node; most projects wrap it in a `text(value)` helper |
| `{() => note.title \|\| 'Untitled'}` | `c.bindings.text(() => note.title \|\| 'Untitled')` (the compiler writes `bindings.method(fn)`, which also accepts a component result; `{state.field}` becomes `bindings.add('textContent', state, 'field')`) |
| `disabled={() => s.busy} aria-pressed={() => note.pinned} type="button"` | `c.attr.add({ type: 'button', disabled: () => s.busy, 'aria-pressed': () => note.pinned })` — a function is a live getter, a literal is written once; `false`/`null` remove the attribute |
| `class={() => open() ? 'row active' : 'row'}` | `c.class.add(() => open() ? 'row active' : 'row')`; a string or an array of both forms also works |
| `onclick={() => …}`, `onsubmit:prevent={…}` | `c.motif.on('click', () => …)`, `form.motif.on('submit:prevent', fn)`; modifiers chain freely here. The typed signature is `(sender, e)`: write `(_c, e: Event) => …` when the event is needed |
| `<form action={save}>`, `<button formAction={remove}>` | `form.attr.add({ action: () => save })`, `button.attr.add({ formaction: () => remove })` — the getter's value is the submit handler (called with the `FormData`); a string value is the URL |
| `<input x-model={() => s.q} />` | `field.bindings.model(() => s.q, (value) => (s.q = value))` |
| `<p x-display={() => cond}>` | `motifComponent('p', { preconfig: (c) => c.bindings.display(() => cond), initializeComponent: (c) => { … } })` — see `preconfig` below; `x-wait` is `bindings.wait` |
| `{cond ? <A/> : <B/>}` | `c.bindings.ternary(() => cond, (frame) => frame.navigate(a()), (frame) => frame.navigate(b()))` |
| `{cond && <X/>}` | `c.bindings.when(() => cond, () => x())` |
| `{rows.map((item) => <Row key={item.id} item={item} />)}` | a holder with no element: `motifComponent({ initializeComponent: (h) => h.bindings.list(() => rows, (item) => motifComponent(Row, { indexkey: () => item.id, item })) })` |
| `<input ref={(c) => (this.search = c)} />` | `motifComponent('input', { ref: (c) => (this.search = c), initializeComponent: (c) => { … } })`; the node is `this.search.element` |
| `<RouterLink to="/x" activeClass="active">All notes <span class="badge">…</span></RouterLink>` | `motifComponent(RouterLink, { to: '/x', activeClass: 'active', childs: [text('All notes '), badge] })` |
| `<RouterView class="outlet" />` | `motifComponent(RouterView, { class: 'outlet' })` |
| `<a rel="router" href="/notes">` | `motifComponent('a', (a) => a.attr.add({ rel: 'router', href: '/notes' }))` |
| `<NoteRow note={note} folder={folder} />` | `motifComponent(NoteRow, { note, folder })`: a class is constructed with the props, a function part is called with them |
| `<svg viewBox="0 0 236 236"><path d="…" /></svg>` | `motifComponent('svg', { options: { isSvg: true }, initializeComponent: (svg) => { svg.attr.add({ viewBox: '0 0 236 236' }); svg.controls.add(motifComponent('path', { options: { isSvg: true }, initializeComponent: (p) => p.attr.add({ d: '…' }) })); } })` — every SVG child carries `isSvg` too |
| `onMounted={(c) => …}` on a tag | `onMounted` (and the other hooks, `(sender, e)`) in the object form, or `c.motif.on('x:mounted', …)` inside the setup |

### `motifComponent(element, props?)`

- `motifComponent('div', fn)`: `fn` is the component's `initializeComponent`. It runs once, at build,
  right before the children are built, and never again; `this` is not bound, use the parameter.
- `motifComponent('div', { … })`: the object form, for the framework keys `initializeComponent`,
  `preconfig`, `ref`, `childs`, `options`, `transition`, `key` / `indexkey` and the lifecycle hooks
  (`onConfig`, `onBuilt`, `onMounted`, …). Attributes of a plain tag are written with `attr.add` inside
  the setup, as the compiler does, not as keys of this object.
- `motifComponent(Class, props)` constructs the class, and the object-form keys (`preconfig`,
  `initializeComponent`, `onMounted`, `childs`, `ref`) apply to it; `motifComponent(fnPart, props)` calls the
  function and applies only `ref`, the fallthrough attributes (`class`, `id`, `aria-*`…), `options` and
  `transition` to the root it returns: the hook keys do not reach that root, write them inside the function;
  `motifComponent(instance)` returns the instance as is; `motifComponent({ initializeComponent })` (an
  object with no tag) is a component **without an element**, a fragment: the holder the compiler uses for
  lists. `new Component('div', { initializeComponent })` is the same as the string form and is what the
  test-suite writes.
- `motifFragment({ initializeComponent: (f) => f.controls.add(a, b) })` is a fragment for a branch or a
  slot with several roots (comment markers in the DOM; no attributes, no events on it).
- A third argument (`1`) and the keys `__childExpr`, `__isSvgElement` appear in compiled
  output; they are compiler bookkeeping. Hand-written code leaves them out and never calls
  `motifCompiled` / `_mv`. The `catch (__e) { _mr(__e); … }` around a compiled `{this.part()}`,
  `{a ?? b}` or `{cond && …}` is bookkeeping too (`motifReport` reports the caught error as `MJX208`);
  a hand-written getter needs no `try`/`catch`: the runtime reports its error as `MJX208`.

### `preconfig`

`preconfig: (c) => …` runs when the component is configured, before `onConfig` and before `build()`. A
`bindings.display` / `bindings.wait` registered there keeps an element that starts hidden **unbuilt**
(its subtree, `onBuilt` and `onMounted` wait, like `x-wait` in JSX). The same binding registered in
`initializeComponent` builds the element first and hides it afterwards.

### Text

`'text'` is a real tag: `motifComponent('text', (n) => n.setText('…'))` creates a text node, and text
next to an element is one more child in the same `controls.add` call. The runtime also turns strings and
numbers passed to `controls.add` into text nodes, but the TypeScript signature of `controls.add` takes
components, so TypeScript projects keep a helper:

```ts
import { ComponentBase, motifComponent } from '@motifx/core';
export function text(value: string): ComponentBase {
  return motifComponent('text', (node: ComponentBase) => node.setText(value));
}
```

## Class pages and layouts in a `.ts` file

The Motif Vite plugin also processes `.ts`, `.mts` and `.cts` files: `class Page extends Component<HTMLDivElement>`
gets `static elementTag = 'div'` there as in a `.tsx` file, so the page has its own element. A class that
declares no element type is a **fragment**: `this.class` and `this.attr` have no element to
act on (`this.motif.on` listens on the content elements), and a CSS grid sees the children, not the page. Where the plugin does not process the file (a library
compiled with plain `tsc`), write `static elementTag = 'div'` or call `super('div')`. Two correct shapes:

```ts
// 1. No element declared (the starter's choice): the component is a fragment and view() returns its
//    content. The main below is the first child, not the component's element: this.element is a comment
//    marker; this.class / this.attr and the tag's class / id reach nothing (silently); this.motif.on and
//    the tag's onclick listen on the content elements; x-display and motif.hide() hide the children.
//    Configure main itself.
export default class SettingsPage extends Component {
  settings = this.getService(SettingsService)!;

  override view() {
    return motifComponent('main', (main: ComponentBase) => {
      main.class.add('detail');
      const heading = motifComponent('h1', (h1: ComponentBase) => h1.controls.add(text('Settings')));
      main.controls.add(heading, this.appearance());
    });
  }

  private appearance() { /* returns one component */ }
}

// 2. A declared element; view() then returns the children.
export class Panel extends Component<HTMLDivElement> {
  override onConfigured() { this.class.add('panel'); }
  override view() {
    return motifFragment({ initializeComponent: (f: ComponentBase) => f.controls.add(heading(), body()) });
  }
}
```

The reading order stays the JSX one: children as `const`s, one `controls.add` at the end, and private
methods for the parts (`toolbar(note)`, `editor(note)`), each returning one component. A function part
is a function that returns one component and reads services with `FromService(T)`:

```ts
export function NoteRow(props: { note: Note; folder: Folder }) {
  const nav = useNavigation();
  const { note, folder } = props;
  const open = () => nav.params.id === String(note.id);

  return motifComponent('a', (row: ComponentBase) => {
    row.attr.add({ rel: 'router', href: `/${folder}/${note.id}`, 'aria-current': () => (open() ? 'page' : null) });
    row.class.add(() => `row${open() ? ' active' : ''}`);
    const pin = motifComponent('i', {
      preconfig: (i: ComponentBase) => i.bindings.display(() => note.pinned),
      initializeComponent: (i: ComponentBase) => i.class.add('pin'),
    });
    const title = motifComponent('span', (span: ComponentBase) => span.bindings.text(() => note.title.trim() || 'Untitled'));
    row.controls.add(pin, title);
  });
}
```

A branch of a ternary is a fragment when it has several roots, a component when it has one, and a text
node when it is only text:

```ts
bar.bindings.ternary(
  () => this.state.confirming,
  (frame) => frame.navigate(motifFragment({ initializeComponent: (f) => f.controls.add(question(), confirmButton(), cancelButton()) })),
  (frame) => frame.navigate(this.actions(note)),
);
button.bindings.ternary(() => note.pinned, (f) => f.navigate(text('Pinned')), (f) => f.navigate(text('Pin')));
```

What still works in `.ts`: `@Injectable` and the other decorators (the Vite plugin lowers them in
`.ts` / `.js` files too), DI, the router, `useNavigation()`, transitions, `Virtualization`, and the
`MJX015` check of `motif-lint` (an override that does not reach `super`).

## Porting from JSX or from the docs

`npx motif-explain file.tsx` prints, per expression, the lowered call to write. For a whole file,
`new Compiler().motifCompile(code, 'file.tsx')` from `@motifx/compiler` returns the compiled code; read
it with this key: `_mc` = `motifComponent`, `_mf` = `motifFragment`, `_mfc` = `FNComponent`,
`_mv(1)` = `motifCompiled` (drop it), `_mr(__e)` = `motifReport` (drop it with its `try`/`catch`), `preconfig` = `x-wait` / `x-display`, `indexkey` = `key`,
`__isSvgElement: true` = `options: { isSvg: true }`, `ref: (s) => { this.x = s; if (this.onRefCreated) … }`
= `ref: (s) => (this.x = s)`. Drop `childs: []` and the trailing `, 1`.

## Pitfalls of hand-written views

1. **A fragment by accident.** A class that declares no element type is a fragment, and so is a
   `Component<HTMLDivElement>` class in a file the Motif Vite plugin does not process (a library compiled with
   plain `tsc`); write `static elementTag` or call `super('div')` there. Returning a wrapper from `view()`
   does not give the component an element: it stays a fragment and the wrapper is its first child.
2. **Errors inside `initializeComponent` are swallowed** (`MJX122`, SKILL.md rule 14): the element
   renders empty and `console.error` is the only sign. Typical cause: a `const` that shadows a helper used
   in its own setup (`const text = motifComponent('p', (p) => p.controls.add(text('…')))` calls the
   component, not the helper, when the setup runs at build). `motifComponent` returns `any`, so the type
   checker does not see it. In tests, assert the rendered text or spy on `console.error`.
3. **`display` / `wait` in the wrong hook.** Registered in `initializeComponent` the hidden element is
   built once; register them in `preconfig`.
4. **No compiler, no diagnostics.** `MJX001`–`MJX005`, `MJX016` and the switch/ternary getter rule (SKILL.md rule 5) concern
   JSX only; nothing warns about a `.map()` without `indexkey` or a prop that should be a getter. A plain
   expression in a branch (`motifComponent(Badge, { count: state.n })`) is a snapshot, exactly as in JSX;
   pass `count: () => state.n`.
5. **Types are yours.** Annotate setup parameters (`(c: ComponentBase)`), `ref` targets
   (`search!: Component<HTMLInputElement>`) and `motif.on` handlers (`(_c, e: KeyboardEvent)`).
6. **`controls.add` is immediate** and there is nothing to flush, but reactive *updates* still land on
   a microtask (`await Promise.resolve()` in tests), as in JSX.
