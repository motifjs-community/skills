# Reporting a MotifJS bug

When something you see contradicts this skill, the docs at https://motifjs.com or the type
declarations, and a minimal reproduction shows the cause is MotifJS rather than the app, offer
to report it. Never file anything without the user's yes.

## 1. Make sure it is a MotifJS bug

Go through this list before you say "bug":

- **Reproduce it outside the app.** Write the smallest component that shows the behavior: one
  file, one component, no services or router unless the bug needs them. If the small version
  works, the cause is in the app; keep looking there.
- **Rule out JavaScript itself.** A plain value is read once: `note={editor.selected}` does not
  follow later changes, `note={() => editor.selected}` with a `Bind<T>` prop and `read()` does.
  A destructured reactive field is a copy. These are not bugs (`references/pitfalls.md`).
- **Rule out the documented quirks.** Check `references/pitfalls.md` and the reference file for
  the area. A behavior the skill already describes is not a bug, even when it surprises you.
- **Check the versions.** `npm ls @motifx/core @motifx/compiler`. The two packages are always
  released with the same version number; a mismatch prints `MJX121` in development mode.
- **Turn development mode on.** `app.useDevelopment(true)` before `app.run(...)` prints the `MJX`
  warnings. A correct warning (for example `MJX202` for a duplicate key) is the framework doing
  its job.
- **Find the layer.** `npx motif-explain file.tsx` prints what the compiler made of each JSX
  expression. Wrong output there is a compiler bug (`@motifx/compiler`); correct output with
  wrong behavior at runtime is a core bug (`@motifx/core`). For a view written without JSX it is
  always the core.

## 2. Ask the user

Tell the user in one or two sentences, in their language, what is wrong and how you confirmed
it, then ask whether they want it reported. Offer only the options that exist:

- give them the filled issue text to post themselves,
- file it yourself (only when you can, see step 5),
- leave it.

If they say no, do not ask again for the same problem in the session.

## 3. Write the reproduction

The reproduction in the issue is **made up**. Never paste the user's code, names, URLs, data,
tokens, file paths or business rules. Rebuild the behavior with neutral names (`Item`, `List`,
`state.items`, `/api/items`) in the smallest code that still fails, and run it once to confirm
it shows the bug. Give it in the style the bug appears in (class, function or Options; JSX or
not; TypeScript or JavaScript) and say which styles you tried.

## 4. Fill the template

Write the issue in English. Pick the repository by layer:

| The problem is in | Repository |
|---|---|
| `@motifx/core` or `@motifx/compiler` (runtime, compiler, `motif-lint`, `motif-explain`) | `motifjsdev/motifjs` |
| this skill told you something wrong | `motifjs-community/skills` |

```markdown
**Title:** [core|compiler] <what goes wrong> when <condition>

### Versions
- @motifx/core: <version>
- @motifx/compiler: <version>
- Bundler: <Vite x.y | Rollup x.y | none (UMD)>
- TypeScript: <version or "JavaScript">
- Runtime: <browser and version | Node version + jsdom>
- OS: <name>

### Component style
<class | function | Options>, <JSX | without JSX>, <TypeScript | JavaScript>.
Also tried: <other styles and whether they fail>

### What happens
<one paragraph: the wrong behavior, plainly>

### Reproduction
<every file needed, each in its own fenced block with its path>

### Steps
1. ...
2. ...

### Expected
<what should happen, and where the docs or types say so (link)>

### Actual
<what happens instead>

### Console
<MJX codes, warnings and stack traces, verbatim; "nothing" if silent>

### Compiled output (compiler bugs only)
<the relevant `npx motif-explain` lines>

### Workaround
<what avoids it, or "none found">
```

### A filled example

The bug below is invented. It only shows the level of detail an issue needs.

````markdown
**Title:** [core] `x-display` stops reacting after the component is detached and added back

### Versions
- @motifx/core: 1.2.0
- @motifx/compiler: 1.2.0
- Bundler: Vite 6.3.5
- TypeScript: 5.6.3
- Runtime: Chrome 141
- OS: Windows 11

### Component style
Class, JSX, TypeScript.
Also tried: function component with JSX (fails the same way), class without JSX using
`preconfig: (p) => p.bindings.display(...)` (fails the same way).

### What happens
A component whose element is shown with `x-display` keeps reacting while it stays in place. After
`controls.detach` and `controls.add` put it back in the same parent, changing the condition no
longer hides or shows the element. Text bindings inside it still update.

### Reproduction

`src/Details.tsx`
```tsx
import { Component, reactive } from "@motifx/core";

export class Details extends Component<HTMLDivElement> {
  state = reactive({ open: true, label: "Shipping details" });

  override view() {
    return <p x-display={() => this.state.open}>{() => this.state.label}</p>;
  }
}
```

`src/main.ts`
```ts
import { Application, Component } from "@motifx/core";
import { Details } from "./Details";

const app = Application.CreateBuilder().build();
app.useDevelopment(true);
const page = new Component("main");
app.run("#app", page);

const details = new Details();
page.controls.add(details);

page.controls.detach(details).then(() => {
  page.controls.add(details);
  details.state.open = false;
  details.state.label = "Billing details";
});
```

### Steps
1. Run the reproduction with `npm run dev`.
2. Look at the `<p>` inside `main` once the page has loaded.

### Expected
The `<p>` is hidden and a `<!--h-->` trace stands in its place; a detached component keeps its
bindings when it is added again (https://motifjs.com/en/docs/lifecycle).

### Actual
The `<p>` stays visible and reads "Billing details": the text binding works, the display binding
does not. Setting `open` back to `true` and to `false` again changes nothing.

### Console
Nothing. No `MJX` warning in development mode.

### Workaround
Create a new `Details` instead of adding the detached one back.
````

## 5. File it or hand it over

- **Look for duplicates first.** With the GitHub CLI:
  `gh issue list --repo motifjsdev/motifjs --state all --search "<two or three keywords>"`.
  Without it, give the user the search link
  `https://github.com/motifjsdev/motifjs/issues?q=<keywords>`. If an open issue matches, offer to
  add the new details to it instead.
- **Show the final title and body to the user** and wait for their yes, even when they already
  agreed to a report in step 2.
- **You can file it yourself only when a tool for it is there and authorized:** `gh auth status`
  succeeds, or a GitHub tool is connected in this session. Then:
  `gh issue create --repo motifjsdev/motifjs --title "<title>" --body-file issue.md`.
  Do not add labels or assignees. Give the user the link to the new issue.
- **Otherwise hand it over:** give the user the title and the body as a Markdown block and the
  link `https://github.com/motifjsdev/motifjs/issues/new` (or
  `https://github.com/motifjs-community/skills/issues/new` for the skill).
