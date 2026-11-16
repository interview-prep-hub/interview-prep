# HTML and CSS Essentials

> **Tier:** T2 (working knowledge) · **Related:** [Tailwind CSS](03-tailwind-css.md), [Caching and performance](08-caching-and-performance.md), [Browser DOM and events](../03-languages/javascript/06-browser-dom-events.md), [Views, ERB, forms and assets](../05-ruby-on-rails/07-views-erb-forms-assets.md), [React fundamentals](../06-react/01-fundamentals.md)
> **On your resume:** ERB views and React components in the Colochousing web app at Devntech, React dashboards in the PureLogics LMS, Tailwind CSS in your skills, and the Web Development labs you assisted at PUCIT.

## Quick revision
- Semantic elements (`header`, `nav`, `main`, `article`, `section`, `aside`, `footer`, `button`, `label`) carry meaning: accessibility, SEO and built-in behavior for free.
- `button` for actions, `a href` for navigation; a clickable `div` has no role, focus or keyboard support. One `h1` per page; never skip heading levels for a smaller font.
- Every input needs a `label` (a placeholder is not one); client-side validation is UX, server-side validation is mandatory.
- `alt` describes informative images; `alt=""` hides decorative ones. First rule of ARIA: use a native element instead.
- Keyboard: everything interactive reachable with Tab, visible `:focus-visible` styles, no positive `tabindex`, focus trapped and restored in modals.
- Contrast (WCAG AA): 4.5:1 for body text, 3:1 for large text and UI component boundaries.
- SEO: unique `title`, meta description, canonical URL, real headings, server-rendered content, JSON-LD structured data, fast pages.
- Box model: content, padding, border, margin. `box-sizing: border-box` makes `width` include padding and border. Vertical margins collapse (not inside flex or grid).
- `position`: static, relative (offset; anchors absolute children), absolute (nearest positioned ancestor), fixed (viewport), sticky (relative until a scroll threshold inside its scroll container).
- `z-index` only competes inside one stacking context; `transform`, `opacity < 1`, `filter` and positioned elements with a `z-index` create new ones.
- Cascade: origin and importance, then cascade layers, then specificity (inline > IDs > classes, attributes, pseudo-classes > elements, pseudo-elements), then source order. Text properties inherit (`color`, `font`); box properties do not (`margin`, `padding`, `border`).
- `rem` = root font size, `em` = the element's font size, `%` = parent size, `vw`/`vh` = viewport, `dvh` = viewport minus mobile browser bars.
- Flexbox lays out in one dimension, grid in two. Grid for page structure, flex inside components.
- Mobile-first: base styles for phones, then `@media (min-width: ...)`. Container queries respond to the component's container width.
- Custom properties (`--brand: #0e7490`) cascade, inherit and can change at runtime; Sass variables cannot.
- Pseudo-classes select states (`:hover`, `:focus-visible`, `:nth-child()`, `:has()`); pseudo-elements style parts (`::before`, `::after`, `::placeholder`). Animate `transform` and `opacity`; respect `prefers-reduced-motion`.
- Organizing CSS: BEM naming, CSS Modules (scoped at build time), CSS-in-JS (styles in JS), utility-first (Tailwind).

## 1. Semantic HTML

Semantic elements describe what content *is*, not how it looks. Screen readers announce landmarks and let users jump between them, search engines understand the structure, and native elements bring keyboard behavior you would otherwise rebuild badly.

```html
<body>
  <header>
    <nav aria-label="Main">
      <a href="/houses/12">Overview</a>
      <a href="/houses/12/bills" aria-current="page">Bills</a>
    </nav>
  </header>
  <main>
    <h1>Bills</h1>
    <section aria-labelledby="open-bills">
      <h2 id="open-bills">Open bills</h2>
      <article><h3>Internet</h3><p>45.00 EUR, due <time datetime="2026-11-01">1 Nov</time></p></article>
    </section>
  </main>
  <footer>Contact the house admin</footer>
</body>
```

