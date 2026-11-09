# Building a component library

A component library (a kit of controls, a data grid, a scheduler) is used by apps it does not know, often with
thousands of its parts on one page. App code can lean on reactivity; library code must not. These rules come from
building a full kit and measuring it. Read `thinking-in-motif.md` and `components.md` first.

## The three ways in

A library component listens to exactly three things:

1. **Its props**, each a value or a function (`Bind<T>`).
2. **The developer's data**, taken as given: kept and handed back as the very objects, never wrapped in `reactive()`,
   copied or converted. A change made in place is read again on `reState()` or on an explicit `refresh()`.
3. **The developer's `reState()`.**

Everything else is the component's own state, and the component draws it by hand.

## Props: a value or a function, followed by one binding

- Every prop and every property setter takes a value or a function. A function is followed by **one** binding of the
  component, registered with `bindings.add(binding)` (the `IBaseBinding` shape, see `jsx-and-reactivity.md`). Being
  in the component's registry is what lets the developer's `reState()` reach it, so plain data works as well as
  reactive data.
- Hand the value on only when it differs from the last one read (`Object.is`). A `reState()` that finds the same
  value must leave what the user did since (a text typed, a day moved to).
- A property set from code stops following its function.
- Render a property with an applier list instead of an effect: apply now and on every change, untracked. No effect
  exists for a property given as a plain value.
- Data objects with function fields (menu items, toolbar items, summary items): read all their fields in **one**
  binding per source, keep the values read and draw from them. Never call the developer's function again while
  drawing: a function that returns a new node makes a new node on every draw.
- A value the user changes too (a text box, a list, a check box) is given to the control only when its source changed,
  not on every draw, or the user's typing is overwritten:

```ts
const handed = new WeakMap<object, { value: unknown }>();
export function handOver<V>(control: { value: V }, value: V) {
  const last = handed.get(control);
  if (last && Object.is(last.value, value)) return;
  handed.set(control, { value });
  control.value = value;
}
```

## Inside, nothing is followed

- **A change is told where it is made.** Each part of the component's state calls `tell(name)`. The names are
  collected and heard once a turn; each view draws again the parts that read what was told. Nothing watches the
  state.
- **What the component does about a change runs before the views.** Loading again, laying out and closing an editor
  run first, and what they tell joins the same turn, so the views draw once. Such a reaction runs when the values it
  reads are no longer the ones it read last. Compare stable values: objects and `Map`s built anew on every read never
  compare equal, so key them as JSON. Put counters in the key too, such as a "read again" version.
- **Tell only on a real change.** A notice told on every draw is a loop: draw, tell, draw. It froze a browser tab.
- **Hand-drawn parts are components made in code.** No JSX there: the compiler makes every expression that reads a
  variable a binding, an attribute too. Also no `bindings.watch` and no free `effect()`. On a data grid page this
  took bindings from 1,306 to 87 and effects run while opening from 2,478 to 346; a sort runs none (was 140).
- **Reactive state may stay for the app.** An API documented as "reactive" (a total, a selected count) can keep its
  store for the app's own bindings; the component never reads it tracked.
- **A language or locale change is an event**, not a signal. Each control subscribes once, draws its texts and
  formats now and again on each change, and the subscription goes with the control:

```ts
export function onLanguage(component: ComponentBase, draw: () => void) {
  untracked(draw);
  const changed = uiLocale.onChanged(() => !component.isDisposed && untracked(draw));
  component.motif.setDisposable(() => changed.dispose());
}
```

- **Static content is built once.** `{cond && <X/>}` and `.map` rows with a `key` cost bindings per item. Content that
  does not change can be built once in `onConfig` or `initializeComponent` with `controls.add`. Done on one docs page,
  this took the page's effects from 16K to 10K.

## The component's tree

- `view()` returns one `ComponentBase`. Several parts go into `motifFragment()`, which is what `<>…</>` compiles to.
- Parts that must be direct children of the element (header cells, rows that are moved) are added in
  `initializeComponent` (an override), not returned from `view()`.
