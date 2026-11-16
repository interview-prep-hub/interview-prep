# Routing and Controllers

> **Tier:** T1 (know cold) · **Related:** [Rails fundamentals](01-rails-fundamentals.md), [Views, ERB and forms](07-views-erb-forms-assets.md), [Hotwire](08-hotwire-turbo-stimulus.md), [API development](10-api-development.md), [Authentication and authorization](11-authentication-authorization.md), [REST API design](../04-web-fundamentals/04-rest-api-design.md)
> **On your resume:** full-stack features and REST APIs in legacy Rails monoliths at Devntech, the Colochousing Rails 7 web app and REST API, and the Visnext MVC training.

## Quick revision
- `resources :products` = 7 RESTful routes: index, new, create, show, edit, update, destroy. Trim with `only:` / `except:`.
- Singular `resource :profile`: no `:id` in the path and no index, but it still maps to the plural `ProfilesController`.
- **Member** route acts on one record (`/orders/:id/cancel`); **collection** route acts on the set (`/orders/search`).
- Nest at most one level. `shallow: true` nests only index/new/create; show/edit/update/destroy get flat URLs.
- `namespace :admin` = URL prefix + `Admin::` module + `admin_` helper prefix. `scope "/admin"` = URL only. `scope module: :admin` = module only.
- Routes match top to bottom, first match wins. `bin/rails routes -g product` greps, `-c products` filters by controller.
- `*_path` = relative path (views, redirects inside the app); `*_url` = absolute URL (emails, external links, `Location` headers).
- Polymorphic routing: `link_to "Show", @product`, `redirect_to @product` and `form_with model: [@store, @product]` derive the route from the model class.
- `params` merges path params, query string and body; form values arrive as strings.
- **Strong parameters:** `params.require(:product).permit(:name, :price)` stops mass-assignment attacks. Rails 8 added `params.expect(product: [:name, :price])`, which returns 400 instead of 500 for malformed input.
- `before_action` / `after_action` / `around_action` with `only:` / `except:`; `skip_before_action`. A before_action that renders or redirects **halts** the chain.
- `render` = respond now with a template (same request, instance variables kept). `redirect_to` = 302/303 with a `Location` header, so the browser makes a new GET.
- Render or redirect only once per action, or you get `AbstractController::DoubleRenderError`; use `return redirect_to(...)`.
- With Turbo: failed form submission -> `render :new, status: :unprocessable_entity` (422); success -> redirect; after DELETE use `status: :see_other` (303).
- `respond_to { |format| format.html; format.json; format.turbo_stream }`; `head :no_content` for an empty response.
- `flash` survives one redirect (shown on the next request); `flash.now` is for the current render only.
- Session defaults to an encrypted cookie (about 4 KB). `cookies.signed` = tamper-proof, `cookies.encrypted` = tamper-proof and unreadable.
- `rescue_from ActiveRecord::RecordNotFound, with: :not_found` centralizes error responses.
- `current_user` lives in ApplicationController with `helper_method`; shared controller code goes in concerns; JSON-only controllers inherit `ActionController::API`.

## 1. The routes DSL and resources

Routes map an HTTP verb + path to `controller#action`. Rails favors REST: model each thing as a **resource** and use the standard actions.

```ruby
# config/routes.rb
Rails.application.routes.draw do
  root "products#index"                                  # GET /
  resources :products                                    # the 7 actions below
  resources :stores, only: [:index, :show]
  resource :profile, only: [:show, :edit, :update]       # singular: /profile
  get "about", to: "pages#about", as: :about             # about_path
  get "/old-catalog", to: redirect("/products")          # 301 by default
  get "/shops/:id", to: redirect("/stores/%{id}", status: 302)
end
```

| Verb | Path | Action | Helper | Used for |
|---|---|---|---|---|
| GET | `/products` | index | `products_path` | List |
| GET | `/products/new` | new | `new_product_path` | Blank form |
| POST | `/products` | create | `products_path` | Save the form |
| GET | `/products/:id` | show | `product_path(product)` | One record |
| GET | `/products/:id/edit` | edit | `edit_product_path(product)` | Edit form |
| PATCH / PUT | `/products/:id` | update | `product_path(product)` | Save changes |
| DELETE | `/products/:id` | destroy | `product_path(product)` | Delete |

A singular `resource :profile` generates the same set minus index and without `:id` (`GET /profile`, `PATCH /profile`), for things a user has exactly one of. Note that it still routes to `ProfilesController` (plural).

Inspect routes instead of guessing:

```bash
bin/rails routes -g product          # grep by name, path or controller
bin/rails routes -c admin/reports    # one controller
bin/rails routes --expanded          # one block per route, easier to read
```

## 2. Nested and shallow routes, member and collection

Nesting expresses ownership in the URL:

```ruby
resources :stores do
  resources :products, shallow: true
end
```

| Route | Path | Helper |
|---|---|---|
| index / create | `/stores/:store_id/products` | `store_products_path(store)` |
| new | `/stores/:store_id/products/new` | `new_store_product_path(store)` |
| show / update / destroy | `/products/:id` | `product_path(product)` |
| edit | `/products/:id/edit` | `edit_product_path(product)` |

Without `shallow`, every route needs both IDs (`/stores/1/products/42`), and deeper nesting (`/stores/1/products/42/variants/7`) becomes painful to build and redundant, because a product already knows its store. Rule of thumb: never nest more than one level; use `shallow` when the child has its own unique ID.

**Member vs collection** for extra actions:

```ruby
resources :orders do
  member do
    patch :cancel         # PATCH /orders/:id/cancel -> orders#cancel, cancel_order_path(order)
  end
  collection do
    get :search           # GET /orders/search -> orders#search, search_orders_path
  end
end
# one-liner form: patch :cancel, on: :member
```

### Going deeper: CRUD-only controllers
Many Rails teams prefer turning custom actions into their own small resource instead, which keeps controllers to the standard seven actions:

```ruby
resources :orders do
  resource :cancellation, only: :create, module: :orders  # POST /orders/:order_id/cancellation
end                                                       # -> Orders::CancellationsController#create
```

## 3. namespace vs scope vs module

| DSL | URL | Controller | Helper |
|---|---|---|---|
| `namespace :admin { resources :reports }` | `/admin/reports` | `Admin::ReportsController` | `admin_reports_path` |
| `scope "/admin" { resources :reports }` | `/admin/reports` | `ReportsController` | `reports_path` |
| `scope module: :admin { resources :reports }` | `/reports` | `Admin::ReportsController` | `reports_path` |

(The braces in the table are shorthand for `do ... end` blocks.) `namespace` is the usual choice for admin areas and API versions:

```ruby
namespace :api, defaults: { format: :json } do
  namespace :v1 do
    resources :listings, only: [:index, :show]   # GET /api/v1/listings -> Api::V1::ListingsController
  end
end
```

## 4. Constraints, concerns and route order

```ruby
get "products/:id", to: "products#show", constraints: { id: /\d+/ }  # only numeric IDs

constraints subdomain: "admin" do
  resources :reports
end

# Routing concerns: reuse a block of routes across resources
concern :commentable do
  resources :comments, only: [:index, :create]
end
resources :events, concerns: :commentable
resources :tasks,  concerns: :commentable

# Mount a Rack app behind an admin check (authenticate comes from Devise)
require "sidekiq/web"
authenticate :user, ->(user) { user.admin? } do
  mount Sidekiq::Web => "/sidekiq"
end
```

A constraint can also be a lambda or any object with `matches?(request)`, which is handy for feature flags or IP allowlists.

**Order bug:** `resources :products` defines `GET /products/:id`, so a later `get "products/featured"` never matches: `featured` is treated as an ID and you get `RecordNotFound`. Declare it as a collection route or put it above `resources`.

## 5. Path vs URL helpers and polymorphic routes

```ruby
product_path(@product)   # => "/products/42"          relative: links and redirects inside the app
product_url(@product)    # => "https://shop.example.com/products/42"
                         #    absolute: emails, webhooks, Location headers, links shared outside the app
```

Mailers have no request, so `*_url` needs `config.action_mailer.default_url_options = { host: "shop.example.com" }`.

Helpers call `to_param` on records, which returns the ID by default. Override it for friendly URLs; `find` still works because the string is cast to its leading integer:

```ruby
class Product < ApplicationRecord
  def to_param
    "#{id}-#{name.parameterize}"   # /products/42-blue-mug; Product.find("42-blue-mug") finds 42
  end
end
```

**Polymorphic routes** build the helper name from the model class, so the same view works for any resource:

```ruby
link_to "Show", @product                 # product_path(@product)
redirect_to @product                     # product_url(@product)
form_with model: [@store, @product]      # new record: POST store_products_path(@store)
                                         # persisted:  PATCH store_product_path(@store, @product)
                                         # (with shallow routes, use model: @product when persisted)
polymorphic_path([@store, @product])     # the method behind all of these
link_to "Back", @comment.commentable     # works whether commentable is an Event or a Task
```

