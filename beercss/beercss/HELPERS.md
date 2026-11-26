# Helpers (beercss 5.0.3)

Every name below exists in `beer.css` 5.0.3. Anything else does nothing.

## Scales: where guesses go wrong

| family | values |
| --- | --- |
| padding / margin | `padding` (1rem), `no-` 0, `tiny-` .25rem, `small-` .5rem, `large-` 1.5rem; margin also `auto-margin`. **No medium, no extra.** |
| side variants | `top-` `bottom-` `left-` `right-` `horizontal-` `vertical-` + `padding`/`margin` (1rem on that side only) |
| space | `space`, `no-` `tiny-` `small-` `medium-` `large-` `extra-space` |
| round | `round` (2rem), `small-round`, `large-round`, `no-round`, `top-` `bottom-` `left-` `right-round`. **No medium, no tiny.** |
| width / height | `small-` `medium-` `large-` `auto-` + `width`/`height` |
| element size | `tiny small medium large extra` |

Size and side don't combine on one class (no `small-top-padding`).

`space` sets the **gap between children** on `nav`, `.row`, `.grid`, `menu`,
`.list`, `table`, `.tooltip` and `.shape`. On any other element it's a vertical
spacer block.

## Layout and position

```
grid  row  max  min  wrap  no-wrap  responsive  scroll  no-scroll
s1..s12  m1..m12  l1..l12        grid spans: s ≤600px, m 601–992px, l ≥993px
s  m  l                          visibility: shown only at those breakpoints
horizontal  vertical             flex direction
absolute  fixed  front  back
top  bottom  left  right  center  middle      placement (with absolute/fixed, menus, dialogs, tooltips, badges)
left-align  right-align  center-align  top-align  bottom-align  middle-align
```

`class="m l"` = hidden on phones. `class="s"` = phones only.

## Colour

M3 roles, each also `-text` (foreground only) and `-border`:

```
primary  primary-container  secondary  secondary-container
tertiary  tertiary-container  error  error-container
surface  surface-variant  surface-dim  surface-bright
surface-container-lowest  surface-container-low  surface-container
surface-container-high  surface-container-highest
inverse-surface  inverse-primary  background
transparent  white  black  link  inverse-link
```

Prefer roles: they follow light/dark mode and `ui("theme")`. Fixed palette
(bare, `-text`, `-border`, tints `1`–`10` such as `red5`, `blue-grey2`):

```
red pink purple deep-purple indigo blue light-blue cyan teal green
light-green lime yellow amber orange deep-orange brown grey blue-grey
```

## Typography

```
h1..h6 (as classes, e.g. <p class="h5">)  + small / large on any heading
small-text  medium-text  large-text
bold  italic  underline  overline  upper  lower  capitalize  truncate
no-line  tiny-line  small-line  medium-line  large-line  extra-line
```

## Borders and shape

```
border  no-border  circle  square  fill
divider  small-divider  medium-divider  large-divider
```

## Effects

```
elevate  small-elevate  medium-elevate  large-elevate  no-elevate
shadow  top-shadow  bottom-shadow  left-shadow  right-shadow
blur  small-blur  large-blur
opacity  small-opacity  medium-opacity  large-opacity  no-opacity
zoom  tiny-zoom  small-zoom  medium-zoom  large-zoom  extra-zoom
wave  no-wave                                    hover/press state layer
ripple  fast-ripple  slow-ripple                 click ripple (needs JS)
rotate  fast-rotate  slow-rotate
```

## State

```
active        open/selected (dialog, menu, snackbar, page, tab, nav link, overlay)
invalid       on .field
indeterminate on progress
light  dark   on body
none          hides a badge
```

There is no generic `hide` class. Use the `hidden` attribute or `s`/`m`/`l`.
