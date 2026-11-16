# Hotwire: Turbo and Stimulus

> **Tier:** T1 (know cold) · **Related:** [Views, ERB and forms](07-views-erb-forms-assets.md), [Routing and controllers](02-routing-and-controllers.md), [Action Cable](15-actioncable-mailer-activestorage.md), [React on Rails](16-react-on-rails.md), [Realtime and WebSockets](../04-web-fundamentals/09-realtime-websockets.md), [React fundamentals](../06-react/01-fundamentals.md)
> **On your resume:** Hotwire (Turbo, Stimulus) from the Visnext Rails training, server-rendered Rails views at Devntech, and the Colochousing Rails 7 app where ERB and React on Rails coexist.

## Quick revision
- **Hotwire = HTML over the wire:** the server renders HTML and sends it; a little JavaScript swaps it into the page. No JSON API and client-side templates needed for most screens. Default since Rails 7.0.
- **Turbo Drive** intercepts same-origin links and forms, fetches the next page with `fetch`, swaps the `<body>` and merges the `<head>`, and updates history: SPA-like speed with plain server pages.
- Form rules: after a successful submit, **redirect** (303 See Other after non-GET); after a failed one, **render with 4xx** (422). A 200 response to a form submission is not rendered.
- Opt out with `data-turbo="false"`; send non-GET links with `data-turbo-method="delete"`; confirm with `data-turbo-confirm`. Run page JS on `turbo:load` (or in Stimulus `connect()`), not `DOMContentLoaded`.
- Turbo Drive caches snapshots for instant back-button restores and previews; opt out per page with the `turbo-cache-control` meta tag.
- **Turbo Frames** (`turbo_frame_tag`) scope navigation to one region: a link or form inside a frame replaces only the matching frame from the response. `src:` + `loading: :lazy` lazy-load a frame; `target: "_top"` or `data-turbo-frame="_top"` breaks out.
- "Content missing" = the response had no frame with the same ID.
- **Turbo Streams** apply targeted DOM changes: `append`, `prepend`, `replace`, `update`, `remove`, `before`, `after`, plus `refresh` (Turbo 8). Respond with `format.turbo_stream` and a `.turbo_stream.erb` template.
- Broadcasts push streams over WebSockets: `broadcasts_to :event` on the model, `turbo_stream_from @event` in the view; Action Cable needs Redis or Solid Cable (Rails 8 default) in production.
- **Turbo 8 morphing:** `turbo_refreshes_with method: :morph, scroll: :preserve` plus `broadcasts_refreshes` lets the server just say "refresh" and the page re-renders while keeping scroll and state.
- **Stimulus** adds behavior to server HTML: `data-controller`, `data-action="input->search#query"`, targets, values, classes, outlets; lifecycle `initialize`, `connect`, `disconnect`.
- Choose Hotwire for CRUD-heavy apps, dashboards and forms built by a Rails team; choose React for very interactive client state, offline needs or a separate frontend team. They can be mixed: React islands inside Turbo pages.
- **Hotwire Native** wraps a Hotwire web app in native iOS and Android shells.

## 1. The Hotwire idea: HTML over the wire

In a classic SPA, the server returns JSON and the browser renders everything with JavaScript, so you maintain two apps: an API and a client with its own routing, state and templates. Hotwire keeps rendering on the server (ERB, partials, helpers you already have) and makes it feel fast:

| | SPA (React + JSON API) | Hotwire |
|---|---|---|
| Who renders HTML | The browser, from JSON | The server, from ERB |
| State lives | Client stores plus server | The server (database, session) |
| Code to maintain | API + frontend app + API contract | One Rails app |
| Typical JS size | Large bundle | Small (Turbo + a few Stimulus controllers) |
| Interactivity ceiling | Very high | High for most business apps; awkward for heavy client-side state |

Hotwire has three parts: **Turbo** (Drive, Frames, Streams) for navigation and updates without writing JavaScript, **Stimulus** for small sprinkles of JavaScript behavior, and **Hotwire Native** for mobile shells.

## 2. Turbo Drive

Turbo Drive is on as soon as `@hotwired/turbo-rails` is imported. For every same-origin link click and form submission it:
1. Cancels the browser's full page load and fetches the URL with `fetch`.
2. Replaces the `<body>`, merges new `<head>` elements, and pushes the URL to history.
3. Keeps the JavaScript and CSS already loaded, so the page does not re-parse them. A progress bar appears for slow requests.