| Element | Use it for |
|---|---|
| `header`, `footer`, `aside` | introductory, closing and tangential content of the page or a section |
| `nav` | major navigation blocks (label several with `aria-label`) |
| `main` | the page's unique main content, once per page |
| `article` | a self-contained item: a post, a listing, a bill card |
| `section` | a thematic group that has a heading |
| `button` / `a href` | actions (submit, open a modal, toggle) / navigation to a URL |
| `table`, `th scope`, `caption` | tabular data, never layout |

A `button` is focusable, activates with Enter and Space, is announced as a button and submits its form. A `div` with an `onclick` does none of that until you add `role`, `tabindex` and key handlers.

## 2. Forms and input types

```html
<form action="/bills" method="post">
  <label for="bill-title">Title</label>
  <input id="bill-title" name="bill[title]" type="text" required maxlength="80">

  <label for="bill-amount">Amount (EUR)</label>
  <input id="bill-amount" name="bill[amount]" type="number" min="0" step="0.01" inputmode="decimal" required>

  <label for="payer-email">Payer email</label>
  <input id="payer-email" name="bill[payer_email]" type="email" autocomplete="email" aria-describedby="email-hint">
  <p id="email-hint">The receipt is sent here.</p>

  <button type="submit">Create bill</button>
</form>
```

- `name` is what gets submitted; `id` connects the `label`. Rails `form_with` generates `bill[title]`-style names that arrive as nested params.
- Input types bring the right mobile keyboard and native validation: `email`, `tel`, `url`, `number`, `date`, `time`, `search`, `password`, `file` (with `accept="image/*"`), `checkbox`, `radio` (same `name` = one group, wrapped in `fieldset` and `legend`), `hidden`.
- Built-in constraints (`required`, `min`, `max`, `minlength`, `maxlength`, `pattern`, `step`) improve UX, but anyone can bypass them: validate on the server too. `autocomplete` tokens (`email`, `current-password`, `new-password`, `one-time-code`) help autofill and password managers.
- A `button` inside a form defaults to `type="submit"`; give other buttons `type="button"` or they submit the form.
- HTML forms only send GET and POST. GET puts fields in the query string (search, filters); Rails fakes PATCH and DELETE with a hidden `_method` field.

## 3. Accessibility

- **Target:** WCAG 2.1 or 2.2 level AA is the usual bar. The European Accessibility Act applies to many consumer-facing digital products and services in the EU from mid-2025, which matters for EU clients.
- **Images:** informative images get a description ("Floor plan of the second floor"), decorative ones `alt=""`, and icon-only buttons need an accessible name (`aria-label="Close"` or visually hidden text).
- **Labels and errors:** every control has a label, radio groups get `fieldset` and `legend`, and error messages are linked with `aria-describedby` so they are read with the field.
- **ARIA basics:** ARIA changes what assistive technology announces, never behavior. Useful attributes: `aria-label`, `aria-labelledby`, `aria-describedby`, `aria-expanded` (disclosure buttons), `aria-current="page"`, `aria-live="polite"` (announce "Saved" after an async update), `aria-hidden="true"` (decorative icons). No ARIA is better than wrong ARIA.
- **Keyboard and focus:** Tab and Shift+Tab follow DOM order, Enter activates links and buttons, Space toggles buttons and checkboxes, Escape closes dialogs. `tabindex="0"` makes a custom widget focusable, `tabindex="-1"` allows focus only from code, and positive values break the order. Never remove outlines without a replacement; style `:focus-visible`. A modal must move focus inside, trap it, close on Escape and return focus to its trigger; the native `dialog` element opened with `showModal()` handles much of this. Add a "skip to content" link on pages with long navigation.
- **Color:** 4.5:1 contrast for normal text, 3:1 for large text (about 24px, or about 19px bold) and for input borders and icons. Never signal meaning by color alone; add text or an icon to errors.
- **Testing:** a keyboard-only pass, Lighthouse or axe in DevTools, a screen reader (NVDA on Windows, VoiceOver on macOS and iOS, TalkBack on Android) and 200% zoom. In React the label attribute is `htmlFor`; React Native has its own accessibility props.

## 4. SEO basics