## 6. Controllers and params

A controller is a Ruby class whose public methods are actions. Rails creates a fresh instance per request, so instance variables (`@product`) are per request and are how data reaches the view.

`params` (an `ActionController::Parameters` object) merges three sources:

| Source | Example | Access |
|---|---|---|
| Path segments | `/products/42` | `params[:id]` => `"42"` |
| Query string | `/products?page=2&sort=price` | `params[:page]` => `"2"` |
| Request body | form fields `product[name]=Mug`, or a JSON body | `params[:product][:name]` |

Form and query values are always strings (`params[:page].to_i`); JSON bodies keep their types. For JSON requests, Rails' params wrapper also nests top-level keys under the controller's model name, so `{"name": "Mug"}` sent to `ProductsController` is also available as `params[:product][:name]`. `params[:controller]` and `params[:action]` are always present.

## 7. Strong parameters and mass assignment

**Mass assignment** means setting many attributes at once: `Product.new(params[:product])`. Passing raw user input is dangerous because an attacker can add any field to the form (`product[store_id]=7`, `user[admin]=true`). Since Rails 4, passing unfiltered params raises `ActiveModel::ForbiddenAttributesError`, so you must whitelist:

```ruby
class ProductsController < ApplicationController
  def create
    @product = current_store.products.new(product_params)
    # ...
  end

  private

  def product_params
    params.require(:product).permit(
      :name, :price, :description,
      tag_ids: [],                                       # array of scalars
      dimensions: [:width, :height],                     # nested hash
      variants_attributes: [:id, :sku, :price, :_destroy] # accepts_nested_attributes_for
    )
  end
end
```

- `require(:product)` raises `ActionController::ParameterMissing` (rendered as 400 Bad Request) when the key is missing or empty.
- `permit` silently drops anything not listed (logged in development; set `config.action_controller.action_on_unpermitted_parameters = :raise` to fail loudly in tests).
- `permit(metadata: {})` allows an arbitrary hash: use it sparingly.

Rails 8 added `params.expect`, which combines both steps and checks the shape. If someone sends `product=hello` (a string instead of a hash), `require(...).permit(...)` blows up with a 500, while `expect` raises `ParameterMissing` and returns 400:

```ruby
params.expect(product: [:name, :price, tag_ids: []])   # hash of permitted keys
params.expect(:id)                                       # a single scalar
params.expect(items: [[:sku, :quantity]])                # double brackets = array of hashes
```

Strong parameters are not authorization: permitting `:store_id` still lets a user move a product into another store. Scope lookups to the current user (`current_user.stores.find(params[:store_id])`) and authorize with a policy (see [authorization](11-authentication-authorization.md)).

## 8. Filters: before, after and around actions

```ruby
class OrdersController < ApplicationController
  before_action :require_login
  before_action :set_order, only: %i[show edit update destroy]
  after_action  :log_view, only: :show
  around_action :use_customer_time_zone
  skip_before_action :require_login, only: :receipt   # declared in ApplicationController

  def show; end

  private

  def set_order
    @order = current_user.orders.find(params[:id])     # scoped: other users' orders 404
  end

  def use_customer_time_zone(&block)
    Time.use_zone(current_user&.time_zone || "UTC", &block)  # the action runs inside the block
  end

  def log_view
    Rails.logger.info("order_viewed id=#{@order.id}")
  end
end
```

- Filters run in declaration order, parent class first.
- If a before_action calls `render`, `redirect_to` or `head`, the chain **halts** and the action never runs. That is how `require_login` works.
- after_actions run only when the action completes without raising. An around_action must `yield` (or call the block) or the action is skipped.
- `only:` / `except:` limit filters per action; `skip_before_action` removes an inherited one. Use `if:` / `unless:` for conditional filters.

## 9. render vs redirect_to

| | `render` | `redirect_to` |
|---|---|---|
| What happens | Builds the response body now, in this request | Sends 302 (or 301/303) with a `Location` header |
| New request? | No | Yes, the browser issues a new GET |
| Instance variables | Available in the template | Lost (new request, new controller) |
| URL in the address bar | Unchanged (for example still `/products` after a failed POST) | The new location |
| Typical use | Show a form again with errors; show a page | After a successful create/update/destroy (Post/Redirect/Get), so a refresh does not resubmit |