**Forms and status codes** (the most common Turbo bug):

```ruby
def create
  @task = @house.tasks.new(task_params)
  if @task.save
    redirect_to house_tasks_path(@house), notice: "Task added."       # Turbo follows the redirect
  else
    render :new, status: :unprocessable_entity                         # 422: Turbo renders the errors
  end
end

def destroy
  @task.destroy
  redirect_to house_tasks_path(@house), status: :see_other             # 303: follow-up is a GET
end
```

A form submission answered with 200 and HTML is not rendered, so the page appears frozen. Rails 7+ scaffolds already use 422 and 303.

**Controlling Drive in markup:**

```erb
<%= link_to "Export CSV", export_tasks_path(format: :csv), data: { turbo: false } %>  <%# normal load %>
<%= link_to "Delete", task_path(task), data: { turbo_method: :delete, turbo_confirm: "Delete this task?" } %>
<%= button_to "Delete", task_path(task), method: :delete %>   <%# a real form: works without JS too %>
<%= form_with model: @upload, data: { turbo: false } do |form| %> ... <% end %>
```

**Events and JavaScript that "stops working":** after a Drive navigation there is no new `DOMContentLoaded`, so code bound to it runs only on the first page. Run per-page code on `document.addEventListener("turbo:load", ...)`, or better, put behavior in Stimulus controllers, whose `connect()` runs whenever their element appears. Other useful events: `turbo:before-visit`, `turbo:submit-end`, `turbo:before-cache`, `turbo:frame-load`, `turbo:before-stream-render`.

**Cache and previews:** Drive keeps snapshots of recently visited pages. The back button restores a snapshot instantly; an ordinary visit to a cached page shows the snapshot as a preview while the fresh page loads. Pages that must never show stale content (a checkout, a balance) opt out with `<meta name="turbo-cache-control" content="no-cache">`; tear down third-party widgets in `turbo:before-cache` so the snapshot is clean. `data-turbo-permanent` (with an `id`) keeps an element, such as a playing audio player, across navigations. Assets tagged `data-turbo-track="reload"` force a full reload when their fingerprint changes after a deploy. Turbo 8 also prefetches links on hover by default (`data-turbo-prefetch="false"` disables it).

## 3. Turbo Frames

A frame is a `<turbo-frame id="...">` element that scopes navigation: links and forms inside it fetch a page, Turbo extracts the frame with the **same ID** from the response, and replaces only that region. The rest of the page (scroll position, other inputs) is untouched.

**Inline editing:**

```erb
<%# app/views/tasks/_task.html.erb %>
<%= turbo_frame_tag task do %>                  <%# <turbo-frame id="task_42"> via dom_id %>
  <span><%= task.title %></span>
  <%= link_to "Edit", edit_task_path(task) %>   <%# loads into this frame only %>
<% end %>

<%# app/views/tasks/edit.html.erb %>
<h1>Edit task</h1>                              <%# ignored when requested by the frame %>
<%= turbo_frame_tag @task do %>
  <%= render "form", task: @task %>
<% end %>
```

Flow: "Edit" fetches `/tasks/42/edit` and swaps in the form. Submitting the form either fails (the controller renders `edit` with 422 and the frame shows the errors) or succeeds (the controller redirects to a page that contains `task_42`, and the frame shows the updated row). The same controller actions still work as full pages when visited directly.

**Lazy loading:**

```erb
<%= turbo_frame_tag "house_stats", src: house_stats_path(@house), loading: :lazy do %>
  <p>Loading stats...</p>                       <%# placeholder until the frame is visible and loaded %>
<% end %>
```

Without `loading: :lazy`, the `src` loads as soon as the page renders, which is still useful for slow sections that should not block the main response.

**Targeting and breaking out:**

```erb
<%= turbo_frame_tag "listings", target: "_top" do %> ... <% end %>       <%# links inside navigate the whole page %>
<%= link_to "Open", task_path(task), data: { turbo_frame: "_top" } %>    <%# one link escapes its frame %>
<%= link_to "Page 2", tasks_path(page: 2), data: { turbo_frame: "tasks_list" } %>  <%# drive another frame %>
```

**"Content missing":** if the response contains no frame with the matching ID (a redirect to a page without it, a login page, an error page, a typo in the ID), Turbo replaces the frame with "Content missing", logs an error and fires `turbo:frame-missing`. Fix it by rendering the matching frame, making the link target `_top`, or handling the event to perform a full visit.