```html
<head>
  <title>Retro Zelda cartridge - Game Store</title>
  <meta name="description" content="Original cartridge, tested, ships in two days.">
  <link rel="canonical" href="https://shop.example.com/products/zelda-cartridge">
  <meta property="og:image" content="https://shop.example.com/images/zelda.jpg">
  <script type="application/ld+json">
    {"@context": "https://schema.org", "@type": "Product", "name": "Retro Zelda cartridge",
     "offers": {"@type": "Offer", "price": "39.99", "priceCurrency": "EUR"}}
  </script>
</head>
```

- A unique `title` and meta description per page; descriptive link text and alt text; one clear `h1`.
- `rel="canonical"` tells search engines which URL is the real one when filters, tracking parameters or sorting create duplicates.
- Content present in the HTML (server-rendered or static) is crawled more reliably than content that only appears after client-side fetching. Add `robots.txt`, `sitemap.xml`, and `<meta name="robots" content="noindex">` on private or staging pages.
- Structured data (JSON-LD with schema.org types such as Product, Offer, BreadcrumbList) can earn rich results; Open Graph tags control link previews. HTTPS, mobile-friendliness and Core Web Vitals also feed ranking ([Caching and performance](08-caching-and-performance.md)).

## 5. The box model

```css
*, *::before, *::after { box-sizing: border-box; }   /* the usual reset */

.card {
  width: 300px;
  padding: 16px;
  border: 1px solid #d4d4d8;
  margin: 24px;
}
/* content-box (default): rendered width = 300 + 2*16 + 2*1 = 334px
   border-box:            rendered width = 300px, content area shrinks to 266px
   margin is outside the box in both cases */
```

Adjacent vertical margins of block elements **collapse** into the larger one (24px + 16px gives 24px, not 40px), and a first child's top margin can leak out of its parent; it does not happen inside flex or grid containers, which is one reason `gap` is easier. Inline elements (`span`, `a`) ignore `width` and `height`, and their vertical padding does not push surrounding lines; `inline-block` respects sizes while flowing like text.

## 6. Display and hiding

`display: block` takes the full width and starts on a new line; `inline` flows with text; `inline-block` flows but accepts sizes; `flex` and `grid` lay out children; `none` removes the element from layout.

| | `display: none` | `visibility: hidden` | `opacity: 0` |
|---|---|---|---|
| Takes up space | no | yes | yes |
| Clickable and focusable | no | no | yes, which is a trap |
| Read by screen readers | no | no | yes |
| Can transition | no | not smoothly | yes |

