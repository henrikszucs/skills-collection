---
name: beercss
description: Beer CSS 5.x (Material Design 3) class vocabulary, markup patterns and the ui() JS API. Use when writing or changing HTML styled with beercss - picking a class name, building a dialog, nav, field, menu, list, grid or layout, or handling theming, dark mode and responsive breakpoints.
---

# beercss

Material Design 3 in plain semantic HTML: no build step, no component classes
like `card-header`. Written against **beercss 5.0.3** (current release, Nov 2026).

## Setup

```html
<link href="https://cdn.jsdelivr.net/npm/beercss@5.0.3/dist/cdn/beer.min.css" rel="stylesheet">
<script type="module" src="https://cdn.jsdelivr.net/npm/beercss@5.0.3/dist/cdn/beer.min.js"></script>
<!-- only needed for ui("theme", ...) -->
<script type="module" src="https://cdn.jsdelivr.net/npm/material-dynamic-colors@1.1.4/dist/cdn/material-dynamic-colors.min.js"></script>
```

npm: `npm i beercss material-dynamic-colors`, then `import "beercss"; import "material-dynamic-colors";`.
Variants: `beercss/scoped` (styles only inside `.beer`), `beercss/custom-element`
(inside `<beer-css>`). The JS is optional for static pages; you need it for
`data-ui`, dialogs, snackbars, ripples, field labels without a placeholder,
textarea auto-height and theming.

## Rules that prevent silent failures

A wrong class does nothing: no error, no warning. Most mistakes come from these:

1. **Structure matters as much as class names.** beercss selects on nesting
   (`.field > input + label`, `menu > li > menu`, `.checkbox > input + span`).
   Copy the patterns in [ELEMENTS.md](ELEMENTS.md).
2. **Modifier first:** `top-margin`, `small-round`, `large-text`, `no-padding`.
   `margin-top` or `round-small` is ignored.
3. **Scales differ per helper.** For example, there is no `medium-padding`,
   `medium-round` or `extra-margin`. Check [HELPERS.md](HELPERS.md).
4. **One element per tag, then helpers:** `<button class="circle transparent">`
   is fine; `<div class="chip badge">` is not.
5. **`max` fills free space** in any flex parent (`nav`, `.row`, list row). An
   empty `<div class="max"></div>` is the spacer.

When unsure, check the shipped CSS (from npm, or curl the CDN file):

```bash
grep -oE "\.your-class([^-a-zA-Z0-9_]|$)" node_modules/beercss/dist/cdn/beer.min.css | head -1
```

Bare `round`, `blur` and the `*-round`, `*-padding`, `*-margin`, `*-width`,
`*divider*` families also match on substrings of `class`, so a grep miss there
isn't proof the class is missing.

## Page skeleton

```html
<body class="dark">                 <!-- light | dark; omit and JS picks from the OS -->
  <nav class="left max l">...</nav>  <!-- wide rail on large screens -->
  <nav class="left m">...</nav>      <!-- narrow rail on medium -->
  <nav class="bottom s">...</nav>    <!-- bottom bar on phones -->
  <main class="responsive">...</main>
</body>
```

Direct `nav.left/right/top/bottom`, `header`, `footer`, `main` and `aside`
children of the parent of `main` lay themselves out. Use exactly one `<main>`.
`s` / `m` / `l` alone are visibility classes (≤600px, 601–992px, ≥993px).

## The `ui()` API

`beer.min.js` defines `globalThis.ui`, sets `light`/`dark` on `body` if neither
is present, and watches the DOM so markup added later is wired automatically.

| call | does |
| --- | --- |
| `<button data-ui="#id">` | declarative toggle of `#id` on click (same as `ui("#id")`) |
| `ui("#id")` | `dialog` opens/closes (with overlay), `menu` toggles, `.snackbar` shows, `.page` activates; anything else toggles `.active` |
| `ui("#snackbar", ms)` | show for `ms` (default 6000; `-1` = until clicked) |
| `ui("mode")` / `ui("mode", "light"\|"dark"\|"auto")` | read / set the mode |
| `ui("theme", "#6750a4" \| imageUrl)` | **async**: generates an M3 palette, writes CSS vars to `body[style]`. Needs material-dynamic-colors |
| `ui("theme", {light, dark})` | apply a palette you already have (the object the call above resolves to) |
| `ui("theme")` | returns the current `{light, dark}` palette strings |
| `ui()` | re-wire fields, sliders, ripples and progress now |
| `ui("guid")` | random GUID |

Load `beer.min.js` as a module; it runs deferred. Call `ui` after
`DOMContentLoaded` or from a later module, not from an inline classic script in
`<head>`.

Without the JS you drive state yourself by adding/removing `active`
(`dialog.active`, `.overlay.active`, `.snackbar.active`, `menu.active`,
`.page.active`), or use native `<dialog open>` / `showModal()`.

## Theming without JS

Override the M3 custom properties (`--primary`, `--on-primary`,
`--primary-container`, `--surface`, `--surface-container`, `--outline`...) under
`body.light` and `body.dark`. Global knobs: `--font`, `--font-icon` (`"Material
Symbols Outlined"` by default; Rounded and Sharp are bundled), `--size` (root
scale, 1rem), `--speed1..4`. Note that `ui("theme", ...)` writes inline styles
on `body`, which beat your class rules.

## References

- [ELEMENTS.md](ELEMENTS.md): markup for every component. Read it before
  writing one.
- [HELPERS.md](HELPERS.md): the full helper vocabulary and its scales. Read it
  when you need a class name.
- Upstream docs: https://github.com/beercss/beercss/tree/main/docs ·
  https://www.beercss.com