## 4. Turbo Streams

A Turbo Stream is a small piece of HTML that tells the page what to change. It arrives either as the response to a form submission (content type `text/vnd.turbo-stream.html`) or over a WebSocket:

```html
<turbo-stream action="append" target="comments">
  <template>
    <div id="comment_7">Great idea!</div>
  </template>
</turbo-stream>
```

| Action | Effect on the target element |
|---|---|
| `append` / `prepend` | Insert the template at the end / start of the target's children |
| `replace` | Replace the whole target element |
| `update` | Replace only the target's inner content |
| `remove` | Remove the target (no template) |
| `before` / `after` | Insert next to the target, outside it |
| `refresh` | Ask the page to reload itself, ideally with morphing (Turbo 8) |

Responding with streams from a controller:

```ruby
class CommentsController < ApplicationController
  def create
    @event = current_house.events.find(params[:event_id])
    @comment = @event.comments.build(comment_params.merge(author: current_resident))

    respond_to do |format|
      if @comment.save
        format.turbo_stream                                  # create.turbo_stream.erb
        format.html { redirect_to @event }                   # fallback without JS
      else
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace("new_comment",
                   partial: "comments/form", locals: { event: @event, comment: @comment }),
                 status: :unprocessable_entity
        end
        format.html { render "events/show", status: :unprocessable_entity }
      end
    end
  end
end
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.append "comments", @comment %>              <%# renders comments/_comment %>
<%= turbo_stream.update "comments_count", @event.comments.size.to_s %>
<%= turbo_stream.replace "new_comment" do %>
  <%= render "comments/form", event: @event, comment: Comment.new %>
<% end %>
```

Turbo adds `text/vnd.turbo-stream.html` to the `Accept` header of form submissions, which is how `format.turbo_stream` gets picked. Targets are DOM IDs, so `dom_id(record)` in partials and `turbo_stream.remove(@comment)` line up automatically. Prefer redirects for full page changes and streams when several small regions must change at once (append the new row, update a counter, reset the form).

## 5. Broadcasting streams over Action Cable

Streams can also be pushed to every browser watching a page, for live updates without polling:

```ruby
class Comment < ApplicationRecord
  belongs_to :event
  broadcasts_to :event        # on create/update/destroy: append/replace/remove in "comments" for that event
end

# Or explicitly, with your own target and timing
class Task < ApplicationRecord
  belongs_to :house
  after_create_commit  -> { broadcast_prepend_later_to house, target: "tasks" }
  after_destroy_commit -> { broadcast_remove_to house }
end
```

```erb
<%# app/views/events/show.html.erb %>
<%= turbo_stream_from @event %>        <%# subscribes this page over Action Cable (signed stream name) %>
<div id="comments">
  <%= render @event.comments %>
</div>
```

Under the hood, `turbo_stream_from` opens an Action Cable subscription; when a model broadcasts, the server renders the partial and publishes the stream to every subscriber. The `_later_` variants render in a background job, keeping the request fast. In production Action Cable needs a pub/sub backend: Redis is the classic choice, and Rails 8 defaults to **Solid Cable**, which uses the database. Broadcasts render without a request, so partials must not depend on `current_user` or other request state; stream-specific variants (per user or per role) need separate streams. Details in [Action Cable](15-actioncable-mailer-activestorage.md).

## 6. Turbo 8 morphing and page refreshes

Writing precise streams for every change gets tedious. Turbo 8 (2024) adds **page refreshes with morphing**: instead of describing each DOM change, the server says "refresh", the browser re-fetches the current page, and Turbo morphs the existing DOM into the new one, changing only what differs and keeping scroll position, focus and form state.

```erb
<%# layout head %>
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```

```ruby
class Task < ApplicationRecord
  belongs_to :house
  broadcasts_refreshes_to :house     # any change -> a refresh stream to pages subscribed to the house
end
```

Refresh broadcasts are debounced, so a burst of changes triggers one refresh. Mark elements that must survive untouched (an open dropdown, a video) with `data-turbo-permanent`. The trade-off is an extra full-page request per refresh in exchange for much simpler code.

## 7. Stimulus

Stimulus attaches small JavaScript controllers to server-rendered HTML through data attributes. It never renders HTML itself; it adds behavior to markup that already exists, and it re-connects automatically whenever Turbo swaps content in.