```ruby
def create
  @product = current_store.products.new(product_params)
  if @product.save
    redirect_to @product, notice: "Product created."            # 302 -> GET /products/:id
  else
    render :new, status: :unprocessable_entity                  # 422 + form with errors
  end
end

def destroy
  @product.destroy
  redirect_to products_path, notice: "Product deleted.", status: :see_other  # 303
end
```

**Why the status codes matter with Turbo:** Turbo submits forms with `fetch`. It only renders a form response when the status is 4xx or 5xx (so a failed save must return 422; a 200 is ignored and the page looks frozen), and it expects a redirect after success. The 303 See Other after a DELETE guarantees the follow-up request is a GET instead of repeating the DELETE. Rails 7+ scaffolds generate exactly this. Newer Rack versions also name the 422 symbol `:unprocessable_content`; `status: 422` works everywhere.

**render does not return.** Code after it keeps running, and a second render or redirect raises `AbstractController::DoubleRenderError`:

```ruby
def update
  redirect_to(root_path, alert: "Locked") and return if @order.locked?  # or: return redirect_to(...)
  @order.update!(order_params)
  redirect_to @order
end
```

Other response options: `render json: @product, status: :created`, `render plain: "ok"`, `render partial: "row", locals: { product: @product }`, `render "shared/empty"`, `send_data csv, filename: "orders.csv"`, and `head :no_content` (status only, no body).

**Formats with respond_to:**

```ruby
def create
  @comment = @post.comments.build(comment_params)
  respond_to do |format|
    if @comment.save
      format.turbo_stream                                    # create.turbo_stream.erb
      format.html { redirect_to @post, notice: "Comment added." }
      format.json { render json: @comment, status: :created }
    else
      format.html { render :new, status: :unprocessable_entity }
      format.json { render json: { errors: @comment.errors }, status: :unprocessable_entity }
    end
  end
end
```

Rails picks the format from the `Accept` header or an extension (`/comments.json`); an unsupported format gets `406 Not Acceptable`.

## 10. Flash, session and cookies

**flash** is a hash stored in the session that lives for exactly one more request, which makes it perfect for messages after a redirect. **flash.now** is visible only in the current request, which is what you want with `render`:

```ruby
redirect_to @order, notice: "Order placed."      # sets flash[:notice]; shown after the redirect
flash.now[:alert] = "Please fix the errors."     # shown in this render only
render :new, status: :unprocessable_entity
```

| Mistake | Symptom |
|---|---|
| `flash[:alert]` + `render` | Message shows now and again on the next page |
| `flash.now[:alert]` + `redirect_to` | Message never shows (it died with the first request) |

`flash.keep` carries messages one more request. `notice` and `alert` have shortcut readers in views.

**session** stores small per-user data across requests: `session[:user_id] = user.id`. The default `CookieStore` keeps it in an encrypted, signed cookie: no server storage, about a 4 KB limit, and you cannot invalidate one session server-side unless you check something stored in the database. Call `reset_session` on login and logout to prevent session fixation. Alternatives: a cache or Redis store, or a database store via a gem.

**cookies**:

```ruby
cookies[:theme] = "dark"                                   # plain: user can read and edit it
cookies.signed[:visitor_id] = 123                          # readable but tamper-proof
cookies.encrypted[:remember_token] = token                 # unreadable and tamper-proof
cookies.permanent.encrypted[:remember_token] = token       # expires in about 20 years
cookies[:banner] = { value: "hidden", expires: 1.week, httponly: true, secure: true, same_site: :lax }
cookies.delete(:theme)
```

## 11. rescue_from and error handling

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionController::ParameterMissing do |e|
    render json: { error: e.message }, status: :bad_request
  end

  private

  def not_found
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end
end
```

Handlers are searched bottom-up, so declare general exceptions first and specific ones later. Avoid `rescue_from StandardError` unless you also report the error to your error tracker; swallowing exceptions hides bugs.

## 12. ApplicationController patterns, concerns and ActionController::API

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login
  helper_method :current_user, :logged_in?     # make these callable from views

  private

  def current_user
    return unless session[:user_id]
    @current_user ||= User.find_by(id: session[:user_id])  # memoized: one query per request
  end

  def logged_in?
    current_user.present?
  end

  def require_login
    redirect_to login_path, alert: "Please sign in." unless logged_in?
  end
end
```

`ActionController::Base` enables CSRF protection by default: non-GET requests must carry the authenticity token that `form_with` and `csrf_meta_tags` provide (see [forms](07-views-erb-forms-assets.md)).

Shared controller behavior goes into a **concern**:

```ruby
# app/controllers/concerns/store_scoped.rb
module StoreScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_current_store
    helper_method :current_store
  end

  private

  def set_current_store
    @current_store = current_user.stores.find(params[:store_id])
  end

  def current_store
    @current_store
  end
end

class ListingsController < ApplicationController
  include StoreScoped
end
```

**ActionController::API** is a lighter base class for JSON endpoints: no view layouts, cookies, flash or CSRF protection by default. Authenticate with a token header instead of a session, and include modules back only when needed (for example `ActionController::Cookies`). A common layout is an `Api::V1::BaseController < ActionController::API` with its own `rescue_from` handlers returning JSON errors. Details in [API development](10-api-development.md).

## Common pitfalls
- Defining a custom route after `resources` so it is swallowed by `/:id` (`/products/featured` hits `show`).
- Nesting routes three levels deep instead of using `shallow: true`.
- Loading records with `Order.find(params[:id])` instead of `current_user.orders.find(...)`, an insecure direct object reference that lets users read others' data by changing the ID.
- Permitting sensitive attributes (`:role`, `:store_id`, `:price` on an order) because "the form does not show that field anyway".
- Returning 200 for a failed form submission in a Turbo app: the page looks frozen. Return 422.
- Code after `render` or `redirect_to` still runs; forgetting `return` leads to double renders or unintended updates.
- `flash` with `render` (message appears twice) or `flash.now` with `redirect_to` (never appears).
- Storing large objects in the session and hitting the cookie size limit (`ActionDispatch::Cookies::CookieOverflow`).
- Fat controllers: business logic, emails and API calls inside actions. Move them to models, service objects or jobs (see [architecture patterns](12-architecture-patterns.md)).

## How to talk about this in an interview
- Routing and controllers are where you start every ticket in a legacy monolith: find the route with `bin/rails routes -g`, read the controller's filters (inherited ones too), then the action. Say that out loud; it shows a method.
- For Colochousing, the web app and the REST API live in one Rails 7 app, which is a good hook for `namespace :api`, JSON `respond_to` blocks and `ActionController::API`. For CrystalCommerce, store-scoped controllers and strong params on product and listing forms are natural examples.
- If asked about Turbo status codes, connect them to the Visnext Hotwire training: 422 on failed forms and 303 after deletes.
- Security framing works well: scoping lookups to `current_user`, strong params and CSRF tokens are three separate defenses.

> **Fill in:** one controller or endpoint you built or changed at Devntech, what filters and params it used, and how it was tested.

> **Fill in:** whether the Colochousing API is namespaced and versioned (for example `/api/v1`) and how mobile clients authenticate.

## Interview questions

**Q1. What routes does `resources :products` generate?**
Seven: `GET /products` (index), `GET /products/new` (new), `POST /products` (create), `GET /products/:id` (show), `GET /products/:id/edit` (edit), `PATCH/PUT /products/:id` (update), `DELETE /products/:id` (destroy), with helpers like `products_path` and `edit_product_path(product)`.
*Follow-up:* How do you see them? `bin/rails routes -g product`.

**Q2. What is the difference between member and collection routes?**
A member route acts on one record and includes `:id` (`PATCH /orders/:id/cancel`); a collection route acts on the whole set (`GET /orders/search`).
*Follow-up:* An alternative to custom actions? Model the action as its own resource, such as `resource :cancellation, only: :create` handled by `Orders::CancellationsController`.

**Q3. Why avoid deep nesting, and what does `shallow: true` do?**
Deep nesting produces long URLs and helpers that need every parent ID, even though a child record already knows its parent. `shallow: true` nests only index, new and create, and gives show, edit, update and destroy flat URLs like `/products/:id`.
*Follow-up:* How deep should you nest? At most one level.

**Q4. What is the difference between `namespace`, `scope` and `module` in routes?**
`namespace :admin` adds the `/admin` URL prefix, the `Admin::` controller module and the `admin_` helper prefix. `scope "/admin"` changes only the URL; `scope module: :admin` changes only the controller module.
*Follow-up:* How would you version an API? Nest `namespace :v1 do ... end` inside `namespace :api do ... end`, mapping to `Api::V1::` controllers and `/api/v1/...` URLs.

**Q5. When do you use `*_path` vs `*_url`?**
`_path` gives a relative path for links and redirects inside the app; `_url` gives an absolute URL for emails, webhooks, external links and `Location` headers. Mailers need `default_url_options[:host]` to build URLs.
*Follow-up:* What does `redirect_to @product` use? `polymorphic_url`, which resolves to `product_url(@product)`.