- Keyed parts are reconciled by hand. Never move a node that is already in its place: moving the focused element
  takes its focus.
- `controls.remove(part)` takes the part out at once. `part.dispose()` removes its element a turn later, so a draw
  right after it still finds the element.
- A part shown and hidden in its place, like `x-display`, is swapped for a comment placeholder and put back there. A
  placeholder needs a parent, so draw top-level parts the first time in `onBuilt`.

## Writes are change-checked

- Writing an attribute, a style or a text again with the same value still makes the browser work the styles out
  again. Hand-drawn code that rewrote unchanged attributes on every draw had five times the style recalculation of
  the reactive code it replaced. Write a value only when it changed.
- Replacing a constructable stylesheet (`replaceSync`) with the same text invalidates the styles of the whole
  document. A grid that rewrote its per-column sheet on every step of a column resize spent 60–110 ms per step; writing
  the sheet only when its text changed removed the long tasks.

## A temporary attribute on the developer's component

- A library may have to give an attribute for a while to a component the developer wrote and placed inside it:
  `disabled`, `tabindex`, `aria-*`. `attr.add` does not fit: on the key it writes it removes the developer's binding
  and cannot give the old value back.
- `attr.hold` keeps the value over the component's own and returns an `IDisposable`:

```ts
const held = part.attr.hold({ disabled: true, tabindex: -1, 'aria-disabled': 'true' });
this.motif.register(held);            // released with this component; or call held.dispose()
```

- `dispose()` returns each key to the component's own value at that moment. While held, what the component writes
  to those keys stays aside and does not reach the element: a new value from a live getter, `attr.add`,
  `attr.remove`, and a `bindings.add` binding that writes the element's property of the same name (`disabled`,
  `tabIndex`, `ariaLabel`). `reState()`, `setState()` and a transport do not break the hold.
- What comes back is what the element had when the hold began, also an attribute `attr` never wrote and one that
  was absent. If the component wrote to the key during the hold, its last write comes back.
- Holds on one key stack: the element shows the newest; whichever is disposed first, the own value returns when the
  last one is disposed.
- It writes by the rule of `attr.add`: the key is written on every element. Giving `disabled` only to elements that
  have it is the caller's job (`'disabled' in part.element`). A held value is written once; a function is not
  followed.

## The component an event came from

- `part.motif.isEventFrom(e)` is `true` when the DOM event started on the component's element or inside it. A
  component without an element answers for its content elements. The answer holds after the event has finished
  (when `composedPath()` is already empty) and across an open shadow root; a closed shadow root hides its inside
  from a listener outside it.

```ts
this.motif.on('click:capture', (_sender, e) => {
  if (part.motif.isEventFrom(e)) return;
  e.preventDefault();
  e.stopImmediatePropagation();
});
```

## Lifetime

- Disposing a component clears its own fields to `undefined`. Callbacks that come later (timers, promises,
  observers, a deferred draw) check `isDisposed` first.
- `onDisposing` runs before the element leaves the DOM, and removing a focused element fires `focusout`. A
  `focusout` handler can therefore run on a component that is being disposed.
- When focus has to move off a control that is hidden or disabled, focus the next one with
  `focus({ preventScroll: true })`, or the page scrolls to it.
- `ref` callbacks run inside the base constructor, before the subclass's field initializers. State that a `ref` may
  touch lives on a lazily created, symbol-keyed object, not in class fields.

## Long work

- `Promise.then` and `queueMicrotask` do not let the browser paint: a chain of microtasks is still one long task.
  Split long work with macrotasks (`setTimeout`, `MessageChannel`). Measured: one 300 ms long task as microtasks, no
  long task and 18 painted frames as `setTimeout` pieces.

## Checking the work

- Count bindings per component type by walking the component tree (`devtools` `inspect(path).bindings`), not only
  live effects: a binding that follows a prop has no effect owner and is missing from an effect count.
- Rewrite a view under a pixel comparison of old and new on every page of it. Keep its tests and e2e checks green at
  every step, and prove the effect count went down.
- An e2e check that reads the page right after `load` waits for the content first: pages draw their sections after
  `load`.