| Concept | Markup | In the controller |
|---|---|---|
| Controller | `data-controller="toggle"` | `toggle_controller.js` in `app/javascript/controllers` |
| Action | `data-action="click->toggle#toggle"` (event defaults: click for buttons, input for fields, submit for forms) | `toggle(event) { ... }` |
| Target | `data-toggle-target="content"` | `static targets = ["content"]`, then `this.contentTarget`, `this.contentTargets`, `this.hasContentTarget` |
| Value | `data-counter-max-value="280"` | `static values = { max: Number }`, `this.maxValue`, `maxValueChanged()` |
| Class | `data-toggle-hidden-class="hidden"` | `static classes = ["hidden"]`, `this.hiddenClass` |
| Outlet | `data-search-results-outlet="#results"` | `static outlets = ["results"]`, `this.resultsOutlet` (another controller) |
| Lifecycle | n/a | `initialize()`, `connect()`, `disconnect()`, `contentTargetConnected(el)` |

**1. Toggle:**

```js
// app/javascript/controllers/toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["content"]
  static classes = ["hidden"]

  toggle() {
    this.contentTarget.classList.toggle(this.hiddenClass)
  }
}
```

```erb
<div data-controller="toggle" data-toggle-hidden-class="hidden">
  <button type="button" data-action="toggle#toggle">Show bill details</button>
  <div data-toggle-target="content" class="hidden"><%= render "bills/details", bill: @bill %></div>
</div>
```

**2. Character counter:**

```js
// app/javascript/controllers/character_counter_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "count"]
  static values = { max: Number }

  connect() {
    this.update()
  }

  update() {
    const remaining = this.maxValue - this.inputTarget.value.length
    this.countTarget.textContent = `${remaining} characters left`
    this.countTarget.classList.toggle("text-red-600", remaining < 0)
  }
}
```

```erb
<div data-controller="character-counter" data-character-counter-max-value="280">
  <%= form.text_area :body, data: { character_counter_target: "input", action: "character-counter#update" } %>
  <p data-character-counter-target="count"></p>
</div>
```

**3. Debounced auto-submit search** (results load into a frame and the URL updates):

```js
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  search() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => this.element.requestSubmit(), this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

```erb
<%= form_with url: listings_path, method: :get,
      data: { controller: "auto-submit", turbo_frame: "listings", turbo_action: "advance" } do |form| %>
  <%= form.search_field :q, value: params[:q], data: { action: "input->auto-submit#search" } %>
<% end %>

<%= turbo_frame_tag "listings" do %>
  <%= render(@listings) || tag.p("No matches.") %>