For text that only screen readers should hear (an icon button's label), use a visually hidden class (Tailwind calls it `sr-only`: a 1px absolutely positioned, clipped box), not `display: none`.

## 7. Positioning

| Value | Offsets are relative to | Keeps its space in the flow | Typical use |
|---|---|---|---|
| `static` | not offsettable | yes | the default |
| `relative` | its own normal position | yes | small nudges; anchor for absolute children |
| `absolute` | nearest positioned ancestor (else the initial containing block) | no | badges, dropdowns, overlays |
| `fixed` | the viewport (unless an ancestor has a `transform` or `filter`) | no | app bars, chat bubbles, toasts |
| `sticky` | acts relative until a scroll threshold, then sticks inside its scroll container | yes | table headers, section headers |

```css
.card { position: relative; }
.card__badge { position: absolute; top: 8px; right: 8px; }   /* pinned to the card's corner */
.bills-table th { position: sticky; top: 0; background: white; }
```

When `sticky` does nothing: there is no `top` (or `bottom`), an ancestor has `overflow: hidden` or `auto` (that ancestor becomes the scroll container), or the parent is barely taller than the element, so there is no room to stick.

## 8. z-index and stacking contexts

`z-index` works on positioned elements and on flex and grid items. A **stacking context** is a self-contained layer: z-index values inside it only compete with each other, and the whole context is stacked as one unit within its parent. New contexts come from the root, positioned elements with a `z-index` other than `auto`, `position: fixed` or `sticky`, `opacity` below 1, `transform`, `filter`, `isolation: isolate` and a few others.

The classic bug: a modal with `z-index: 9999` still appears under the site header because the modal lives inside a parent with `transform` or `z-index: 1`, and nothing can escape its parent's context. Fix it by rendering the modal at the end of `body` (a React portal or the native `dialog` element, which uses the top layer) or by removing the property that created the context. Keep a small z-index scale (dropdown 10, sticky header 20, modal 50, toast 60) instead of escalating numbers.

## 9. Cascade, specificity and inheritance

When several declarations target the same property, the cascade picks the winner in this order:
1. **Origin and importance:** `!important` declarations beat normal ones.
2. **Cascade layers (`@layer`):** later layers win, and unlayered styles beat layered ones (Tailwind v4 relies on layers).
3. **Specificity**, compared as (IDs, classes and attributes and pseudo-classes, elements and pseudo-elements); an inline `style` attribute beats any selector.
4. **Source order:** the later declaration wins.

| Selector | Specificity |
|---|---|
| `p` | (0,0,1) |
| `.price`, `[type="email"]`, `:hover` | (0,1,0) |
| `p.price:hover` | (0,2,1) |
| `#cart .item a` | (1,1,1) |
| `:where(.card) a` | (0,0,1): `:where()` always counts zero |
| `:is(#main, .card) a` | (1,0,1): `:is()`, `:not()` and `:has()` count their most specific argument |

The universal selector and combinators add nothing. Text-related properties inherit (`color`, `font-family`, `font-size`, `line-height`, `text-align`, `visibility`, `cursor`); box properties do not (`margin`, `padding`, `border`, `background`, `width`, `display`). The keywords `inherit`, `initial`, `unset` and `revert` override that. When a style "does not apply", open DevTools: the Styles pane strikes out losing declarations and Computed shows the final value and its source.

## 10. Units

| Unit | Relative to | Good for |
|---|---|---|
| `px` | a CSS pixel | borders, shadows, small details |
| `rem` | the root font size (16px by default) | font sizes and spacing; respects the user's browser font setting |
| `em` | the element's font size (the parent's, when setting `font-size`) | padding that scales with a component's text; compounds when nested |
| `%` | the parent's size (padding and margin use the parent's width) | fluid widths |
| `vw`, `vh` | 1% of the viewport width or height | full-screen sections |
| `dvh`, `svh`, `lvh` | dynamic, small, large viewport height | mobile layouts: `100vh` ignores collapsing browser toolbars, `100dvh` tracks them |
| `ch` | width of the "0" glyph | readable line length (`max-width: 65ch`) |
| `fr` | a share of free space in a grid | grid tracks |

Fluid type without media queries: `font-size: clamp(1rem, 0.9rem + 0.5vw, 1.25rem);`.

## 11. Flexbox

A flex container lays its children along a **main axis** (`flex-direction: row` or `column`) and aligns them on the **cross axis**. On the container: `justify-content` (main axis), `align-items` (cross axis), `flex-wrap`, `gap`. On items: `flex: 1` (grow 1, shrink 1, basis 0), `flex: none` (never grow or shrink), `align-self`, and `order` (visual only, so it breaks keyboard order; avoid it).

```css
.center { display: flex; justify-content: center; align-items: center; }

.navbar { display: flex; align-items: center; gap: 1rem; }
.navbar__account { margin-left: auto; }    /* pushes this item and the rest to the right */

.media { display: flex; gap: 0.75rem; }
.media__avatar { flex: none; width: 48px; }
.media__body { flex: 1; min-width: 0; }    /* lets long text shrink and truncate */
```

Flex items default to `min-width: auto`, so a long URL or table refuses to shrink and overflows; `min-width: 0` on the item fixes it. React Native lays out with flexbox too (column by default), so this knowledge transfers directly to the Coloc app.

## 12. Grid

