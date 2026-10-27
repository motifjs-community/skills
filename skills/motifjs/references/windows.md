# Windows: `app.windows`

A component can move into another same-origin window without being rebuilt: its object, state, bindings,
services and events stay in the page's JavaScript world; only the elements go. `app.windows` owns these
windows. Reach it as `this.context.windows` in a class or Options component, `useApplication().windows` in
a function component, `Application.main.windows` elsewhere.

## Opening

```ts
const win = await this.context.windows.open(editor, { width: 720, height: 560, title: 'Note' });   // popup
const top = await this.context.windows.open(editor, { kind: 'pip', width: 480, height: 360 });     // always-on-top
const empty = await this.context.windows.create({ width: 400, height: 300 }); empty.add(a, b);
const mine = this.context.windows.attach(iframe.contentWindow, { styles: 'once' });               // a window you already have
```

Call `open` / `create` inside a user gesture (a click handler): the browser call happens synchronously
and the promise resolves once the content is placed. `open(c, o)` is `create(o)` + `add(c)`.

| Option | Meaning |
|---|---|
| `kind` | `'popup'` (default) or `'pip'` (Document Picture-in-Picture: Chromium only, needs a gesture, one per page). |
| `fallback` | What to do when `pip` is missing or refused: `'popup'` (default) or `'none'` → rejects with `MJX419`. |
| `width`, `height`, `left`, `top` | Window size and position. |
| `name` | `window.open` name; unique per window when omitted. In Electron this is the `frameName` the main process sees. |
| `features` | Extra `window.open` feature string parts. |
| `title` | Window title; defaults to the page title. |
| `styles` | `'live'` (default), `'once'`, `'none'` or `(doc) => void`. |
| `orphans` | Content whose origin is gone when the window closes: `'dispose'` (default) or `'keep'`. |
| `closeWhenEmpty` | Close when the last component is disposed (default `true`). |
| `closeOnNavigate` | Close on every route change (default `false`). |
| `returnOnClose` | Put content back to its origin on close (default `true`); `false` sends components with an origin through `orphans` too. |
| `owner` | The window is registered in this component's dispose layer and closes with it. |
| `onClosing` | `(e: { window, components, reason }) => void`, called right before the window is torn down, content still inside. See Closing. |

## The handle (`AppWindow`)

`window`, `document`, `host` (the component wrapping the window body), `kind` (`'popup' | 'pip' | 'attached'`,
what really opened), `components`, `isOpen`, `add(...c)`, `release(c)` (back to its origin, window stays open),
`focus()`, `close()` → `{ returned, disposed, kept }`, `closed` (the same promise however the window closes),
`syncStyles()`. A toast or dialog opened by a component in the window belongs in that window:
`this.context.layerOf(this, dialog)` adds it to the window's `host` when the component's element is in a managed
window, to `app.layer` (the page layer, a plain `div` at the end of `body`) otherwise.

## Origin and return

The first time a component is taken, the manager remembers its page parent and the sibling after it. The
component gets `onDeactivated`, enters the window host through `controls.add` and gets `onActivated`; `onMounted`
does not fire again (it fires in the window's own task queue when it never fired before). Moving between
windows keeps the origin. `release` and closing put the component back before the remembered sibling (or
that sibling's own remembered sibling when it is in a window too), else at the end. `returnOnClose: false` turns
this off (`release` still returns). No origin left → `orphans`:
dispose (default) or keep detached (`closed.kept`). Only a component given through `open` or `add` has an origin:
what the app moves into an attached window on its own is unknown to the manager, nothing is returned for it on
close and `components` does not list it.

## Closing

User close, `win.close()` or the window's `pagehide` → content returns, `closed` resolves. A component disposed
by the app drops out of the window; an empty window closes unless `closeWhenEmpty: false`. `owner` disposed →
closes. `closeOnNavigate` → closes on navigation. `app.dispose()` and the page's own `pagehide` close every
window and dispose their content; `app.windows.closeAll()` does it on demand.

`onClosing` runs first, before anything is moved: at the window's `pagehide` on a user close, before the host is
disposed otherwise. The window already counts as closed (`isOpen` false, `add` throws `MJX420`) but its document is
still there. `e.components` is a copy of what is in the window; `e.reason` is `'user'` (closed by the user or
`pagehide`), `'code'` (`close()` / `closeAll()`), `'owner'`, `'empty'`, `'navigate'` or `'page'` (the page unloads or
`app.dispose()`). A component the hook gives another parent (`controls.add`) is neither returned nor disposed, its
origin is forgotten and it is missing from `closed`; `win.release` works there too. A throwing hook is reported as
`MJX423` and the close goes on.

## Styles and theme

On open every `<link rel="stylesheet">` and `<style>` of the page (anywhere in the document) is cloned into
the window's head; `adoptedStyleSheets` are rebuilt from their rule text into one `<style>`; `class`, `lang`,
`dir` and `data-*` of `<html>` and `<body>` are mirrored. With `styles: 'live'` (default) later additions,
changes and removals of stylesheets and root attributes follow (two `MutationObserver`s while a live window
is open). Rules added later to an adopted sheet are not observable: call `win.syncStyles()`.

## List and events

`app.windows.list` is reactive (read it in a binding), `app.windows.of(component)` gives the window or
`null`, `app.windows.supports('pip')` tells whether Picture-in-Picture exists. Application events
`window-opened` / `window-closed` carry `{ window }`. A `rel="router"` link inside a window navigates the
page's router; the window is never reloaded.

## Electron

The main process decides on `window.open`; allow the name you pass and give the window its options there:

```ts
win.webContents.setWindowOpenHandler(({ url, frameName }) => {
  if (frameName === 'motif-popup') return { action: 'allow', overrideBrowserWindowOptions: { width: 560, height: 420, alwaysOnTop: true, frame: false } };
  if (/^https?:\/\//.test(url)) void shell.openExternal(url);
  return { action: 'deny' };
});
```

Renderer: `app.windows.open(editor, { name: 'motif-popup' })`. The child shares the opener's renderer process
(also with `file://` pages), so everything above holds. Electron exposes `documentPictureInPicture` but refuses
the request; `fallback` lands on a popup, and `alwaysOnTop` gives the always-on-top window instead. A window
created in the main process with `new BrowserWindow` is another process: no component can move there, talk
over IPC or `MessagePort`.

## Errors

`MJX418` the browser did not open the window (blocker, no gesture) · `MJX419` PiP missing or refused with
`fallback: 'none'` · `MJX420` `add`/`focus` on a closed handle · `MJX421` `attach` got something that is not a
window, or a window of another origin · `MJX422` a disposed component was given to a window · `MJX423` `onClosing`
threw (reported, the close goes on).
