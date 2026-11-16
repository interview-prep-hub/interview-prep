# Views: ERB, Forms, Helpers and the Asset Pipeline

> **Tier:** T1 (know cold) · **Related:** [Routing and controllers](02-routing-and-controllers.md), [Hotwire](08-hotwire-turbo-stimulus.md), [React on Rails](16-react-on-rails.md), [Web security (OWASP)](../04-web-fundamentals/07-web-security-owasp.md), [HTML and CSS](../04-web-fundamentals/02-html-css.md), [Tailwind CSS](../04-web-fundamentals/03-tailwind-css.md)
> **On your resume:** ERB views in the Colochousing Rails 7 app (alongside React on Rails components), server-rendered pages in legacy Rails monoliths at Devntech, and the Visnext training on MVC and Hotwire.

## Quick revision
- `<% %>` runs Ruby, `<%= %>` outputs **HTML-escaped** text, `<%# %>` is a comment, `<%== %>` outputs raw. `-%>` trims the trailing newline.
- Escaping is the default XSS defense. `html_safe` and `raw` only **mark** a string as safe (no cleaning); `sanitize` actually strips disallowed tags and attributes.
- Layout (`app/views/layouts/application.html.erb`) wraps every view at `yield`; `content_for :title` / `yield :title` fill named slots. Pick layouts with `layout "admin"` or `render layout: false`.
- Partials start with `_`: `render "listing", listing: @listing`. `render @listings` renders `_listing.html.erb` for each record (returns nil when empty). Pass locals instead of reading instance variables.
- Helpers are modules in `app/helpers`, available in every view by default; build markup with `tag.*`, `safe_join` and `class_names`, never by concatenating strings and calling `html_safe`.
- `form_with model: @listing` posts to `listings_path` for a new record and PATCHes `listing_path(@listing)` for a persisted one; field names become `listing[title]`.
- Show errors from `@listing.errors.full_messages`; invalid fields get wrapped in `field_with_errors`.
- CSRF: `form_with` adds a hidden `authenticity_token`; `csrf_meta_tags` exposes it for JavaScript; `protect_from_forgery` verifies it on non-GET requests.
- Nested forms: `form.fields_for :bill_shares` + `accepts_nested_attributes_for` in the model.
- I18n: `t(".title")` (lazy lookup by view path), `l(time, format: :short)`, `number_to_currency`, `number_with_delimiter`, `time_ago_in_words`, `pluralize`.
- Asset history: **Sprockets** (3.1 to 6) -> **Webpacker** (5.1 to 6, now retired) -> Rails 7: **import maps** by default or **jsbundling / cssbundling** (esbuild, Tailwind) -> Rails 8: **Propshaft** by default. **Shakapacker** (Webpacker's community successor) powers React on Rails.
- ViewComponent: a Ruby class + template per component, with explicit arguments and unit tests.
- React inside ERB: `react_component("BillSplitter", props: {...})` from React on Rails mounts a component on a server-rendered page.

## 1. ERB tags, escaping and XSS

ERB (Embedded Ruby) templates mix HTML and Ruby. Rails compiles each template into a Ruby method once and reuses it.

```erb
<% @listings.each do |listing| %>            <%# runs Ruby, prints nothing %>
  <h2><%= listing.title %></h2>             <%# prints the value, HTML-escaped %>
<% end %>
<%== trusted_html_from_our_own_code %>        <%# prints raw: same as raw(...) %>
```

**Cross-site scripting (XSS)** means an attacker's input runs as script in another user's browser. If a resident names an event `<script>steal()</script>`, `<%= event.name %>` prints `&lt;script&gt;...`, which is harmless text. Rails tracks this with `ActiveSupport::SafeBuffer`: every string is unsafe unless marked otherwise.

| Method | What it does | Safe with user input? |
|---|---|---|
| `<%= value %>` | Escapes `<`, `>`, `&`, quotes | Yes (the default) |
| `value.html_safe` | Marks the string as safe; changes nothing in it | **No** |
| `raw(value)` / `<%== %>` | Same as `html_safe` | **No** |
| `sanitize(value, tags: %w[b i a], attributes: %w[href])` | Removes tags and attributes not on the allowlist | Yes, for rich text you must render |
| `strip_tags(value)` | Removes all tags | Yes |

Rules: never call `html_safe` or `raw` on anything containing user data; build HTML with helpers (`tag.span`, `link_to`, `safe_join`), which escape their contents; validate user-supplied URLs, because `link_to "site", user.website` will happily render a `javascript:` URL. Escaping is context-specific: for data inside a `<script>` tag, pass it through `to_json` (Rails escapes HTML characters in JSON by default) or a `data-` attribute, not string interpolation. More in [web security](../04-web-fundamentals/07-web-security-owasp.md).

## 2. Layouts, yield and content_for

The layout is the page shell shared by every view:

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : "Shop" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application" %>
    <%= javascript_importmap_tags %>   <%# or javascript_include_tag "application", defer: true %>
    <%= yield :head %>
  </head>
  <body>
    <%= render "shared/flash" %>
    <%= yield %>                       <%# the action's template goes here %>
  </body>
</html>
```

```erb
<%# app/views/listings/index.html.erb %>
<% content_for :title, "Your listings" %>
<% content_for :head do %>
  <meta name="robots" content="noindex">
<% end %>
<h1>Listings</h1>
```

Choosing layouts: by default Rails uses `layouts/<controller>.html.erb` if it exists, otherwise `layouts/application.html.erb`. Override with `layout "admin"` in a controller (optionally `only:` / `except:`), a method name for dynamic choice, or `render layout: false` for a bare response.

## 3. Partials and collections

A partial is a reusable template whose file name starts with an underscore:

```erb
<%# app/views/listings/_listing.html.erb %>
<%# locals: (listing:, show_store: false) -%>
<div id="<%= dom_id(listing) %>" class="listing">
  <h2><%= listing.product.title %></h2>
  <p><%= number_to_currency(listing.price_cents / 100.0) %></p>
  <% if show_store %><p><%= listing.store.name %></p><% end %>
</div>
```

```erb
<%= render "listing", listing: @listing %>                       <%# short form %>
<%= render partial: "listings/listing", locals: { listing: @listing, show_store: true } %>
<%= render @listings %>                                           <%# collection via to_partial_path %>
<%= render(@listings) || tag.p("No listings yet.") %>             <%# render returns nil for empty %>
<%= render partial: "listing", collection: @listings, cached: true %>  <%# fragment-cached collection %>
```

- `render @listings` calls each record's `to_partial_path` (`"listings/listing"`) and renders them in one pass, which is faster than looping and calling `render` yourself. Inside, a `listing_counter` local holds the index.
- The magic comment `<%# locals: (listing:, show_store: false) -%>` (Rails 7.1) declares **strict locals**: missing or unknown locals raise, so the partial's inputs are explicit.
- `dom_id(listing)` returns `listing_42`, which Turbo Streams use to target elements.
- Prefer locals to instance variables inside partials, so the partial works from any view, from a Turbo Stream or from a mailer. Remember the N+1 risk: `listing.product` and `listing.store` in a collection partial need `includes(:product, :store)` in the controller.

## 4. Helpers

Helpers are plain modules in `app/helpers` for small pieces of presentation logic. By default every helper is available in every view, so give methods distinct names.

```ruby
# app/helpers/listings_helper.rb
module ListingsHelper
  def price_tag(listing)
    tag.span(number_to_currency(listing.price_cents / 100.0),
             class: class_names("price", "price--sale": listing.on_sale?))
  end

  def condition_badges(listings)
    safe_join(listings.map { |l| tag.span(l.condition.humanize, class: "badge") }, " ")
  end
end
```

Use a helper for a short snippet with logic (formatting, a badge, a conditional class) and a partial for a larger chunk of markup. When views pile up model-specific formatting, a presenter or decorator object keeps both models and views clean. Built-in helpers worth knowing: `link_to`, `button_to` (a mini form, which is how you send DELETE without JavaScript), `image_tag`, `time_tag`, `truncate`, `simple_format`, `current_page?`, `dom_id`.

### Going deeper: fragment caching in views

```erb
<% cache listing do %>                                <%# key: model name, id, updated_at, template digest %>
  <%= render "listings/listing", listing: listing %>
<% end %>

<% cache [current_store, "listing_table"] do %>       <%# outer fragment wrapping inner ones %>
  <%= render partial: "listings/listing", collection: @listings, cached: true %>
<% end %>
```

The cache key changes whenever the record's `updated_at` changes or the template changes (Rails adds a digest of the template), so you never expire fragments by hand. Nested ("Russian doll") caches need `belongs_to ..., touch: true` on the children, so a changed child also refreshes its parent's key. Details in [caching](14-caching-performance-security.md).

## 5. Forms with form_with

`form_with` (Rails 5.1) unified the older `form_for` (model forms) and `form_tag` (URL forms), which you will still meet in legacy views.

```erb
<%= form_with model: @listing do |form| %>
  <% if @listing.errors.any? %>
    <div class="errors" role="alert">
      <h2><%= pluralize(@listing.errors.count, "error") %> prevented saving:</h2>
      <ul>
        <% @listing.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <%= form.label :product_id %>
  <%= form.collection_select :product_id, Product.order(:title), :id, :title, prompt: "Choose a product" %>

  <%= form.label :condition %>
  <%= form.select :condition, Listing.conditions.keys.map { |c| [c.humanize, c] } %>

  <%= form.label :price_cents %>
  <%= form.number_field :price_cents, min: 0 %>

  <%= form.check_box :active %> <%= form.label :active %>
  <%= form.file_field :photo, accept: "image/*" %>   <%# switches the form to multipart %>
  <%= form.hidden_field :lock_version %>
  <%= form.submit %>                                  <%# "Create Listing" or "Update Listing" %>
<% end %>
```

How it decides where to submit:

| `@listing` state | Generated form | Route |
|---|---|---|
| New record | `method="post"` | `POST /listings` (`create`) |
| Persisted | `method="post"` + hidden `_method=patch` | `PATCH /listings/:id` (`update`) |
| `model: [@store, @listing]` | Nested URL | `POST /stores/:store_id/listings` |
| `url: search_path, method: :get` | No model; field names are not namespaced | Any URL |

- Field names follow the model (`listing[price]`), which is why the controller reads `params.require(:listing)`.
- Fields with errors are wrapped in `<div class="field_with_errors">` by default; style it or change `config.action_view.field_error_proc`.
- The failed `create` must `render :new, status: :unprocessable_entity` so the form comes back with errors (see [controllers](02-routing-and-controllers.md)).
- History note: in Rails 5.1 to 6.0, `form_with` submitted via Ajax by default (`local: true` turned that off). Since 6.1 forms are plain by default, and in Rails 7+ Turbo intercepts them.
- File uploads usually go through Active Storage (`has_one_attached :photo`); see [Active Storage](15-actioncable-mailer-activestorage.md).

**Nested forms** pair `fields_for` with `accepts_nested_attributes_for` in the model (see [associations](04-activerecord-associations.md)):

```erb
<%= form_with model: @bill do |form| %>
  <%= form.text_field :title %>
  <%= form.number_field :amount_cents %>
  <%= form.fields_for :bill_shares do |share_form| %>     <%# once per share in @bill.bill_shares %>
    <%= share_form.hidden_field :resident_id %>
    <%= share_form.number_field :amount_cents %>
    <%= share_form.check_box :_destroy %> Remove
  <% end %>
  <%= form.submit "Save bill" %>
<% end %>
```

Build blank children in the controller (`@bill.bill_shares.build`) for new rows; adding rows dynamically is a small Stimulus controller that clones a `<template>` element.

## 6. CSRF protection

**Cross-site request forgery** tricks a logged-in user's browser into submitting a request to your site from another site; the browser sends the session cookie automatically. Rails defends with a per-session token:

1. `form_with` embeds a hidden `authenticity_token` field; `csrf_meta_tags` puts the token in the page head for JavaScript (Turbo and Rails' JS helpers send it as the `X-CSRF-Token` header).
2. `ActionController::Base` has forgery protection on by default and checks the token on every non-GET request, raising `ActionController::InvalidAuthenticityToken` when it is missing or wrong.
3. GET requests are not checked, so they must never change data. That is why "delete" links use `button_to` or `data-turbo-method="delete"`, not a plain GET link.

Token-authenticated JSON APIs (`ActionController::API`) do not use cookies for auth, so they skip CSRF checks. `SameSite=Lax` cookies add a second layer of defense.

## 7. I18n, numbers and dates

```yaml
# config/locales/en.yml
en:
  listings:
    index:
      title: "Your listings"
      count:
        one: "1 listing"
        other: "%{count} listings"
  activerecord:
    attributes:
      listing:
        price_cents: "Price"          # used by labels and error messages
```

```erb
<h1><%= t(".title") %></h1>                         <%# lazy lookup: listings.index.title %>
<p><%= t(".count", count: @listings.size) %></p>    <%# pluralization rules per locale %>
<p><%= l(order.created_at, format: :short) %></p>   <%# localized date and time %>
<p><%= number_to_currency(12.5, unit: "€", format: "%n %u") %></p>  <%# "12.50 €" %>
<p><%= number_with_delimiter(8_000_000) %></p>       <%# "8,000,000" %>
<p><%= number_to_human(8_000_000) %></p>             <%# "8 Million" %>
<p><%= time_ago_in_words(order.created_at) %> ago</p>
```

Set `config.i18n.default_locale` and `available_locales`, and switch per request with an `around_action` that calls `I18n.with_locale(locale) { yield }`, taking the locale from the user's settings or the URL, never trusting it blindly. Run `i18n-tasks` (a gem) or tests to catch missing keys.

## 8. The asset pipeline: from Sprockets to Propshaft

| Era | Tool | What it does | Today |
|---|---|---|---|
| Rails 3.1 to 6 | **Sprockets** | Concatenates, minifies and fingerprints CSS, JS and images; `//= require` directives; Sass and CoffeeScript preprocessing | Still runs countless legacy apps |
| Rails 5.1 to 6 | **Webpacker** | Wrapped webpack so Rails apps could use npm packages, ES modules and React | Retired by the Rails team; continued by ShakaCode as **Shakapacker** |
| Rails 7.0 | **jsbundling-rails / cssbundling-rails** | Run a real bundler (esbuild, rollup or webpack) and CSS tool (Tailwind, Sass, PostCSS) through npm scripts into `app/assets/builds` | The choice when you need JSX, TypeScript or npm-heavy code (`rails new -j esbuild --css tailwind`) |
| Rails 7.0 default | **importmap-rails** | No bundling and no Node: the browser loads ES modules directly, and an import map resolves names like `@hotwired/turbo-rails` to fingerprinted URLs; pins live in `config/importmap.rb` | Default for Hotwire apps; no JSX or TypeScript transpiling |
| Rails 8.0 default | **Propshaft** | A minimal pipeline: load paths and digest fingerprinting only; transpiling and bundling are left to a bundler or skipped entirely | Default for new apps |
| React on Rails | **Shakapacker** | webpack integration with `bin/shakapacker`, packs and a dev server | What React on Rails builds with |

What stays constant: `stylesheet_link_tag`, `javascript_include_tag` and `image_tag "logo.png"` emit fingerprinted URLs (`/assets/logo-3f2a...png`) so browsers can cache files forever; `bin/rails assets:precompile` builds everything for production; `bin/dev` runs the web server plus JS and CSS watchers from `Procfile.dev`. The `tailwindcss-rails` gem compiles Tailwind with a standalone binary, without Node.

How to recognize a legacy app's setup: `app/assets/config/manifest.js` and `//= require` lines mean Sprockets; `app/javascript/packs` and `config/webpacker.yml` mean Webpacker (or `config/shakapacker.yml` for Shakapacker); `config/importmap.rb` means import maps; `package.json` scripts named `build` and `build:css` mean jsbundling and cssbundling.

## 9. ViewComponent and React inside ERB

**ViewComponent** (a gem from GitHub) turns a partial into a Ruby class with a template, explicit arguments and fast unit tests:

```ruby
# app/components/price_component.rb
class PriceComponent < ViewComponent::Base
  def initialize(cents:)
    @cents = cents
  end
end
```

```erb
<%# app/components/price_component.html.erb %>
<span class="price"><%= number_to_currency(@cents / 100.0) %></span>

<%# usage in any view %>
<%= render PriceComponent.new(cents: listing.price_cents) %>
```

Choose it when a design system has many reusable UI pieces with logic; plain partials remain fine for page-specific markup.

**Mounting React components in ERB.** Your resume lists both ERB and React on Rails for the Colochousing Rails 7 app. A common split is server-rendered ERB for most pages and React only where rich interactivity pays off. With React on Rails, a registered component is mounted from a view:

```erb
<%# app/views/bills/new.html.erb %>
<h1>Split a bill</h1>
<%= react_component("BillSplitter",
      props: { houseId: @house.id, residents: @residents.as_json(only: %i[id first_name]) },
      prerender: false) %>
```

The helper renders a placeholder element plus the JSON props, and the client bundle (built by Shakapacker) hydrates or renders the component there; `prerender: true` renders it on the server first. Serialize props deliberately: use `only:` to avoid leaking attributes and preload associations to avoid N+1s while building them. Registration, server rendering and Shakapacker setup are covered in [React on Rails](16-react-on-rails.md).

## Common pitfalls
- Calling `html_safe` or `raw` on strings that include user input, or building HTML by string concatenation in helpers.
- Database queries in views and partials (`Category.all` inside a loop, `listing.product` without `includes`), which hide N+1s.
- Partials that read instance variables, so they break when rendered from another controller, a Turbo Stream or a mailer.
- Forgetting `status: :unprocessable_entity` when re-rendering a form, so Turbo ignores the response.
- Using `flash` instead of `flash.now` when rendering, so the message shows twice.
- Links that change data over GET, which bypass CSRF protection and get triggered by prefetchers and crawlers.
- Hard-coded user-facing strings in an app that needs translations.
- Mixing asset approaches (Sprockets directives, Webpacker packs and import maps) without knowing which one actually serves a file; check the layout tags first.
- Serializing whole models into `react_component` props, leaking attributes and triggering N+1s.

## How to talk about this in an interview
- Colochousing: your resume lists ERB and React on Rails together, so be ready to explain how a server-rendered ERB page mounts a React component with props, and why you would not rewrite every page in React.
- Legacy monoliths at Devntech: describe how you identify the asset setup (Sprockets, Webpacker, jsbundling or import maps) before touching JavaScript, and how you keep views thin with partials, helpers and preloaded data.
- Visnext training: forms with `form_with`, validation errors and Turbo-friendly status codes are the bridge to the Hotwire topics.
- Security is a good differentiator: mention automatic escaping, when `sanitize` is appropriate and how CSRF tokens work.

> **Fill in:** which asset setup the Colochousing app uses (Shakapacker with React on Rails, plus anything else) and one ERB page that mounts a React component.

> **Fill in:** a form you built or fixed (fields, nested attributes, error display) and how you tested it.

## Interview questions

**Q1. What is the difference between `<% %>`, `<%= %>` and `<%# %>` in ERB?**
`<% %>` executes Ruby without output (loops, conditionals), `<%= %>` evaluates and outputs the HTML-escaped result, and `<%# %>` is a comment. `<%== %>` outputs without escaping, like `raw`.
*Follow-up:* What does `-%>` do? Trims the newline after the tag.

**Q2. How does Rails protect views from XSS?**
Everything printed with `<%= %>` is HTML-escaped unless it is a `SafeBuffer` marked safe. Helpers like `link_to` and `tag` escape their content. Developers break this by calling `html_safe` or `raw` on user input; use `sanitize` with an allowlist when you must render user HTML.
*Follow-up:* Does `html_safe` clean the string? No, it only marks it as trusted.

**Q3. What do `yield` and `content_for` do in layouts?**
`yield` without arguments inserts the action's template into the layout; `content_for :name` in a view stores content that the layout prints with `yield :name`, for titles, extra head tags or sidebars. `content_for?(:name)` checks whether it was set.
*Follow-up:* How do you skip the layout? `render layout: false`, or `layout false` in the controller.

**Q4. Why pass locals to partials instead of using instance variables?**
Locals make the partial's inputs explicit, so it can be rendered from any controller, a collection, a Turbo Stream or a mailer. Rails 7.1 strict locals (`<%# locals: (listing:) -%>`) enforce this.
*Follow-up:* What does `render @listings` do? Renders `listings/_listing` once per record, using `to_partial_path`, and returns nil for an empty collection.

**Q5. When do you use a helper versus a partial?**
A helper for small reusable logic that returns a short string or tag (formatting a price, a status badge). A partial for a bigger block of markup. When formatting logic for one model grows, consider a presenter or ViewComponent.
*Follow-up:* Why avoid `"<span>#{x}</span>".html_safe` in helpers? It skips escaping `x`; use `tag.span(x)`.

**Q6. How does `form_with model:` know the URL and HTTP method?**
It asks the model: a new record posts to the collection path (`POST /listings`), a persisted one submits to the member path with a hidden `_method=patch` field, which `Rack::MethodOverride` turns into PATCH. Field names are namespaced by the model (`listing[title]`).
*Follow-up:* A form without a model? `form_with url: search_path, method: :get`.

**Q7. How do you display validation errors in a form?**
Re-render the form with status 422 after a failed save and loop over `@model.errors.full_messages` (or show `errors[:field]` next to each field). Rails also wraps invalid fields in `field_with_errors` for styling.
*Follow-up:* Why 422 and not 200? Turbo only renders form responses with error statuses, and it is semantically correct.

**Q8. How does CSRF protection work in Rails?**
Rails stores a token in the session, embeds it in every form (`authenticity_token`) and in a meta tag for JavaScript, and verifies it on every non-GET request in `ActionController::Base`. A forged cross-site request cannot read the token, so it is rejected with `InvalidAuthenticityToken`.
*Follow-up:* Why are GET requests exempt? They must be safe and side-effect free, which is why destructive actions use POST, PATCH or DELETE.

**Q9. How do nested forms work?**
`accepts_nested_attributes_for :bill_shares` on the model plus `form.fields_for :bill_shares` in the view, which renders fields named `bill[bill_shares_attributes][0][amount_cents]`. Permit `bill_shares_attributes: [:id, :resident_id, :amount_cents, :_destroy]` in the controller.
*Follow-up:* How do you add rows dynamically? A Stimulus controller that clones a template with a unique index.

**Q10. How does I18n work in Rails views?**
Strings live in `config/locales/*.yml`; `t(".title")` looks up a key relative to the view path, `t("key", count: n)` pluralizes, `l(date)` localizes dates, and model attribute names come from `activerecord.attributes`. The locale is set per request with `I18n.with_locale`.
*Follow-up:* Name some number helpers. `number_to_currency`, `number_with_delimiter`, `number_to_percentage`, `number_to_human`.

**Q11. Walk me through the history of the Rails asset pipeline.**
Sprockets (Rails 3.1) concatenated and fingerprinted assets. Webpacker (5.1 to 6) brought webpack and npm, and has since been retired (Shakapacker continues it). Rails 7 offered import maps (default, no build step) or jsbundling and cssbundling (esbuild, Tailwind). Rails 8 made Propshaft, a minimal fingerprinting pipeline, the default.
*Follow-up:* Why fingerprint assets? The digest in the file name changes when content changes, so browsers can cache files forever.

**Q12. Import maps or jsbundling: how do you choose?**
Import maps suit Hotwire apps with modest JavaScript: no Node, no build, browsers load ES modules directly. Choose jsbundling (esbuild) or Shakapacker when you need JSX, TypeScript, heavy npm dependencies or React.
*Follow-up:* Can import maps handle JSX? No, JSX must be compiled first.

**Q13. What is Shakapacker, and why does React on Rails use it?**
The community-maintained successor to Webpacker (by ShakaCode). It integrates webpack builds with Rails (packs, manifest, dev server, view helpers), which React on Rails relies on for client and server bundles.
*Follow-up:* What did Webpacker do? Gave Rails apps an npm and webpack toolchain from Rails 5.1 to 6.

**Q14. How do you render a React component inside an ERB page?**
With React on Rails, register the component in a client bundle and call `react_component("Name", props: {...}, prerender: false)` in the view; the helper emits a DOM node and serialized props that the bundle mounts. Set `prerender: true` for server rendering.
*Follow-up:* What do you watch for in props? Only serialize needed attributes, camelCase keys if the JS expects them, and preload associations.

**Q15. What is ViewComponent, and when is it better than partials?**
A gem where each UI component is a Ruby class plus template with an explicit `initialize` signature. It gives clear inputs, encapsulated logic and fast unit tests, which pays off for design-system pieces reused across many pages.
*Follow-up:* How do you test one? `render_inline(PriceComponent.new(cents: 999))` and assert on the output.

**Q16. Your page shows the same flash message twice. Why?**
The action set `flash[:alert]` and then called `render`: the message shows now and survives to the next request. Use `flash.now` with `render` and `flash` with `redirect_to`.
*Follow-up:* Where should flash markup live? In a shared partial rendered from the layout.

## Further reading
- [Action View Overview](https://guides.rubyonrails.org/action_view_overview.html) and [Layouts and Rendering](https://guides.rubyonrails.org/layouts_and_rendering.html)
- [Action View Form Helpers](https://guides.rubyonrails.org/form_helpers.html) and [Rails Internationalization (I18n) API](https://guides.rubyonrails.org/i18n.html)
- [The Asset Pipeline](https://guides.rubyonrails.org/asset_pipeline.html) and [Securing Rails Applications](https://guides.rubyonrails.org/security.html)