```css
.cards {          /* as many 220px+ columns as fit; auto-fit instead would collapse empty tracks */
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 1rem;
}

.layout {                                  /* mobile-first page layout with named areas */
  display: grid;
  grid-template-areas: "header" "sidebar" "main" "footer";
  grid-template-rows: auto auto 1fr auto;
  min-height: 100dvh;
}
@media (min-width: 768px) {
  .layout {
    grid-template-columns: 240px 1fr;
    grid-template-rows: auto 1fr auto;
    grid-template-areas: "header header" "sidebar main" "footer footer";
  }
}
.layout > header { grid-area: header; }
.layout > aside { grid-area: sidebar; }
.layout > main { grid-area: main; }
.layout > footer { grid-area: footer; }
```

| | Flexbox | Grid |
|---|---|---|
| Dimensions | one: a row or a column | two: rows and columns together |
| Sizing driven by | the content | the container's track definitions |
| Best for | navbars, toolbars, button groups, centering, media objects | page layouts, card galleries, dashboards, aligned form columns |
| Aligning items across rows | no | yes |
| Overlapping items | awkward | easy (same grid area) |

## 13. Responsive design

- Without `<meta name="viewport" content="width=device-width, initial-scale=1">`, phones render a desktop-width page and zoom out.
- **Mobile-first:** write base styles for small screens and add `@media (min-width: ...)` layers for larger ones. Phones download less override CSS, and it matches Tailwind's breakpoint prefixes.
- Choose breakpoints where the content breaks, not per device; 640, 768, 1024 and 1280px are common.
- Stay fluid between breakpoints: `max-width` containers, `%` and `fr`, `clamp()`, and responsive images with `srcset` ([Caching and performance](08-caching-and-performance.md)).
- **Container queries** style a component by its container's size, so the same card works in a narrow sidebar and a wide main column. They are supported in all major browsers in recent versions.

```css
.card-list { container-type: inline-size; }
@container (min-width: 480px) {
  .card { display: grid; grid-template-columns: 120px 1fr; }
}
```

Other useful media features: `prefers-color-scheme: dark`, `prefers-reduced-motion: reduce`, `hover: hover`, `pointer: coarse`.

## 14. Custom properties

```css
:root { --color-brand: #0e7490; --color-text: #18181b; --radius: 0.5rem; }
[data-theme="dark"] { --color-text: #f4f4f5; }

.button { background: var(--color-brand); border-radius: var(--radius); color: white; }
.alert { color: var(--color-danger, #b91c1c); }   /* fallback when the variable is undefined */
```

Sass variables disappear at compile time; custom properties are live values that cascade and inherit, can be redefined per component or inside a media query, and can be changed from JavaScript with `document.documentElement.style.setProperty("--color-brand", "#7c3aed")`. They are the basis of theming and dark mode, and Tailwind v4 exposes its whole theme as custom properties.

## 15. Pseudo-classes and pseudo-elements

A **pseudo-class** (one colon) selects an element in a state or position; a **pseudo-element** (two colons) styles a part of an element that is not in the DOM.

| Pseudo-classes | Pseudo-elements |
|---|---|
| `:hover`, `:active`, `:focus`, `:focus-visible`, `:focus-within` | `::before`, `::after` (need a `content` value) |
| `:disabled`, `:checked`, `:required`, `:invalid` | `::placeholder` |
| `:first-child`, `:last-child`, `:nth-child(2n+1)`, `:nth-of-type()` | `::marker` (list bullets) |
| `:not()`, `:is()`, `:where()`, `:has()`, `:empty`, `:target` | `::selection`, `::first-line`, `::first-letter` |

```css
.button:focus-visible { outline: 2px solid var(--color-brand); outline-offset: 2px; }
.field:has(input:focus-visible) label { color: var(--color-brand); }   /* :has() styles a parent */
.required::after { content: " *"; color: #b91c1c; }
tr:nth-child(even) { background: #fafafa; }
```

`:focus-visible` shows the ring for keyboard users without flashing it on every mouse click. `:has()` (the long-awaited "parent selector") is supported in all major browsers in recent versions.

## 16. Transitions and animations