**Q6. What are strong parameters, and why do they exist?**
They whitelist which request params may be mass-assigned to a model: `params.require(:product).permit(:name, :price)`. Without them, an attacker could add fields such as `admin` or `store_id` to a form and set them. Unfiltered params passed to `new` or `update` raise `ForbiddenAttributesError`; unpermitted keys are dropped.
*Follow-up:* What did Rails 8 add? `params.expect(product: [:name, :price])`, which also validates the shape and returns 400 instead of a 500 for tampered input.

**Q7. Where do params come from, and what type are their values?**
Path segments, the query string and the request body, merged into one `ActionController::Parameters` object. Form and query values are strings; JSON bodies keep their types.
*Follow-up:* What does `require` do when the key is missing? Raises `ActionController::ParameterMissing`, rendered as 400 Bad Request.

**Q8. How do before_action filters halt a request?**
If a filter renders, redirects or calls `head`, Rails stops the chain and the action never runs; that is how authentication filters work. Use `only:`/`except:` to scope them and `skip_before_action` to remove inherited ones.
*Follow-up:* When would you use an around_action? To wrap the action, for example in `Time.use_zone` or a timing block; it must yield.

**Q9. What is the difference between `render` and `redirect_to`?**
`render` builds the response from a template in the same request, keeping instance variables and the current URL. `redirect_to` returns a 3xx with a `Location` header, and the browser makes a new GET, so instance variables are lost. Render the form again with 422 when validation fails; redirect after a successful write so refreshing does not resubmit.
*Follow-up:* What is a DoubleRenderError? Rendering or redirecting twice in one action; neither call returns, so use `return redirect_to(...)`.

**Q10. Why does Turbo need 422 on failed forms and 303 after deletes?**
Turbo submits forms with `fetch` and only renders a form response with a 4xx/5xx status; a 200 is ignored. After success it follows the redirect, and 303 guarantees the follow-up is a GET rather than repeating a DELETE or PATCH.
*Follow-up:* What happens if you return 200 with errors? The page appears frozen and the errors never show.

**Q11. What is the difference between `flash` and `flash.now`?**
`flash` is stored in the session and shown on the next request, so it pairs with `redirect_to`. `flash.now` exists only for the current request, so it pairs with `render`. Mixing them shows a message twice or never.
*Follow-up:* How do you keep a flash for one more request? `flash.keep`.

**Q12. Where is the session stored by default, and what are the trade-offs?**
In a signed and encrypted cookie (`CookieStore`): no server storage and easy scaling, but about 4 KB of space and no server-side revocation of a single session. Server-side stores (cache, Redis, database) allow larger data and revocation.
*Follow-up:* Signed vs encrypted cookies? Signed cookies are readable but tamper-proof; encrypted ones are unreadable and tamper-proof.

**Q13. How do you return consistent 404s, including for JSON clients?**
`rescue_from ActiveRecord::RecordNotFound, with: :not_found` in `ApplicationController` (or an API base controller), with a `respond_to` that renders an HTML page or a JSON error and `status: :not_found`.
*Follow-up:* In what order are handlers matched? The most recently declared matching handler wins, so declare general ones first.

**Q14. How do you implement `current_user`?**
A memoized private method in `ApplicationController` that loads the user from `session[:user_id]` (or a token for APIs), exposed to views with `helper_method`, plus a `require_login` before_action that redirects when nobody is signed in.
*Follow-up:* Why memoize? To avoid querying the user several times per request.

**Q15. How do you serve HTML, JSON and Turbo Streams from one action?**
`respond_to do |format| ... end` with `format.html`, `format.json { render json: ... }` and `format.turbo_stream` (which renders `action.turbo_stream.erb`). Rails picks the format from the `Accept` header or the URL extension.
*Follow-up:* What happens with an unsupported format? `ActionController::UnknownFormat`, a 406 response.

**Q16. What is `ActionController::API`, and when do you use it?**
A slimmer base controller for JSON endpoints without layouts, cookies, flash or CSRF protection. Use it for API-only apps or an `Api::V1::BaseController` inside a full app serving mobile or SPA clients.
*Follow-up:* How do API clients authenticate without sessions? Usually a bearer token in the `Authorization` header, checked in a before_action.

## Further reading
- [Rails Routing from the Outside In](https://guides.rubyonrails.org/routing.html)
- [Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html)
- [Layouts and Rendering in Rails](https://guides.rubyonrails.org/layouts_and_rendering.html)
