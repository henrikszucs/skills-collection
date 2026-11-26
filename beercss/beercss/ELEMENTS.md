# Elements (beercss 5.0.3)

Component size scale where supported: `tiny small medium large extra`.
Icons are `<i>name</i>` (Material Symbols); `i.fill` gives the filled glyph.

## Button, chip, FAB

```html
<button>Text</button>
<button class="border">Outlined</button>
<button class="transparent circle"><i>close</i></button>
<button class="large round"><i>add</i><span>Create</span></button>
<button class="extend"><i>edit</i><span>Compose</span></button>  <!-- FAB that grows on hover -->
<a class="button fill">Link styled as button</a>
<div class="chip"><i>done</i><span>Filter</span></div>
<button class="circle"><progress class="circle small"></progress></button>
```

Fill: default (primary), `fill` (tonal), `border`, `transparent`, or any colour
class. Shape: `round`, `circle`, `square`, `small-round`, `left-round`,
`right-round`. Also `vertical` (icon over label), `responsive` (full width),
`disabled` attribute.

## Button group

```html
<nav class="group connected">          <!-- connected | split -->
  <button class="active">Day</button>
  <button>Week</button>
</nav>
```

## Field

```html
<div class="field label border">
  <input type="text">
  <label>Name</label>
  <output>Helper text</output>
  <output class="invalid">Shown only when the field has .invalid</output>
</div>
```

- **`input`/`select`/`textarea` first, then `label`.** The label floats via
  `:focus + label` and `[placeholder]:not(:placeholder-shown) + label`. The JS
  sets `placeholder=" "` for you; without the JS, add it yourself.
- Helper and error text are **`<output>`**. On `.field.invalid` the
  `output.invalid` shows and the plain one hides.
- Wrapper modifiers: `label`, `border`, `fill`, `round`, `small`/`large`/`extra`,
  `prefix`/`suffix` (with an `<i>`, `<img>` or `<progress class="circle">` before/after
  the input), `invalid`.
- Password toggle: `<input type="password"><i>visibility</i>` in a `.field`
  (the JS swaps the icon and type).
- File/colour picker: `<input type="file"><input type="text">` in a `.field`;
  the text input shows the file name.
- Autocomplete: put a `<menu>` inside the field after the input.

## Selection controls

```html
<label class="checkbox">                <!-- checkbox | radio | switch -->
  <input type="checkbox">
  <span>Label</span>
</label>
<label class="switch icon">             <!-- icon: glyphs inside the thumb -->
  <input type="checkbox">
  <span><i>close</i><i>done</i></span>
</label>
```

Always `input` then `span`; the control is drawn on the span. Group them in a
`nav` or `.row`.

## Slider

```html
<label class="slider">
  <input type="range" value="40">
  <span></span>                         <!-- the active track; required -->
</label>
<label class="slider">                  <!-- range: two inputs -->
  <input type="range" value="20"><input type="range" value="80"><span></span>
</label>
```

Modifiers: `vertical`, `max`, sizes. Add `<div class="tooltip"></div>` after the
span for a value bubble.

## Dialog

```html
<button data-ui="#d1">Open</button>
<dialog id="d1">                        <!-- add class="modal" for showModal() -->
  <h5>Title</h5>
  <p>Body</p>
  <nav class="right-align">
    <button class="border" data-ui="#d1">Cancel</button>
    <button data-ui="#d1">OK</button>
  </nav>
</dialog>
```

The JS inserts the `.overlay` before the dialog. A click on it or Escape closes
the dialog, and it focuses `[autofocus]` on open. Sizes:
`small`/`medium`/`large`. Placement: `top`, `bottom`, `left`, `right` (sheets),
`max` (fullscreen). `header.fixed` / `footer.fixed` stay put while the body
scrolls. Native `<dialog popover>` also works.

## Navigation

```html
<nav class="left max">                  <!-- left|right|top|bottom; max = wide rail/drawer -->
  <header>
    <button class="extend square round"><i>edit</i><span>Compose</span></button>
  </header>
  <a class="active"><i>home</i><span>Home</span></a>
  <a><i>inbox</i><span>Inbox</span><div class="badge">3</div></a>
  <div class="divider"></div>
  <a><i>settings</i><span>Settings</span></a>
</nav>

<nav>                                   <!-- plain nav = horizontal flex row -->
  <button class="circle transparent"><i>menu</i></button>
  <h6 class="max">Title</h6>
  <button class="circle transparent"><i>more_vert</i></button>
</nav>
```

