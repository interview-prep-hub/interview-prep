# Tailwind CSS

> **Tier:** T2 (working knowledge) · **Related:** [HTML and CSS](02-html-css.md), [React patterns and architecture](../06-react/07-patterns-and-architecture.md), [Views, ERB, forms and assets](../05-ruby-on-rails/07-views-erb-forms-assets.md), [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [React Native fundamentals](../08-react-native-expo/01-react-native-fundamentals.md)
> **On your resume:** Tailwind CSS in your skills list, next to Rails views, React and React Native (Expo) work at Devntech.

## Quick revision
- **Utility-first:** style by composing small single-purpose classes (`flex`, `gap-4`, `px-3`, `text-sm`, `bg-teal-600`) in markup instead of writing a custom CSS rule per component.
- Tailwind scans your source files as text and generates CSS only for the class names it finds, so the output stays small.
- It only recognizes complete class strings: `bg-${color}-500` never gets generated; map values to full class names instead.
- **v4 (January 2025)** is CSS-first: `@import "tailwindcss";`, design tokens in `@theme`, automatic content detection, `@tailwindcss/vite` or `@tailwindcss/postcss`, built on cascade layers and modern CSS.
- **v3** uses `tailwind.config.js` with a `content` array and the `@tailwind base; @tailwind components; @tailwind utilities;` directives through PostCSS.
- Breakpoint prefixes are mobile-first `min-width` queries: `sm` 640px, `md` 768px, `lg` 1024px, `xl` 1280px, `2xl` 1536px. Unprefixed classes apply everywhere.
- State variants: `hover:`, `focus-visible:`, `disabled:`, `group-hover:` (parent marked `group`), `peer-checked:` (earlier sibling marked `peer`), `aria-*`, `data-*`. Variants stack: `md:hover:bg-teal-700`.
- `dark:` follows `prefers-color-scheme` by default; a manual toggle needs a class-based variant.
- Arbitrary values for one-offs (`w-[327px]`, `bg-[#1da1f2]`, `grid-cols-[200px_1fr]`); values you repeat belong in the theme.
- Reuse through components (React components, Rails partials, ViewComponents), with `@apply` only for markup you do not control.
- `clsx` joins conditional classes, `tailwind-merge` resolves conflicts (`px-4` vs `px-8`), `cva` defines typed variants; `cn()` is `twMerge(clsx(...))`.
- v4 renames: `shadow-sm` became `shadow-xs` (the scale shifted), `rounded` became `rounded-sm`, `outline-none` became `outline-hidden`, `ring` became `ring-3`; borders default to `currentColor` and `ring` is 1px.
- Rails: the `tailwindcss-rails` gem wraps the standalone CLI (no Node needed) and `bin/dev` runs the watcher.
- Next.js uses `@tailwindcss/postcss`; Vite stacks (TanStack Start, React with Vite) use `@tailwindcss/vite`.
- React Native: NativeWind adds `className` to RN components; NativeWind v4 targets Tailwind v3.
- Pros: speed, consistent tokens, no naming, no dead CSS, styles next to markup. Cons: noisy markup, learning curve, needs tooling, class conflicts when composing.

## 1. The utility-first idea

```html
<!-- Traditional: invent a name here, write its CSS in another file -->
<div class="bill-card">...</div>

<!-- Utility-first: the styling is readable where it is used -->
<div class="rounded-lg border border-zinc-200 bg-white p-4 shadow-sm">
  <p class="text-sm text-zinc-500">Internet</p>
  <p class="mt-1 text-2xl font-semibold text-zinc-900">45.00 EUR</p>
</div>
```

Why teams like it: the spacing, color and type scales act as a design system, so values stay consistent; you never switch files or invent names; deleting markup deletes its styles, so CSS stops growing with every feature; utilities all have single-class specificity, so there are no specificity wars; and responsive and state styles sit right next to the base style.

| Pros | Cons |
|---|---|
| Fast to build and to change; no naming | Long class lists can hurt readability |
| Consistent spacing, color and type tokens | Learning curve: you must know the class names |
| Small CSS that does not grow with the app | Needs editor tooling (IntelliSense, the Prettier class-sorting plugin) |
| No dead CSS, no specificity fights | Styling is coupled to markup; repeated patterns need components |
| Works anywhere classes are strings: JSX, ERB, NativeWind | Composing components needs tailwind-merge to resolve conflicts |

## 2. How Tailwind generates CSS, and why dynamic classes fail

Tailwind does not run your code. It reads source files as plain text, collects anything that looks like a class name, and generates CSS for those tokens only.

```tsx
// Broken: the scanner never sees "bg-red-100" or "bg-emerald-100" as complete strings
<span className={`bg-${color}-100`} />

// Works: every class appears in full somewhere in the source
const statusClasses = {
  paid: "bg-emerald-100 text-emerald-800",
  overdue: "bg-red-100 text-red-800",
  pending: "bg-amber-100 text-amber-800",
} as const;

<span className={statusClasses[status]} />
```

ERB follows the same rule: `class="<%= bill.overdue? ? 'text-red-700' : 'text-zinc-700' %>"` works because both strings appear in full. For class names that only exist at runtime (from a database or CMS), safelist them: the `safelist` option in a v3 config, or `@source inline(...)` in recent v4 releases.

## 3. Setup: v4 vs v3

**v4** keeps configuration in CSS:

```css
/* app.css */
@import "tailwindcss";

@theme {
  --color-brand-50: #ecfeff;
  --color-brand-600: #0e7490;
  --font-display: "Inter", sans-serif;
  --breakpoint-3xl: 1920px;
}
```

This generates `bg-brand-600`, `text-brand-50`, `font-display` and a `3xl:` variant, and every token is also emitted as a CSS variable (`var(--color-brand-600)`) for custom CSS or inline styles. Content detection is automatic (it skips `.gitignore`d and binary files); add other paths with `@source "../node_modules/@acme/ui";`. Custom utilities use `@utility`, custom variants `@custom-variant`, JS plugins `@plugin`, and `@config` loads a legacy JS config during a migration. v4 targets modern browsers (Safari 16.4+, Chrome 111+, Firefox 128+), a real constraint if a client must support older devices.

```ts
// vite.config.ts: React with Vite, TanStack Start and other Vite-based stacks
import { defineConfig } from "vite";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [tailwindcss()],
});
```

```js
// postcss.config.mjs: Next.js and other PostCSS setups; then @import "tailwindcss" in app/globals.css
export default {
  plugins: { "@tailwindcss/postcss": {} },
};
```

**v3** keeps configuration in JavaScript:

```js
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./app/views/**/*.erb", "./app/helpers/**/*.rb", "./app/javascript/**/*.{js,ts,jsx,tsx}"],
  darkMode: "class",
  theme: {
    extend: {
      colors: { brand: { 50: "#ecfeff", 600: "#0e7490" } },
      fontFamily: { display: ["Inter", "sans-serif"] },
    },
  },
  plugins: [],
};
```

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

| | v3 | v4 |
|---|---|---|
| Configuration | `tailwind.config.js` | CSS: `@theme`, `@utility`, `@custom-variant` |
| Entry stylesheet | three `@tailwind` directives | `@import "tailwindcss";` |
| Finding classes | `content` globs (miss a folder and its classes vanish) | automatic detection plus `@source` |
| Build integration | PostCSS plugin plus autoprefixer | `@tailwindcss/vite`, `@tailwindcss/postcss` or `@tailwindcss/cli`; prefixing and imports built in |
| Theme values | a JS object and the `theme()` function | CSS variables usable anywhere |
| Container queries | official plugin | built in (`@container` on the parent, `@md:` on children) |
| Upgrading | | `npx @tailwindcss/upgrade` automates most changes |

## 4. Responsive design, state variants and dark mode

```html
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">...</div>
<h1 class="text-xl md:text-3xl">Bills</h1>
<nav class="hidden md:flex">...</nav>
```

Unprefixed classes apply at every size and `md:` means "768px and up", so you style the phone first and add prefixes for larger screens. Using `sm:` to target phones is the classic mistake. `max-*` variants target below a breakpoint (`max-md:hidden`), and they combine into ranges (`md:max-xl:flex`).

```html
<button class="rounded-md bg-teal-600 px-4 py-2 text-white hover:bg-teal-700 focus-visible:ring-2
               focus-visible:ring-teal-600 focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50">
  Pay rent
</button>

<a href="/houses/12" class="group block rounded-lg p-4 hover:bg-zinc-50">
  <span class="font-medium group-hover:underline">House 12</span>   <!-- reacts to hovering the parent -->
</a>

<label>
  <input type="checkbox" class="peer sr-only">
  <span class="rounded-full bg-zinc-200 px-3 py-1 peer-checked:bg-teal-600 peer-checked:text-white">Recurring</span>
</label>

<div data-state="open" class="hidden data-[state=open]:block">...</div>
```

`group` styles children based on a parent's state; `peer` styles a later sibling based on an earlier one (the peer must come first in the DOM). The visually hidden checkbox stays keyboard-focusable, so the custom toggle remains accessible.

**Dark mode:** `class="bg-white text-zinc-900 dark:bg-zinc-900 dark:text-zinc-100"`. In v4, `dark:` follows the OS preference by default. For a manual toggle, redefine the variant and put a `dark` class on `html`:

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));
```

In v3 the equivalent is `darkMode: "class"` in the config. Persist the user's choice and apply the class from a tiny inline script in the `head` to avoid a flash of the wrong theme. On larger apps, semantic tokens (a `--color-surface` variable redefined under `.dark`, used as `bg-surface`) beat sprinkling `dark:` pairs everywhere.

## 5. Arbitrary values and theme customization

- Arbitrary values: `w-[327px]`, `top-[3.25rem]`, `bg-[#1da1f2]`, `grid-cols-[200px_1fr]` (underscores become spaces), `text-[clamp(1rem,2vw,1.5rem)]`.
- CSS variables: `bg-(--brand-color)` in v4, `bg-[var(--brand-color)]` in v3.
- Arbitrary properties and variants: `[mask-type:luminance]`, `[&>li]:py-2`.
- Rule of thumb: an arbitrary value used once is fine; used three times, it deserves a token in the theme so it has a name and stays consistent.
- v4 customization: add tokens in `@theme`; replace a whole namespace by resetting it (`--color-*: initial;`) and defining your own palette. v3: `theme.extend` adds to the defaults, while top-level `theme` keys replace them.
- v4 derives spacing from one `--spacing` variable: `p-4` is `calc(var(--spacing) * 4)`, so any integer works (`p-13`) and changing `--spacing` rescales the whole UI.

## 6. Reusing styles: components first, `@apply` sparingly

When the same twelve classes appear on every button, reach for these in order:
1. A loop, so repeated markup is written once.
2. A component: a React `<Button>`, a Rails partial (`render "shared/button"`), a ViewComponent or a helper.
3. Multi-cursor editing for small, local duplication.
4. `@apply`, only where you cannot add classes to the markup: third-party widgets, rich text from an editor or CMS, base element styles.

```css
/* fine: Action Text's Trix editor renders HTML you do not control */
.trix-content a { @apply text-teal-700 underline; }
```

Using `@apply` to build `.btn`, `.card` and `.badge` classes everywhere brings back everything utility-first removed: naming, a parallel stylesheet to maintain, and ordering problems. In v4, a stylesheet processed separately (a CSS Module, a Vue or Svelte style block) must start with `@reference "../app.css";` before it can `@apply` your theme's utilities.

## 7. Variants with clsx, tailwind-merge and cva

When a component accepts a `className` prop, `px-4` from inside and `px-8` from outside both end up on the element, and the winner depends on the order of the rules in the generated stylesheet, not on the order in the class attribute. tailwind-merge fixes that by removing the earlier conflicting class.

```tsx
import type { ButtonHTMLAttributes } from "react";
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";
import { cva, type VariantProps } from "class-variance-authority";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs)); // clsx handles conditionals, twMerge keeps the last conflicting class
}

const button = cva(
  "inline-flex items-center justify-center rounded-md font-medium disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      intent: {
        primary: "bg-teal-600 text-white hover:bg-teal-700",
        secondary: "border border-zinc-300 bg-white text-zinc-900 hover:bg-zinc-50",
        danger: "bg-red-600 text-white hover:bg-red-700",
      },
      size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4", lg: "h-12 px-6 text-lg" },
    },
    defaultVariants: { intent: "primary", size: "md" },
  },
);

type ButtonProps = ButtonHTMLAttributes<HTMLButtonElement> & VariantProps<typeof button>;

export function Button({ intent, size, className, ...props }: ButtonProps) {
  return <button className={cn(button({ intent, size }), className)} {...props} />;
}

// <Button intent="danger" size="sm" className="px-8">Delete</Button>   px-8 replaces px-3
```

`cva` also supports `compoundVariants` (classes for specific combinations, such as `intent: "danger"` with `size: "lg"`), and `VariantProps` keeps the props typed. This is the pattern shadcn/ui components use. tailwind-merge has to know Tailwind's class groups, so its versions track Tailwind's (tailwind-merge v3 targets Tailwind v4, v2 targets Tailwind v3); check its compatibility notes when you upgrade either one.

## 8. v4 renamed and changed utilities

| v3 | v4 | Note |
|---|---|---|
| `shadow-sm` / `shadow` | `shadow-xs` / `shadow-sm` | the scale shifted down one step |
| `rounded-sm` / `rounded` | `rounded-xs` / `rounded-sm` | same shift; also `blur`, `drop-shadow`, `backdrop-blur` |
| `outline-none` | `outline-hidden` | `outline-none` now really sets `outline-style: none` |
| `ring` | `ring-3` | bare `ring` is now 1px and `currentColor` (was 3px blue) |
| `border` (gray-200 by default) | `border` in `currentColor` | add an explicit color such as `border-zinc-200` |
| `bg-opacity-50` | `bg-black/50` | the opacity utilities were removed; use the slash modifier |
| `flex-shrink-0`, `flex-grow` | `shrink-0`, `grow` | old aliases removed |
| `@tailwind base;` and friends | `@import "tailwindcss";` | |

Also: stacked variants now apply left to right (v3 applied them right to left), `hover:` only applies on devices that support hover, and buttons default to `cursor: default`. Run `npx @tailwindcss/upgrade` on a branch and review the UI visually; borders and focus rings are the usual surprises.

## 9. Tailwind in Rails, Next.js and TanStack Start

```bash
bundle add tailwindcss-rails
bin/rails tailwindcss:install     # adds the input stylesheet, a watcher in Procfile.dev and the build step
bin/dev                           # runs the Rails server and tailwindcss:watch together
# new apps: rails new myapp --css tailwind
```

- The gem wraps the standalone Tailwind CLI, so a Rails app needs no Node toolchain for CSS; `assets:precompile` runs `tailwindcss:build` during deploys.
- Recent versions of the gem ship Tailwind v4 and moved the input file and configuration (v3-era setups used `config/tailwind.config.js` and `app/assets/stylesheets/application.tailwind.css`). Read the gem's upgrade notes before bumping a legacy app.
- Every place class names live (views, helpers, components, Stimulus controllers) must be visible to the scanner; in v3 that means `content` globs covering `app/views`, `app/helpers` and `app/javascript`.
- **Next.js:** `create-next-app` offers Tailwind during setup; v4 uses `@tailwindcss/postcss` with `@import "tailwindcss";` in `app/globals.css`. There is no runtime, so it works with Server Components.
- **TanStack Start:** it is Vite-based, so add `@tailwindcss/vite` and import the stylesheet from the root route. Start's config and asset APIs changed during the release candidates, so follow its current docs for the exact wiring.

## 10. NativeWind for React Native

```tsx
import { Pressable, Text, View } from "react-native";

export function BillRow({ title, amount, overdue }: { title: string; amount: string; overdue: boolean }) {
  return (
    <Pressable className="flex-row items-center justify-between border-b border-zinc-200 px-4 py-3">
      <Text className="text-base text-zinc-900">{title}</Text>
      <View className={overdue ? "rounded-full bg-red-100 px-2 py-0.5" : "rounded-full bg-zinc-100 px-2 py-0.5"}>
        <Text className={overdue ? "text-xs text-red-800" : "text-xs text-zinc-700"}>{amount}</Text>
      </View>
    </Pressable>
  );
}
```

```js
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: "./global.css" });
```

- NativeWind turns Tailwind classes into React Native style objects, so you write `className` on core components.
- NativeWind v4 targets Tailwind v3: a `tailwind.config.js` using the `nativewind/preset`, a `global.css` with the three `@tailwind` directives, and Babel and Metro integration as above. Follow the install guide for your Expo SDK. A NativeWind version for Tailwind v4 has been in preview; check the docs before choosing.
- Not everything maps to native: no CSS grid, limited pseudo-classes, no hover on touch screens. RN flexbox defaults to `flex-direction: column`, so `flex-row` must be explicit.
- Alternatives include plain `StyleSheet`, Tamagui and Unistyles.

## Common pitfalls
- Building class names dynamically (`text-${size}`), so they are never generated.
- v3 `content` globs that miss a folder (ViewComponents, helpers, a shared UI package), so classes silently disappear.
- Using `sm:` to target phones; it means 640px and up.
- Composing classes without tailwind-merge, so `className` overrides lose depending on stylesheet order.
- Rebuilding a component library out of `@apply` classes.
- Arbitrary values everywhere instead of theme tokens.
- Upgrading to v4 without checking renamed utilities, border and ring defaults and the browser support floor.
- Thirty-class strings nobody can read: extract a component or a `cva` definition, and sort classes with the official Prettier plugin.
- `outline-hidden` on focusable elements without a `focus-visible:` replacement style.

## How to talk about this in an interview
- Tailwind is in your skills list, so expect "pros and cons" and "how do you keep it maintainable". Answer with trade-offs, then your habits: components for reuse, `cva` for variants, `cn()` for overrides, tokens in the theme, and the Prettier plugin for consistent class order.
- Point out that the classes are just strings, so the same approach works in ERB views, React components and, through NativeWind, in a React Native app.
- If asked about v4, describe the CSS-first config, automatic detection and the upgrade surprises (renamed shadows, `outline-hidden`, border and ring defaults).
- Do not claim Tailwind on a specific project unless you used it there.

> **Fill in:** where you used Tailwind (a Devntech project, the PureLogics LMS, the RemoteDev Next.js app, personal projects) and whether it was v3 or v4.

> **Fill in:** how the team organized shared UI (a component library, shadcn/ui-style components with `cva`, Rails partials or ViewComponents).

## Interview questions

**Q1. What is utility-first CSS, and what are the pros and cons of Tailwind?**
You style by composing small single-purpose classes in markup instead of writing named CSS rules. Pros: speed, consistent design tokens, no naming, CSS that does not grow, no dead code or specificity wars. Cons: noisy markup, a learning curve, dependence on tooling, and the need for components and tailwind-merge to keep things maintainable.
*Follow-up:* Is it just inline styles? No: it supports responsive and state variants, uses a constrained design scale and produces cacheable CSS.

**Q2. How does Tailwind keep the CSS small, and why does `bg-${color}-500` not work?**
It scans source files as text and generates rules only for class names it finds. A template-literal class is never present as a complete string, so its CSS is never generated. Map values to full class strings, or safelist runtime classes.
*Follow-up:* Why might a class work locally but not in production? In v3, a `content` glob may miss the file, and the class only worked because another file used it.

**Q3. What changed between Tailwind v3 and v4?**
v4 moved configuration into CSS (`@import "tailwindcss";`, `@theme`, `@utility`, `@custom-variant`), detects content automatically, ships dedicated Vite, PostCSS and CLI packages with prefixing built in, exposes theme values as CSS variables, builds in container queries, and renames some utilities (shadow and radius scales, `outline-hidden`, `ring-3`) while changing border and ring defaults.
*Follow-up:* How do you upgrade? Run `npx @tailwindcss/upgrade` on a branch, then review the UI, especially borders, rings and focus styles.

**Q4. How does responsive design work in Tailwind?**
Breakpoint prefixes are mobile-first `min-width` queries: unprefixed classes apply everywhere and `md:` applies from 768px up, so you style the phone first and layer on larger screens. `max-*` variants target below a breakpoint, and container query variants respond to a parent marked `@container`.
*Follow-up:* What does `sm:hidden` do on a phone? Nothing below 640px; it hides the element from 640px up.

**Q5. How do you implement dark mode with a manual toggle?**
Write `dark:` variants (or semantic color tokens). Switch the variant to a class strategy (`@custom-variant dark (&:where(.dark, .dark *));` in v4, `darkMode: "class"` in v3), toggle the `dark` class on `html`, persist the choice, and apply it with an inline script before first paint to avoid a flash.
*Follow-up:* What does `dark:` do without that change? It follows the operating system's `prefers-color-scheme`.

**Q6. How do you avoid repeating long class lists? When is `@apply` appropriate?**
Extract components (React components, Rails partials, ViewComponents) and loops, and define variants with `cva`. Use `@apply` only where you cannot add classes, such as third-party widgets or rich-text content, because overusing it recreates traditional CSS problems.
*Follow-up:* What does v4 need for `@apply` in a CSS Module? An `@reference` to the main stylesheet so it knows your theme.

**Q7. Why do you need tailwind-merge? How do you build a Button with variants?**
Two conflicting utilities (`px-4` and `px-8`) both apply, and the stylesheet order decides the winner, not the class order. `twMerge` keeps the last conflicting class, so callers can override. A typical Button combines `cva` for intent and size variants with a `cn()` helper (`twMerge(clsx(...))`) that merges the caller's `className`.
*Follow-up:* What does `clsx` add? Conditional joining with objects, arrays and falsy values.

**Q8. How do `group` and `peer` work?**
Mark a parent with `group`, and children can react to its state with `group-hover:` or `group-focus:`. Mark an element with `peer`, and later siblings can react with `peer-checked:` or `peer-invalid:`; the peer must come before them in the DOM because CSS sibling selectors only look forward.
*Follow-up:* A use case? A styled toggle built on a visually hidden checkbox that stays keyboard accessible.

**Q9. Arbitrary values vs extending the theme?**
Arbitrary values (`w-[327px]`) are fine for genuine one-offs such as matching a third-party embed. Anything reused or part of the design language belongs in the theme, so it gets a name, stays consistent and changes in one place.
*Follow-up:* How do you reference a CSS variable? `bg-(--brand-color)` in v4, `bg-[var(--brand-color)]` in v3.

**Q10. How would you use Tailwind in a Rails app, and in React Native?**
In Rails, the `tailwindcss-rails` gem wraps the standalone CLI (no Node), `bin/dev` runs the watcher and `assets:precompile` builds for production; classes in ERB, helpers and Stimulus controllers must be visible to the scanner. In React Native, NativeWind maps classes to native styles through `className`; its v4 targets Tailwind v3, and grid and hover do not exist on native.
*Follow-up:* Can ERB conditionals pick classes? Yes, as long as each full class name appears as a literal string.

**Q11. Tailwind vs CSS Modules vs CSS-in-JS?**
Tailwind gives speed and consistent tokens with tiny CSS, at the cost of verbose markup. CSS Modules give scoped plain CSS with no runtime but slower iteration on design tokens. Runtime CSS-in-JS gives dynamic styles but costs runtime performance and fits poorly with Server Components. Choose by team skills, rendering model and design system needs.
*Follow-up:* Which work with React Server Components without extra setup? Tailwind and CSS Modules.

**Q12. After upgrading to v4, borders look darker and focus rings look thinner. Why?**
v4 changed defaults: `border` now uses `currentColor` instead of gray-200, and bare `ring` is 1px in `currentColor` instead of a 3px blue ring. Add explicit colors (`border-zinc-200`) and use `ring-3` where you relied on the old default.
*Follow-up:* What else commonly changes visually? Shadow and radius sizes shifted one step, and `outline-none` became `outline-hidden`.

## Further reading
- [Tailwind CSS documentation](https://tailwindcss.com/docs) and the [v4 upgrade guide](https://tailwindcss.com/docs/upgrade-guide)
- [tailwindcss-rails README](https://github.com/rails/tailwindcss-rails)
- [NativeWind documentation](https://www.nativewind.dev)
- [class-variance-authority](https://cva.style/docs) and [tailwind-merge](https://github.com/dcastil/tailwind-merge)