```css
.dropdown { opacity: 0; transform: translateY(-4px); transition: opacity 150ms ease-out, transform 150ms ease-out; }
.dropdown.is-open { opacity: 1; transform: translateY(0); }
@keyframes spin { to { transform: rotate(360deg); } }
.spinner { animation: spin 1s linear infinite; }
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

Transitions animate between two states; `@keyframes` animations can loop and have many steps. Animate `transform` and `opacity`, which the browser can do on the compositor without layout or paint. Animating `width`, `height`, `top` or `margin` recalculates layout every frame, which causes jank and, if content moves, layout shift.

## 17. Organizing CSS: BEM, CSS Modules, CSS-in-JS, utility-first

With CSS Modules you write `.card` in `BillCard.module.css`, `import styles from "./BillCard.module.css"`, and use `className={styles.card}`; the build rewrites it to a unique class name.

| Approach | How it works | Strengths | Weaknesses |
|---|---|---|---|
| BEM | naming convention: `.bill-card`, `.bill-card__amount`, `.bill-card--overdue` | no tooling, flat specificity, readable | global namespace, long names, needs discipline |
| CSS Modules | class names scoped per file at build time | plain CSS, no collisions, no runtime cost, fine with Server Components | conditional classes are verbose, theming via custom properties |
| CSS-in-JS (styled-components, Emotion) | styles written in JS, often driven by props | co-location, dynamic styles, theming | runtime cost and SSR complexity; runtime libraries need client components under React Server Components |
| Utility-first (Tailwind) | compose small single-purpose classes in markup | speed, consistent design tokens, tiny CSS output, no naming | long class lists, learning curve; see [Tailwind CSS](03-tailwind-css.md) |

## Common pitfalls
- Clickable `div`s instead of buttons and links; icon buttons with no accessible name; a placeholder as the only label.
- `outline: none` with no replacement focus style.
- Fighting specificity with IDs and `!important` instead of fixing the selector design.
- `z-index: 9999` inside a parent that created a stacking context.
- `100vh` sections hidden behind mobile browser bars (use `dvh`); flex items overflowing because of `min-width: auto`.
- `position: sticky` with no `top`, or inside an ancestor with `overflow: hidden`.
- Buttons in forms without `type="button"` that submit unexpectedly.
- Desktop-first CSS full of overrides at small sizes; animating layout properties.
- A staging site without `noindex` (it gets indexed) or a production site that accidentally keeps it.

## How to talk about this in an interview
- Tie it to real screens: ERB views in the Colochousing Rails 7 app, React components through React on Rails, the role-based dashboards in the PureLogics LMS, and flexbox layouts in the Coloc resident app. The PUCIT Web Development labs give you a teaching angle: explain the box model, specificity or flex vs grid the way you would to a student, with a tiny example.
- On accessibility, be honest about your depth and concrete about your habits: semantic elements, labels, a keyboard pass and a Lighthouse or axe check before shipping.
- When asked about CSS architecture, compare approaches with trade-offs rather than declaring a favorite.

> **Fill in:** a layout or styling bug you fixed (overflow, z-index, sticky header, responsive breakage) and how you found the cause in DevTools.

> **Fill in:** which styling approach the Colochousing web app and the Coloc app used (plain CSS or Sass, Tailwind, NativeWind, component library).

## Interview questions

**Q1. What is semantic HTML, and why does it matter?**
Using elements for their meaning (`nav`, `main`, `article`, `button`, `label`) rather than generic `div`s. Screen readers announce roles and landmarks, keyboard behavior comes built in, search engines understand the structure, and the code is easier to read.
*Follow-up:* Why is a clickable `div` bad? It has no role, cannot receive focus and ignores Enter and Space unless you rebuild all of that.

**Q2. Explain the CSS box model and `box-sizing`.**
Every element is content plus padding plus border, with margin outside. With the default `content-box`, `width` sets only the content, so padding and border make the box bigger; `border-box` makes `width` include padding and border, which is far easier to reason about, hence the global reset.
*Follow-up:* What is margin collapsing? Adjacent vertical margins between blocks merge into the larger one; it does not happen in flex or grid layouts.

**Q3. How does the browser decide which CSS rule wins?**
The cascade: importance and origin first, then cascade layers, then specificity (inline style, then IDs, then classes, attributes and pseudo-classes, then elements and pseudo-elements), then source order. `:where()` adds zero specificity, while `:is()`, `:not()` and `:has()` take their most specific argument.
*Follow-up:* How do you avoid specificity wars? Mostly single-class selectors (BEM, CSS Modules or utilities), cascade layers, and no IDs or `!important` for styling.

**Q4. Flexbox vs grid: when do you use each?**
Flexbox for one-dimensional, content-driven layouts: navbars, toolbars, centering, media objects. Grid for two-dimensional, container-driven layouts: page shells, card galleries, dashboards. They combine well: grid for the page, flex inside the components.
*Follow-up:* Make a responsive card grid without media queries. `grid-template-columns: repeat(auto-fill, minmax(220px, 1fr))`.

**Q5. How do you center an element horizontally and vertically?**
On the parent, `display: flex; justify-content: center; align-items: center;` or `display: grid; place-items: center;`. For a known-size absolutely positioned element, `inset: 0; margin: auto;` also works, and `margin: 0 auto` centers a block horizontally.
*Follow-up:* Why might vertical centering fail? The parent has no height to center within.

**Q6. Explain the `position` values. Why would `sticky` not work?**
Static is the default; relative offsets from the normal position and anchors absolute children; absolute leaves the flow and positions against the nearest positioned ancestor; fixed positions against the viewport; sticky acts relative until a scroll threshold and then sticks within its scroll container. Sticky fails without a `top`, inside an ancestor with `overflow: hidden` or `auto`, or when the parent leaves no room.
*Follow-up:* When does `fixed` stop being relative to the viewport? When an ancestor has a `transform` or `filter`.

**Q7. Your modal has `z-index: 9999` but appears behind the header. Why?**
The modal sits inside an element that created its own stacking context (a `transform`, `opacity` below 1, or a positioned element with a `z-index`), so its z-index only competes inside that context. Render it at the end of `body` with a portal or use the native `dialog` top layer, or remove the property creating the context.
*Follow-up:* How do you prevent it? A small shared z-index scale and portals for overlays.

**Q8. em vs rem vs px vs %?**
`px` is fixed; `rem` follows the root font size; `em` follows the element's own font size and compounds when nested; `%` follows the parent's size. Use `rem` for font sizes and spacing so the page respects the user's font settings, `px` for borders and fine details.
*Follow-up:* What is wrong with `100vh` on phones? It ignores the collapsing browser toolbars; `100dvh` adapts.

**Q9. How do you make a page accessible? What do you check first?**
Semantic elements, labels on every input, alt text, sufficient contrast, full keyboard operation with visible focus, focus management in modals, and ARIA only where native elements fall short. Check with a keyboard-only pass, Lighthouse or axe, and a screen reader for key flows.
*Follow-up:* What is the first rule of ARIA? Do not use ARIA when a native element provides the semantics and behavior.

**Q10. What is mobile-first responsive design, and what are container queries?**
Base styles target small screens, and `min-width` media queries add layout for bigger ones, keeping phone CSS small and priorities clear. Container queries respond to the size of a component's container instead of the viewport, so a component adapts wherever it is placed.
*Follow-up:* Which meta tag is essential? `<meta name="viewport" content="width=device-width, initial-scale=1">`.

**Q11. BEM vs CSS Modules vs CSS-in-JS vs utility-first?**
BEM is a naming convention with no tooling. CSS Modules scope plain CSS per file at build time. CSS-in-JS writes styles in JavaScript with dynamic props but adds runtime cost and friction with Server Components. Utility-first composes small classes in markup with consistent tokens and tiny output, at the price of long class lists. Choose by team, framework and rendering model.
*Follow-up:* Which fits React Server Components best? CSS Modules, plain CSS or Tailwind, since they need no runtime.

## Further reading
- [MDN: HTML](https://developer.mozilla.org/en-US/docs/Web/HTML) and [MDN: CSS](https://developer.mozilla.org/en-US/docs/Web/CSS)
- [MDN: Accessibility](https://developer.mozilla.org/en-US/docs/Web/Accessibility)
- [WCAG overview (W3C)](https://www.w3.org/WAI/standards-guidelines/wcag/)