`a.active` marks the current item. A badge goes directly in the `<a>` or inside
the `<i>`. Other variants: `nav.toolbar` (floating pill of `<a>`; `fill`,
`vertical`, `max`), `nav.tabbed`, and `nav.vertical`. Alignment helpers
(`center-align`, `right-align`, `middle-align`) apply to `nav` and `.row`.

## Header / footer (app bars)

```html
<header class="fixed primary">          <!-- fixed = sticky -->
  <nav>
    <button class="circle transparent"><i>arrow_back</i></button>
    <h5 class="max">Page</h5>
  </nav>
</header>
```

## Tabs and pages

```html
<div class="tabs">                      <!-- .min = short indicator -->
  <a class="active" data-ui="#p1">One</a>
  <a data-ui="#p2">Two</a>
</div>
<div class="page padding active" id="p1">...</div>
<div class="page padding" id="p2">...</div>
```

`ui("#p2")` activates a `.page` and deactivates its sibling pages. It doesn't
move `a.active` on the tabs, so do that yourself. Add `left`, `right`, `top` or
`bottom` to `.page` for the enter direction.

## Menu

```html
<button class="circle transparent">
  <i>more_vert</i>
  <menu class="left no-wrap">           <!-- left|right|top, min, max, group -->
    <li><i>edit</i><span>Edit</span></li>
    <li>More
      <menu><li>Nested</li></menu>
    </li>
  </menu>
</button>
```

The menu is a **child of its trigger** and opens while the trigger holds focus.
With `data-ui="#menuId"` on the trigger, clicks toggle it instead, and it
closes on any outside click. A whole-row link has to be the `li`'s only child.

## List and expansion

```html
<ul class="list border">                <!-- also no-space|medium-space|large-space -->
  <li class="wave">
    <i>person</i>
    <div class="max"><h6 class="small">Name</h6><div>Subtitle</div></div>
    <label>12:30</label>
  </li>
  <li><a><i>link</i><span class="max">Whole row is a link</span></a></li>
  <li>
    <details>
      <summary><i>tune</i><span class="max">Expand</span><i>expand_more</i></summary>
      <div>Content</div>
    </details>
  </li>
</ul>
```

## Card

```html
<article class="round">                 <!-- small|medium|large (min height), border -->
  <img class="responsive top-round" src="...">
  <h5>Title</h5>
  <p>Body</p>
  <nav><button>Action</button></nav>
</article>
```

## Grid and row

```html
<div class="grid">                      <!-- 12 columns; no-space|medium-space|large-space -->
  <div class="s12 m6 l4">...</div>
</div>
<div class="row">                       <!-- flex row; wrap, space helpers, align helpers -->
  <i>info</i><div class="max">Text</div><button>Go</button>
</div>
```

## Feedback

```html
<progress></progress>                           <!-- no value = indeterminate -->
<progress value="40" max="100"></progress>
<progress class="circle wavy"></progress>       <!-- circle, wavy, indeterminate, max -->

<div class="snackbar" id="s1">Saved <a class="inverse-link">Undo</a></div>  <!-- .top to place at top -->
<div class="badge">3</div>                       <!-- min = dot; top|bottom|left|right; border -->

<button>Hover me<div class="tooltip bottom">Explains</div></button>  <!-- left|right|bottom; max = rich -->
```

## Table

```html
<table class="stripes">                 <!-- no-space|medium-space|large-space -->
  <thead><tr><th class="fixed">Name</th><th class="fixed min">Qty</th></tr></thead>
  <tbody><tr><td>Beer</td><td>2</td></tr></tbody>
</table>
```

Wrap it in `<div class="scroll">` when it might overflow.

## Media and empty state

`img` / `video` take sizes, `round`, `circle`, `square`, `responsive`, and
`empty-state`. `<div class="divider"></div>` or `<hr>` draws a rule; add
`vertical` for a vertical one.

## Shapes (M3 expressive)

```html
<div class="shape sunny large primary"><i>star</i></div>
<div class="shape loading-indicator rotate"></div>
```

Names: `arch arrow boom bun burst circle clamshell diamond fan flower gem
ghost-ish heart leaf-clover4 leaft-clover8 loading-indicator oval pentagon pill
pixel-circle pixel-triangle puffy puffy-diamond semicircle sided-cookie4
sided-cookie6 sided-cookie7 sided-cookie9 sided-cookie12 slanted soft-boom
soft-burst square sunny triangle very-sunny`. Animate with `rotate`,
`fast-rotate` or `slow-rotate`.

`leaft-clover8` is misspelled upstream; the correct spelling does nothing.
