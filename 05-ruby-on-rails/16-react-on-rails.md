# React on Rails

> **Tier:** T1 (know cold) · **Related:** [Views, ERB and assets](07-views-erb-forms-assets.md), [Hotwire](08-hotwire-turbo-stimulus.md), [API development](10-api-development.md), [React fundamentals](../06-react/01-fundamentals.md), [React 19 and Server Components](../06-react/09-react-19-and-server-components.md), [React performance](../06-react/06-performance.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md)
> **On your resume:** the Colochousing Rails 7 web app built with React on Rails, ERB and Sidekiq, plus React and TypeScript across Devntech work and the marketplace page migration to the new TypeScript platform.

## Quick revision
- Ways to use React with Rails: **react_on_rails** (ShakaCode: components in ERB, props, SSR), **react-rails** (simpler helper), **Inertia.js** (controllers render React pages, no API), **jsbundling islands** (mount widgets yourself, often via Stimulus), or a **separate SPA + Rails API**.
- React on Rails builds with **Shakapacker** (the community successor to the retired Webpacker): a **client bundle** for browsers and a **server bundle** for server rendering.
- Components are registered in JavaScript (`ReactOnRails.register({ BillSplitter })`) and mounted in ERB with `react_component("BillSplitter", props: {...}, prerender: false)`.
- The helper renders a placeholder element plus the props as JSON in the page; on load the client bundle finds it and renders, or **hydrates** when the HTML was server-rendered.
- Props: build a plain hash in the controller with only the fields the component needs, camelCase the keys (`deep_transform_keys { |k| k.to_s.camelize(:lower) }`), preload associations, and keep them small (they ship inside the HTML).
- **SSR** (`prerender: true`): Rails runs the server bundle in an embedded V8 through ExecJS (mini_racer) and inserts the HTML; React on Rails Pro adds a separate Node renderer. No `window`, `document` or async data fetching during server render.
- **Hydration mismatch:** the first client render differs from the server HTML (dates, time zones, `Math.random`, `typeof window` branches, invalid nesting). Fix by passing values as props, using `useId`, or deferring browser-only UI to `useEffect`.
- Several components sharing state on one page: the Redux store helpers (`redux_store` in the view, `ReactOnRails.registerStoreGenerators` (formerly `registerStore`) and `getStore` in JS) or a shared context root.
- React components calling Rails need the CSRF token: `ReactOnRails.authenticityHeaders()`.
- Turbo Drive swaps pages without full reloads, so components must mount and unmount on Turbo events; recent versions register Turbo handlers with `ReactOnRails.setOptions({ turbo: true })` (check your version's docs).
- i18n: `rake react_on_rails:locale` converts Rails locale YAML for JavaScript; `railsContext.i18nLocale` passes the current locale.
- Debug SSR with `raise_on_prerender_error`, `trace` and `replay_console`; "window is not defined" means browser code ran on the server.
- Performance: per-page bundles, code splitting with dynamic `import()`, bundle analysis, small props, SSR only where it pays (SEO, first paint).

## 1. Options for React in a Rails app

| Option | How React reaches the page | SSR | Data | Fits |
|---|---|---|---|---|
| react_on_rails | `react_component` helper mounts registered components inside ERB | Yes: ExecJS/mini_racer; Pro adds a Node renderer | Props from controllers and views | Mixed ERB + React apps adopting React page by page |
| react-rails | Similar helper: `react_component("Name", props, options)` | Basic, ExecJS | Props from views | Older or simpler apps |
| Inertia.js (inertia_rails) | Controllers `render inertia: "Bills/Index", props: {...}` instead of ERB | Optional, via Node | Controller props per page; client-side navigation without writing an API | SPA feel while keeping Rails routing, controllers and auth |
| jsbundling-rails islands | Your own mount code, often a Stimulus controller | No | `data-` attributes or JSON | A few widgets in a Hotwire app |
| Separate SPA + Rails API | A separate frontend (Vite, Next.js, TanStack Start) | Up to that framework | JSON API, tokens or cookies, CORS | Separate frontend team, several clients on one API |

A lightweight island mount with jsbundling and Stimulus shows the core idea that React on Rails automates:

```tsx
// app/javascript/controllers/react_controller.tsx (esbuild via jsbundling-rails)
import { Controller } from "@hotwired/stimulus";
import { createRoot, type Root } from "react-dom/client";
import { components } from "../components";             // { BillSplitter, ... }

export default class extends Controller {
  static values = { component: String, props: Object };
  declare readonly componentValue: string;
  declare readonly propsValue: Record<string, unknown>;
  private root?: Root;

  connect() {                                             // runs on page load and after Turbo swaps
    const Component = components[this.componentValue];
    this.root = createRoot(this.element);
    this.root.render(<Component {...this.propsValue} />);
  }

  disconnect() {                                          // Turbo removed the element: avoid leaks
    this.root?.unmount();
  }
}
```

## 2. How React on Rails is wired

```text
Gemfile: react_on_rails + shakapacker        package.json: react, react-dom, react-on-rails
config/shakapacker.yml                       where packs live, output paths, dev server
config/initializers/react_on_rails.rb        server bundle name, build commands, SSR options
client bundle (browser)                      registers components for client rendering and hydration
server bundle (server-bundle.js)             registers the same components for ExecJS server rendering
```

```ruby
# config/initializers/react_on_rails.rb (excerpt)
ReactOnRails.configure do |config|
  config.server_bundle_js_file = "server-bundle.js"
  config.build_test_command = "RAILS_ENV=test bin/shakapacker"   # build bundles before the test suite
  config.raise_on_prerender_error = Rails.env.development?        # surface SSR errors loudly
  config.trace = Rails.env.development?                           # log component names and props
end
```

```ts
// Client entry point loaded by the layout (javascript_pack_tag); paths vary by generator version
import ReactOnRails from "react-on-rails";
import BillSplitter from "../bundles/Bills/BillSplitter";

ReactOnRails.register({ BillSplitter });
// For SSR, the server bundle registers the same components.
```

- `bundle exec rails generate react_on_rails:install` sets this up. Recent versions can also register components automatically from conventional folders; follow the docs for your version.
- Shakapacker provides `javascript_pack_tag` / `stylesheet_pack_tag`, `bin/shakapacker` for builds and `bin/shakapacker-dev-server` for hot reloading; `bin/dev` usually runs Rails plus the webpack watchers.
- A component must be registered in every bundle that renders it: client for browser rendering, server for SSR. "Could not find component registered with name" means it is missing from one of them.

## 3. Passing props from controllers and ERB

```ruby
class BillsController < ApplicationController
  def new
    house = current_user.house
    @bill_splitter_props = {
      house_id: house.id,
      currency: "EUR",
      residents: house.residents.order(:first_name).map { |r| { id: r.id, first_name: r.first_name } },
      create_url: house_bills_path(house)
    }.deep_transform_keys { |key| key.to_s.camelize(:lower) }   # firstName, createUrl
  end
end
```

```erb
<%# app/views/bills/new.html.erb %>
<h1>Split a bill</h1>
<%= react_component("BillSplitter", props: @bill_splitter_props, prerender: false) %>
```

```tsx
// app/javascript/bundles/Bills/BillSplitter.tsx
import { useState } from "react";
import ReactOnRails from "react-on-rails";

type Resident = { id: number; firstName: string };
type Props = { houseId: number; currency: string; residents: Resident[]; createUrl: string };

export default function BillSplitter({ residents, currency, createUrl }: Props) {
  const [amount, setAmount] = useState("");
  const share = residents.length ? Number(amount || 0) / residents.length : 0;

  async function save() {
    await fetch(createUrl, {
      method: "POST",
      headers: ReactOnRails.authenticityHeaders({ "Content-Type": "application/json" }),  // CSRF token
      body: JSON.stringify({ bill: { amount_cents: Math.round(Number(amount) * 100) } }),
    });
  }

  return (
    <div>
      <label>
        Amount <input value={amount} onChange={(e) => setAmount(e.target.value)} inputMode="decimal" />
      </label>
      <p>{residents.length} residents pay {share.toFixed(2)} {currency} each</p>
      <button onClick={save}>Save bill</button>
    </div>
  );
}
```

Rules for props:
- **Explicit fields only.** `@house.as_json` or `@residents.to_json` dumps every column, including emails, tokens and internal flags, straight into the page source.
- **Same shape as the API.** Reusing the API serializer (or Jbuilder with `json.key_format! camelize: :lower`) keeps web props and mobile responses consistent.
- **Preload** associations used while building props, or the page fires N+1 queries before React even starts.
- **Keep them small.** Props travel inside the HTML, are parsed on the main thread, and with SSR are sent twice (as HTML and as JSON). Send the first page and IDs; let the component fetch the rest from the API.
- **XSS:** Rails escapes `<`, `>` and `&` in the embedded JSON by default; never build that JSON by hand with `raw`.
### Going deeper: render functions and railsContext

Besides plain components, you can register a **render function**: a function taking two parameters, `(props, railsContext)`, that returns a component. `railsContext` carries request details such as the current location, the locale (`i18nLocale`) and whether the code is running on the server (`serverSide`). Use it to wrap components in providers (a store, a router that needs the URL during SSR) or to read the locale:

```tsx
import { createContext } from "react";

export const LocaleContext = createContext("en");
type Ctx = { i18nLocale: string; serverSide: boolean };

const BillSplitterApp = (props: Props, railsContext: Ctx) => {
  // React on Rails treats a registered function with two parameters as a render function
  return () => (
    <LocaleContext.Provider value={railsContext.i18nLocale}>
      <BillSplitter {...props} />
    </LocaleContext.Provider>
  );
};

ReactOnRails.register({ BillSplitterApp });
```

## 4. Server-side rendering

With `prerender: true`, React on Rails renders the component to an HTML string on the server and inserts it into the page; the browser shows content before JavaScript loads, then the client bundle hydrates it.

```erb
<% cache [@listing, "listing-card-ssr"] do %>                       <%# SSR output is plain HTML, so it caches %>
  <%= react_component("ListingCard", props: @listing_props, prerender: true) %>
<% end %>
```

| | ExecJS (open source) | Node renderer (React on Rails Pro) |
|---|---|---|
| Where it runs | Embedded V8 inside the Ruby process via mini_racer, with a pool of contexts | A separate pool of Node processes Rails calls over HTTP |
| Environment | Bare JavaScript: no `window`, no Node APIs, no real timers or `fetch` | Full Node: more libraries work |
| Performance | Fine for small components; adds CPU to Rails processes | Faster at scale, keeps CPU off Puma |
| Extras | Basic | Caching, streaming and newer React rendering features in recent Pro versions (check the docs) |

Constraints during server render:
- No browser APIs (`window`, `document`, `localStorage`) at module load or during render; effects (`useEffect`) do not run on the server.
- No async data fetching inside the server render with ExecJS; pass the data in as props.
- The server bundle must be rebuilt when components change, or SSR serves stale markup.

When SSR pays off: public, SEO-relevant pages (listings, marketing, a marketplace) and content-heavy pages where first paint matters. When it does not: dashboards behind login, highly interactive tools, anything where the server CPU cost and complexity buy nothing.

## 5. Hydration and hydration mismatches

Hydration attaches React's event handlers and state to server-rendered HTML instead of recreating it (`hydrateRoot` in React 18+; React on Rails chooses it automatically when HTML is present). React expects the first client render to produce the same markup as the server.

| Cause | Example | Fix |
|---|---|---|
| Time and time zones | `new Date().toLocaleString()` on a UTC server vs a Brussels browser | Pass formatted strings or timestamps as props; format after mount |
| Randomness and IDs | `Math.random()` keys, generated IDs | Stable keys from data; `useId` for element IDs |
| Environment checks in render | `typeof window !== "undefined" ? <Map /> : null` | Render the browser-only part after mount (`useEffect` + state) |
| Different data | Server and client read different props or globals | One source of truth: the props JSON |
| Invalid HTML nesting | `<div>` inside `<p>`, `<a>` inside `<a>` | Fix the markup; the browser rewrites invalid HTML before React hydrates |
| Locale differences | Number or date formatting with different defaults | Pass the locale from `railsContext` and format explicitly |

Symptoms range from console warnings to React discarding the server HTML and re-rendering on the client (losing the SSR benefit) or event handlers attaching to the wrong nodes. React 19 reports mismatches with clearer diffs. `suppressHydrationWarning` is acceptable for a single unavoidable text difference such as a timestamp, not as a general fix.

## 6. Redux and shared state between components

Several `react_component` calls on one page are separate React roots: they do not share context. Options:
- **React on Rails store helpers:** render the store's initial state once with `<%= redux_store("HouseStore", props: @house_state) %>`, register a store generator with `ReactOnRails.registerStoreGenerators({ HouseStore })` (`registerStore` in older versions), and have each component read it with `ReactOnRails.getStore("HouseStore")`.
- **One root:** render a single top-level component that contains the others and shares a context or a lightweight store.
- **The server as the source of truth:** TanStack Query or plain fetches against the API, with each component reading what it needs.

Many apps reach for Redux by habit; for a few widgets on server-rendered pages, local state plus the server is usually enough (see [state management](../06-react/03-state-management.md)).

## 7. Turbo compatibility

With Turbo Drive, navigating replaces the `<body>` without a full page load, so `DOMContentLoaded` does not fire again. Components must be mounted after each Turbo visit and unmounted before the page is swapped, or you get missing components and memory leaks. React on Rails historically supported Turbolinks, and recent versions register Turbo handlers when you call `ReactOnRails.setOptions({ turbo: true })` before registering components; confirm the details for your version and test the back button, Turbo's page cache and Turbo Frames, which can insert components outside a full visit. When in doubt, disable Turbo on React-heavy pages (`data-turbo="false"`) or mount through a Stimulus controller as in section 1.

## 8. Internationalization

- `rake react_on_rails:locale` reads `config/locales/*.yml` and writes translation files for JavaScript (JSON by default; configured with `config.i18n_dir` and `config.i18n_yml_dir`) for use with a library such as react-intl, so ERB and React share one set of translations.
- The current locale reaches components through `railsContext.i18nLocale`; set `I18n.locale` per request on the Rails side as usual.
- Format numbers, currencies and dates with `Intl` using that locale, both on the server and the client, to avoid hydration mismatches.

## 9. Mixed ERB and React apps: deciding page by page

| Page needs | Choose |
|---|---|
| Mostly content, forms with simple validation, CRUD screens | ERB (plus Turbo and Stimulus) |
| Rich client state: live calculations, drag and drop, multi-step wizards, charts, maps | React component |
| SEO with interactivity | React with SSR, or ERB with small islands |
| A React design system already exists | React for shared components, ERB for the layout |
| Logic shared with the React Native app | React with TypeScript, sharing types and validation where possible |

Keep the layout, navigation and authentication in Rails; mount React where it earns its weight. Avoid duplicating the same UI in ERB and React, and keep business rules on the server so both stay thin.

## 10. Debugging SSR errors

1. **Make errors visible:** `raise_on_prerender_error` (development and test) raises with the JavaScript stack instead of silently falling back to client rendering; `trace: true` logs the component and props; `replay_console: true` replays server-side `console.log` output into the browser console.
2. **"window is not defined" / "document is not defined":** browser code ran during server render, often in a third-party library imported at module level. Move it into `useEffect`, lazy-load it on the client only, or guard the import.
3. **"Could not find component registered with name":** the component is not registered in the server bundle (or the client bundle).
4. **Stale bundle:** markup does not match your code because the server bundle was not rebuilt; rebuild with `bin/shakapacker` and restart if the renderer caches the bundle.
5. **Missing APIs in ExecJS:** `fetch`, `setTimeout` or Node modules are not available in mini_racer; pass data as props or move to the Node renderer.
6. **Isolate:** render the component in a Jest test or Storybook to tell React bugs apart from SSR and bundling problems.

## 11. Testing React on Rails pages

Test at two levels: the component in isolation with Jest and React Testing Library, and the full page (Rails view, props, bundle, browser) with a system spec.

```tsx
// app/javascript/bundles/Bills/BillSplitter.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import BillSplitter from "./BillSplitter";

test("shows each resident's share", async () => {
  render(
    <BillSplitter houseId={1} currency="EUR" createUrl="/houses/1/bills"
      residents={[{ id: 1, firstName: "Sara" }, { id: 2, firstName: "Tom" }]} />
  );
  await userEvent.type(screen.getByLabelText("Amount"), "60");
  expect(screen.getByText("2 residents pay 30.00 EUR each")).toBeInTheDocument();
});
```

```ruby
# spec/system/split_bill_spec.rb: exercises the controller props, the bundle and the browser together
it "splits a bill", js: true do
  sign_in create(:resident)
  visit new_bill_path
  fill_in "Amount", with: "60"
  expect(page).to have_content("residents pay")
end
```

Make sure bundles are compiled before specs run (`config.build_test_command`, wired into RSpec with React on Rails' test helper), or system specs fail with missing packs. More on component testing in [React testing](../06-react/10-testing.md).

## 12. Performance

- **Per-page bundles:** load the bills bundle only on bill pages instead of one giant pack everywhere.
- **Code splitting:** dynamic `import()` with `React.lazy` for heavy client-only pieces (charts, editors); combining code splitting with SSR needs extra setup (Pro documents loadable-components support).
- **Measure bundles:** webpack-bundle-analyzer; replace heavy dependencies (moment with `Intl` or date-fns), import only what you use.
- **Small props and preloaded queries** (section 3); cache SSR output with fragment caching.
- **SSR capacity:** ExecJS rendering uses CPU in your Puma processes; profile it and render on the server only where it pays.
- General React techniques (memoization, list virtualization, avoiding needless re-renders) are in [React performance](../06-react/06-performance.md).

## 13. How this maps to the Colochousing app

Your resume lists the Colochousing Rails 7 web app and REST API with React on Rails, ERB and Sidekiq, plus a migration of the marketplace page onto the new TypeScript platform (TanStack Start). That gives you three angles: how React components live inside ERB pages, how the same Rails backend also serves the resident mobile app through the API, and how a React on Rails page moves to a separate frontend (props built in a controller become a loader or server function calling an API, and Rails-only concerns such as CSRF and `railsContext` disappear).

> **Fill in:** which pages or widgets are React and which are ERB, and why the team chose React for them.

> **Fill in:** whether the app uses `prerender: true` anywhere, and how props are built (controller hashes, Jbuilder, serializers shared with the API).

> **Fill in:** how state is shared between components (Redux store helpers, context, fetching from the API), and how React components are tested.

> **Fill in:** what moved during the marketplace page migration, what replaced the React on Rails props, and how you verified the new page matched the old one.

## Common pitfalls
- Passing `as_json` or `to_json` of whole records as props, leaking private columns into the HTML.
- Huge props (entire collections) that bloat pages and slow hydration.
- Forgetting to register a component in the server bundle, or a stale server bundle in production.
- Browser APIs at module level in components that are server-rendered.
- Formatting dates and numbers differently on server and client, causing hydration mismatches.
- Components not unmounted on Turbo navigation, leaking memory and duplicating listeners.
- `fetch` POSTs from React without the CSRF token, failing with `InvalidAuthenticityToken`.
- Turning on SSR everywhere "for speed" and paying CPU cost on authenticated pages that need no SEO.
- Duplicating validation or pricing rules in React and Rails instead of keeping them on the server.

## How to talk about this in an interview
- Explain the architecture in one breath: Rails renders the page and layout, `react_component` drops in a placeholder with JSON props, Shakapacker's client bundle mounts or hydrates the component, and optional SSR runs the server bundle in V8.
- Show judgment about when React is worth it on a Rails page versus Turbo and Stimulus, using your Visnext Hotwire training as the contrast.
- Connect to the Coloc resident app: the same Rails models and serializers can feed both React props on the web and the REST API for React Native.
- Use the migration as a story about moving from server-injected props to a separate TypeScript frontend, and be clear about your part in it.

> **Fill in:** the React on Rails component you know best: what it does, where its props come from, and how you would explain its data flow in two sentences.

## Interview questions

**Q1. What are the main ways to use React in a Rails app?**
react_on_rails (components mounted in ERB with props and optional SSR), react-rails (a simpler helper), Inertia.js (controllers render React page components with client-side navigation and no API), hand-mounted islands with jsbundling and Stimulus, or a separate SPA consuming a Rails API. The choice depends on how much of the UI is React, SEO needs and team structure.
*Follow-up:* Which suits gradual adoption in an ERB app? react_on_rails or islands, page by page.

**Q2. How does React on Rails work end to end?**
Shakapacker builds a client bundle and a server bundle in which components are registered with `ReactOnRails.register`. In a view, `react_component("Name", props:)` renders a placeholder and the props as JSON; on page load the client bundle finds the placeholder and renders the component, or hydrates it when `prerender: true` produced HTML on the server.
*Follow-up:* What happens if a component is missing from the server bundle? SSR fails with a "could not find component registered" error.

**Q3. How does React on Rails pass props, and what do you watch out for?**
The controller or view builds a hash, the helper serializes it to JSON inside the page, and the component receives it as props. Include only needed fields, camelCase keys for JavaScript, preload associations, keep payloads small, and never pass `as_json` of whole records, which leaks private columns.
*Follow-up:* How do you keep web props and the mobile API consistent? Build both with the same serializer.

**Q4. How does server-side rendering work in React on Rails?**
With `prerender: true`, Rails runs the server bundle in an embedded V8 through ExecJS (mini_racer), calls the registered component with the props and inserts the resulting HTML; the client then hydrates it. React on Rails Pro can render in a separate Node process pool instead, which supports more libraries and offloads CPU from Rails.
*Follow-up:* What can't run during ExecJS SSR? Browser APIs, effects, timers and async data fetching.

**Q5. What is hydration, and what causes hydration mismatches?**
Hydration attaches React to server-rendered HTML instead of rebuilding it, so the first client render must match the server markup. Mismatches come from dates and time zones, random values, `typeof window` branches in render, differing data and invalid HTML nesting; fix them by passing values as props, using `useId` and deferring browser-only UI to effects.
*Follow-up:* What happens on a mismatch? Warnings, or React throwing away the server HTML and rendering on the client.

**Q6. When would you not use SSR?**
For authenticated dashboards and interactive tools where SEO does not matter and first paint is acceptable without it; SSR adds server CPU, bundle constraints and hydration risks. Use it for public, content-heavy or SEO pages.
*Follow-up:* How can you reduce SSR cost? Cache the rendered HTML with fragment caching and use the Node renderer at scale.

**Q7. What is Shakapacker, and how does it relate to Webpacker?**
Webpacker wrapped webpack for Rails and was retired by the Rails team after Rails 7 moved to import maps and jsbundling. ShakaCode continued it as Shakapacker, which React on Rails uses to build its client and server bundles, providing pack tag helpers, a config file and a dev server.
*Follow-up:* How do you spot each in a legacy app? `config/webpacker.yml` vs `config/shakapacker.yml`, and the binstub names.

**Q8. How does a React component on an ERB page send data to Rails?**
Through `fetch` to a Rails endpoint (often the JSON API) with the CSRF token in the headers, which `ReactOnRails.authenticityHeaders()` provides because the request carries the session cookie; or by submitting a regular form that Rails handles.
*Follow-up:* Why is the token needed when the API skips CSRF? Here the request is authenticated by the session cookie, so CSRF protection applies.

**Q9. How do several React components on one page share state?**
They are separate roots, so they share nothing by default. Use React on Rails' Redux store helpers (`redux_store` plus `registerStoreGenerators`, formerly `registerStore`, and `getStore`), render a single parent component that provides context, or treat the server as the source of truth and fetch with a query library.
*Follow-up:* Is Redux required? No; for a few widgets, local state and the server are usually enough.

**Q10. Does React on Rails work with Turbo?**
Yes with care: Turbo Drive swaps the body without a full reload, so components must mount after each visit and unmount before the swap. Recent React on Rails versions register Turbo handlers via `ReactOnRails.setOptions({ turbo: true })`; verify against the docs for your version, test caching and back navigation, or disable Turbo on React-heavy pages.
*Follow-up:* What about Turbo Frames? They insert HTML outside full visits, so components inside them need their own mounting (for example via Stimulus).

**Q11. How do you handle translations in a React on Rails app?**
Keep translations in Rails YAML, convert them with `rake react_on_rails:locale` for a JavaScript i18n library, pass the current locale via `railsContext`, and format with `Intl` in that locale on server and client.
*Follow-up:* Why format consistently on both sides? Different locale defaults cause hydration mismatches.

**Q12. How do you decide between ERB with Hotwire and React for a page?**
ERB and Turbo for content and standard CRUD forms; React for rich client-side state such as live calculations, drag and drop, wizards and charts, or when a React component library already exists. Keep layout, routing and business rules in Rails either way.
*Follow-up:* What is the cost of React on a page? A bundle to download and maintain, props or API endpoints to design, and two rendering stacks to test.

**Q13. You see "window is not defined" during server rendering. What do you do?**
Find the browser-only code that runs at import or render time (often a third-party library), move it into `useEffect`, load it lazily on the client, or skip SSR for that component; turn on `raise_on_prerender_error` and `trace` to see which component and props failed.
*Follow-up:* And "Could not find component registered with name"? Register it in the server bundle and rebuild.

**Q14. How do you keep React bundles small in a Rails app?**
Per-page bundles, dynamic `import()` with `React.lazy` for heavy client-only parts, bundle analysis to remove or replace heavy dependencies, tree-shakable imports and fingerprinted, cached assets. Keep props small too, since they are part of the page weight.
*Follow-up:* Why is code splitting harder with SSR? The server render is synchronous, so lazy chunks need extra tooling.

**Q15. React on Rails or Inertia.js?**
React on Rails embeds components inside ERB pages, which suits mixed apps and gradual adoption with optional SSR. Inertia replaces ERB views entirely with React page components while keeping Rails controllers and routing, giving an SPA feel without building an API. Choose by how much of the UI should be React.
*Follow-up:* Which needs an API for the mobile app anyway? Both: neither replaces a JSON API for native clients.

**Q16. How would you migrate a React on Rails page to a separate TypeScript frontend?**
Expose the data the controller used to build props through an API (or sync it to the new platform), port the component nearly as is, replace server-injected props with a loader or server function, handle authentication and CSRF the new way, run both versions behind a route switch or feature flag, compare output, then remove the Rails page.
*Follow-up:* What is the biggest risk? Hidden behavior in the old controller and views (permissions, formatting) that the new page silently drops.

## Further reading
- [React on Rails](https://github.com/shakacode/react_on_rails)
- [Shakapacker](https://github.com/shakacode/shakapacker)
- [hydrateRoot](https://react.dev/reference/react-dom/client/hydrateRoot)