<% end %>
```

`requestSubmit()` (unlike `submit()`) fires the submit event, so Turbo intercepts it. The form targets the `listings` frame, the `index` action renders the same frame with filtered results, and `turbo_action: "advance"` pushes the search URL into history so it is shareable. The debounce sends one request after the user pauses typing instead of one per keystroke.

## 8. Hotwire vs React, and mixing them

| Factor | Hotwire | React (SPA, or islands via React on Rails) |
|---|---|---|
| Typical app | CRUD, admin panels, dashboards, forms, content-heavy pages | Editors, drag-and-drop boards, complex client-side state, offline-capable UIs |
| Rendering | Server-rendered ERB | Client renders from JSON (or SSR with extra setup) |
| Team | Rails developers own the whole feature | Frontend specialists, separate API contract |
| Latency per interaction | A server round trip for most changes | Instant local updates and optimistic UI |
| JavaScript shipped | Small | Larger bundles |
| Reuse with mobile | Hotwire Native shells around the web app | React Native shares language, patterns and some code |
| Testing | Rails system tests cover most behavior | Component tests plus API tests plus end-to-end tests |

Rules of thumb: start with Hotwire when the screen is mostly "show data, edit data" and the team is Rails-first; reach for React for a specific widget whose state lives in the browser (a bill-splitting calculator with live totals, a calendar with drag and drop, a chart builder). Mixing works well: Turbo handles navigation and most pages, and `react_component` mounts React islands where needed. When mixing, make sure React components unmount and remount correctly across Turbo navigations (React on Rails documents its Turbo support; check the version you use), and keep a single source of truth for each piece of state.

**Hotwire Native** (2024, the successor to Turbo Native and Strada) wraps the web app in native iOS and Android shells: native navigation and transitions around web content, with bridge components for native UI where needed. It competes with React Native when one Rails team wants mobile apps without a separate codebase; React Native wins when the app needs a mostly native UI or heavy offline behavior.

## Common pitfalls
- Returning 200 for failed form submissions (frozen page) or redirecting with 302 after DELETE instead of 303.
- Page JavaScript bound to `DOMContentLoaded`, which runs only on the first full load; use Stimulus or `turbo:load`.
- Frame ID mismatches between the page and the response, causing "Content missing".
- Forgetting `target: "_top"` for links inside a frame that should navigate the whole page (for example, "View" links in a frame-wrapped list).
- Stale snapshots: third-party widgets duplicated on back navigation because they were not torn down in `turbo:before-cache`.
- Broadcast partials that use `current_user` or other request state (there is no request during a broadcast).
- Rendering streams with `flash` set for a redirect that never happens.
- Stimulus controllers that leak timers or listeners because `disconnect()` does not clean up.
- Using `form.submit()` in JavaScript, which skips the submit event and therefore Turbo; use `requestSubmit()`.

## How to talk about this in an interview
- The Visnext training covered Turbo and Stimulus explicitly, so you can speak to the mental model with confidence: Drive for navigation, Frames for scoped regions, Streams for targeted updates, Stimulus for sprinkles of behavior.
- Colochousing uses both ERB and React on Rails, which gives you a credible answer to "Hotwire or React?": it depends on the screen, and they can coexist.
- In legacy monoliths you may meet older patterns (rails-ujs `data-method`, `remote: true` forms, jQuery); explain how Turbo replaces them (`data-turbo-method`, Turbo Streams) and why migrations happen screen by screen.
- Keep it honest: if your production Hotwire experience is limited, say the training gave you the foundations and describe exactly what you would build (an inline-edit frame, a live search, a broadcast list).

> **Fill in:** the Hotwire exercises from the Visnext training (for example a Turbo Frame inline edit or a Stimulus controller you wrote).

> **Fill in:** whether any Devntech client app uses Turbo or Stimulus, and on which screens.

## Interview questions

**Q1. What is Hotwire, and how does it differ from an SPA?**
Hotwire sends server-rendered HTML instead of JSON: Turbo swaps HTML into the page and Stimulus adds small behaviors. An SPA renders in the browser from JSON and needs its own routing and state. Hotwire keeps one Rails codebase and little JavaScript; SPAs suit heavy client-side interactivity.
*Follow-up:* What are its parts? Turbo (Drive, Frames, Streams), Stimulus and Hotwire Native.

**Q2. What is the difference between Turbo Drive, Turbo Frames and Turbo Streams?**
Drive speeds up whole-page navigation by fetching pages and swapping the body. Frames scope navigation to one region: a link or form inside a frame replaces only the frame with the matching ID. Streams apply targeted actions (append, replace, remove...) to any element by ID, from a form response or over a WebSocket.
*Follow-up:* Which would you use to add a comment and update a counter? Streams, because two separate regions change.

**Q3. What does Turbo Drive do, and how do you turn it off?**
It intercepts same-origin link clicks and form submissions, fetches the response, replaces the body, merges the head and updates history, so CSS and JS are not reloaded. Disable it per element with `data-turbo="false"` (on a link, form or container).
*Follow-up:* When would you disable it? File downloads, third-party payment redirects or pages with scripts that cannot handle Drive.

**Q4. Why must a failed form return 422, and why 303 after a DELETE?**
Turbo submits forms with `fetch` and only renders a form response when the status is 4xx or 5xx; a 200 is ignored. Success should redirect, and 303 See Other makes the follow-up request a GET instead of repeating the DELETE or PATCH.
*Follow-up:* What does a 200 with errors look like to users? Nothing happens; the page seems frozen.

**Q5. A jQuery plugin works on first load but not after clicking a link. Why?**
Turbo Drive does not trigger a new `DOMContentLoaded`, so initialization code runs only once. Initialize on `turbo:load`, or move it into a Stimulus controller whose `connect()` runs whenever the element appears, and tear it down in `disconnect()` or `turbo:before-cache`.
*Follow-up:* Why might it appear twice after the back button? The cached snapshot included the plugin's generated DOM.

**Q6. How do you build inline editing with Turbo Frames?**
Wrap each row in `turbo_frame_tag record`, link "Edit" to the normal edit action, and wrap the edit form in a frame with the same ID. The frame swaps in the form; on success the controller redirects to a page containing the frame, on failure it renders the form with 422 inside the frame.
*Follow-up:* What happens if you open the edit URL directly? It renders as a normal full page.

**Q7. What does "Content missing" mean in a Turbo Frame?**
The response did not contain a `turbo-frame` with the same ID, for example after a redirect to a login page or a page without that frame. Fix the ID, render the frame in the response, target `_top` for full-page navigations, or handle `turbo:frame-missing`.
*Follow-up:* How do you make one link inside a frame navigate the whole page? `data: { turbo_frame: "_top" }`.

**Q8. What Turbo Stream actions exist, and how do you respond with them from a controller?**
`append`, `prepend`, `replace`, `update`, `remove`, `before`, `after` and (Turbo 8) `refresh`. Add `format.turbo_stream` in `respond_to` and write `create.turbo_stream.erb` with helpers like `turbo_stream.append "comments", @comment`, or render inline with `render turbo_stream: turbo_stream.replace(...)`.
*Follow-up:* `replace` vs `update`? `replace` swaps the whole element; `update` changes only its contents.

**Q9. How do real-time updates with Turbo Streams work?**
The page subscribes with `turbo_stream_from @event`, which opens an Action Cable channel. Models broadcast with `broadcasts_to :event` or `broadcast_append_later_to`, the server renders the partial and pushes the stream to every subscriber. Production needs a pub/sub backend such as Redis or Solid Cable.
*Follow-up:* Why can broadcast partials not use `current_user`? They render outside any request, once for all subscribers.

**Q10. What did Turbo 8 add with morphing?**
Page refreshes with morphing: with `turbo_refreshes_with method: :morph`, a `refresh` (from a broadcast via `broadcasts_refreshes` or a redirect to the same page) re-fetches the page and morphs the DOM, keeping scroll and form state. You stop hand-writing many small streams.
*Follow-up:* How do you protect an element from being morphed? `data-turbo-permanent`.

**Q11. Explain Stimulus controllers, actions, targets and values.**
A controller is a JS class attached with `data-controller`. Actions map DOM events to methods (`data-action="input->search#query"`). Targets are named elements (`data-search-target="results"`, `this.resultsTarget`). Values are typed data attributes (`data-search-delay-value="300"`, `this.delayValue`) with change callbacks. Classes and outlets configure CSS classes and connect to other controllers.
*Follow-up:* Does Stimulus render HTML? No, it adds behavior to HTML the server rendered.

**Q12. What are Stimulus lifecycle callbacks, and why do they matter with Turbo?**
`initialize()` once per controller instance, `connect()` each time the element enters the DOM, `disconnect()` when it leaves. Because Turbo swaps content constantly, putting setup in `connect()` and cleanup in `disconnect()` makes behavior reliable without global event wiring.
*Follow-up:* What belongs in `disconnect()`? Clearing timers, removing global listeners, destroying third-party widgets.

**Q13. How would you build a live search with Hotwire?**
A GET form targeting a results frame, a Stimulus controller that debounces input and calls `requestSubmit()`, and an index action that filters and renders the same frame. `data-turbo-action="advance"` keeps the URL in sync so searches are shareable.
*Follow-up:* Why `requestSubmit()` instead of `submit()`? `submit()` bypasses the submit event, so Turbo never sees it.

**Q14. When would you choose Hotwire over React?**
When most screens are CRUD, forms and dashboards, the team is Rails-first, and you want one codebase with server-side state and small JavaScript. Choose React for screens with rich client-side state, instant feedback on every interaction, offline needs or a dedicated frontend team, often as islands inside a Hotwire app.
*Follow-up:* A real example? A Turbo-driven house management page with a React bill-splitting widget mounted through React on Rails.

**Q15. Can Hotwire and React live in the same app?**
Yes: Turbo handles navigation and most pages while React components mount in specific places through `react_component` or a custom mount. Ensure components unmount on Turbo navigation and do not fight over the same DOM or state.
*Follow-up:* What breaks if they fight? Turbo snapshots or streams can replace DOM that React thinks it owns, causing stale or broken widgets.

**Q16. What is Hotwire Native?**
A framework for wrapping a Hotwire web app in native iOS and Android shells, with native navigation and optional bridge components for native UI. It lets a Rails team ship mobile apps reusing server-rendered screens; React Native is the alternative when the app needs a mostly native UI.
*Follow-up:* What did it replace? Turbo Native and Strada.

## Further reading
- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction) and [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction)
- [Working with JavaScript in Rails](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)
- [Action Cable Overview](https://guides.rubyonrails.org/action_cable_overview.html)
