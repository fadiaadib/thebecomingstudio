# CLAUDE.md

## What this is

The Becoming Studio portfolio site — a single-page gallery of works (apps, stories, campaigns)
that reveal on scroll and expand into a full-screen detail overlay on click.

It is **not** a normal static site and **not** a React/Vite project. It is a *Design Component*
(`x-dc`) document rendered by `support.js`, a bundled runtime that compiles an HTML template plus
a logic class into React at load time. There is no build step, no `package.json`, no node_modules,
and no git repository.

```
index.html    the entire app — template + <script data-dc-script> logic + content data
support.js    GENERATED dc-runtime bundle — do not edit (see below)
logo.png wordmark.png grain.png portrait.png   assets referenced by relative path
.thumbnail    WebP preview image written by the design tooling; not used at runtime
```

## Do not edit `support.js`

Its first line says it: generated from `dc-runtime/src/*.ts`, which is not in this directory.
Hand-edits are lost on the next regeneration. If runtime behaviour needs to change, change
`index.html` to work within the runtime, or raise it as a `dc-runtime` change instead.

Read it freely, though — it is the only specification of the template language available here.

## Running it

Serve the directory over HTTP and open `index.html`:

```
python3 -m http.server 8000   # then http://localhost:8000/
```

Boot needs the network: the runtime pulls React 18.3.1 and ReactDOM UMD builds from unpkg
(`cdn.ts`). `file://` mostly works — the runtime's `fetch(location.href)` re-read of the source
fails there but is caught and ignored — yet a real server is the reliable path.

There are no tests and nothing to lint. Verification is visual: load the page and check the
reveal sequence, hover dimming, and the detail overlay open/close.

## How a change flows

Almost every edit lands in one of three places inside `index.html`:

1. **Content** — the `DATA` array (line ~185). Three columns (`Apps`, `Stories`, `Campaigns`),
   each with `items`. An item needs `name`, `logline`, `desc`, `status`; everything else
   (`kicker`, `link`/`href`, `modes`, `knows`) is optional and gates a `<sc-if>` section in the
   overlay via the `has*` flags computed in `renderVals()`. Item numbering (`01`, `02`, …) is
   assigned by a running counter across all columns, so reordering renumbers.
2. **Layout / styling** — the markup inside `<x-dc>`. All styling is inline `style="…"`;
   keyframes and global rules live in the `<helmet><style>` block.
3. **Behaviour** — the `class Component extends DCLogic` block.

## The template language (`x-dc`)

The template is everything between `<x-dc>` and `</x-dc>`. It compiles to React, so it follows
React's rules, not the DOM's — but the syntax is HTML-ish and the differences bite.

**`{{ … }}` is not JavaScript.** `resolve()` in `support.js` handles only: dotted paths
(`item.name`), number/string/boolean/`null` literals, a leading `!`, `==`/`!=`/`===`/`!==`, and
parentheses. No arithmetic, no function calls, no ternaries, no `&&`. **Every computed value must
be computed in `renderVals()` and returned as a plain field.** That is why the code returns
things like `progressHeight`, `logoTransform`, and per-item `o`/`t`/`line` rather than expressing
them in the markup. An unresolved hole renders empty and logs a `[dc-runtime]` console warning.

**Control flow:**

- `<sc-for list="{{ columns }}" as="col">` — `as` names the loop variable; `$index` is also bound.
- `<sc-if value="{{ active }}">` — truthy test only. There is no `else` in use here.
- `hint-placeholder-count` / `hint-placeholder-val` are **streaming-preview hints for the design
  editor**, not runtime defaults. They only apply while the document is being streamed in. Keep
  them roughly matching the real shape; they have no effect on the served page.

**Attributes:**

- `style="a:b; c:d"` is parsed into a React style object, so `{{ }}` interpolation inside a style
  string works.
- `style-hover="opacity:0.55"` generates a real CSS rule in a runtime-managed sheet — this is how
  you get `:hover` on an inline-styled element. Any `style-<pseudo>` works the same way.
- Event handlers take a **function value**, not a code string: `onClick="{{ item.open }}"` where
  `renderVals()` supplied `open: (ev) => this.open(meta, ev)`. Handler names are matched
  case-insensitively against a React event map, so `onMouseEnter` and `onmouseenter` both land on
  `onMouseEnter`.
- `class` → `className` and `for` → `htmlFor` are translated for you.

**`<helmet>`** hoists its children into `<head>` — font links and the global `<style>` block.
Content here is outside the React tree; runtime values do not interpolate into it.

## The logic class

`<script type="text/x-dc" data-dc-script>` must define `class Component extends DCLogic`; any
other shape is a boot error surfaced as a red overlay on the page. `DCLogic` is a thin React-like
base: `this.props`, `this.state`, `setState`, `forceUpdate`, `componentDidMount`,
`componentDidUpdate`, `componentWillUnmount`.

`renderVals()` returns the flat object the template renders against, **merged over props** — so a
key returned here shadows a prop of the same name. It runs on every render; keep it cheap and
side-effect free.

Manual DOM listeners (`scroll`, `resize`, `keydown`) and the `IntersectionObserver` are registered
in `componentDidMount`. Every `setTimeout` is pushed onto `this._t` and cleared in
`componentWillUnmount` — follow that pattern for any new timer, since the component can unmount
and remount during editing.

## Editor-exposed props (`data-props`)

The `data-props` attribute on the script tag is HTML-escaped JSON declaring which props the design
editor shows as controls: `tagline` (text), `bodyType` (enum over the four loaded font stacks),
`revealStaggerMs` and `parallaxFactor` (range), `dimUnhovered` and `alwaysShowLoglines` (boolean),
each with `default`, `tsType`, and a `section` for grouping.

These arrive as `this.props.*`. Two consequences:

- **Always read them defensively** — `this.props.revealStaggerMs ?? 380` — because a prop can be
  absent when the document is rendered outside the editor.
- Adding a control means editing that escaped JSON blob *and* consuming the prop in
  `renderVals()`. Keep the `default` in the JSON and the fallback in the code in agreement;
  they are two separate sources of the same number.

## Design conventions

Worth preserving when adding UI, because the page reads as one deliberate system:

- Palette is exactly two colours: `#F8F6F2` ground, `#1A1816` ink. Depth comes from `opacity`,
  never from new hues.
- The easing curve is `cubic-bezier(.16,.8,.2,1)` throughout. Reuse it.
- Type is uppercase, small (9–12px), with wide `letter-spacing` (0.26em–0.4em) plus a matching
  `padding-left` to compensate for the trailing letter's space.
- Structural rules are 1px lines animated via `transform: scaleX()` from a `transform-origin`,
  not width or opacity.
- Sizes are `clamp()`/`min()` against viewport units rather than breakpoints — there is not a
  single media query in the page.
- The grain overlay is a fixed, `pointer-events:none`, `mix-blend-mode:multiply` layer at
  `z-index:80`; it sits above content and must stay non-interactive.
