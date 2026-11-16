# Authentication and Authorization in Rails

> **Tier:** T1 (know cold) · **Related:** [Authentication and authorization concepts](../04-web-fundamentals/06-authentication-authorization.md), [Web security (OWASP)](../04-web-fundamentals/07-web-security-owasp.md), [API development](10-api-development.md), [Routing and controllers](02-routing-and-controllers.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [Multi-tenant platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md), [Expo state, storage and auth](../08-react-native-expo/04-state-storage-auth-offline.md)
> **On your resume:** sign-in and permissions across legacy Rails monoliths at Devntech, token auth for the Coloc resident app against the Colochousing Rails API, org-scoped access control in the multi-tenant platform migration, and role-based dashboards in the PureLogics LMS.

## Quick revision
- **Authentication (authN)** = who are you (failure: 401 or redirect to sign-in). **Authorization (authZ)** = what may you do (failure: 403, or 404 to hide the record).
- `has_secure_password` (needs the bcrypt gem and a `password_digest` column) adds `password`/`password_confirmation`, presence and 72-byte length validations, and `authenticate`. bcrypt is slow, salted and has a tunable cost, which is why it beats SHA-256 for passwords.
- Rails 7.1: `User.authenticate_by(email:, password:)` (timing-safe lookup), `generates_token_for` / `find_by_token_for` (purpose-bound expiring tokens), `normalizes`.
- Rails 8.0 `bin/rails generate authentication`: User, database-backed Session, `Current`, an `Authentication` concern, sessions and password-reset controllers and mailer. **No sign-up**, confirmation, OAuth, 2FA, API tokens or roles.
- **Devise** = Warden-based engine. Modules: database_authenticatable, registerable, recoverable, rememberable, validatable, confirmable, lockable, trackable, timeoutable, omniauthable. Helpers: `authenticate_user!`, `current_user`, `user_signed_in?`.
- Customize Devise with `devise_parameter_sanitizer.permit`, generated views (`devise:views`) and controllers (`devise:controllers users` + `devise_for :users, controllers: {...}`).
- Mobile auth: short-lived access token (minutes) + long-lived refresh token (weeks) rotated on every use, both stored as digests per device; revoke by deleting the device row; store on the phone in SecureStore.
- Opaque tokens revoke instantly with a DB lookup; JWTs verify without the DB but need a denylist or allowlist to revoke (devise-jwt offers both). Doorkeeper when you must be an OAuth 2 provider for third-party clients.
- OmniAuth: request phase must be POST (add omniauth-rails_csrf_protection); callback gives `request.env["omniauth.auth"]` with `provider` + `uid`; store identities, link by email only if verified.
- **Pundit:** one policy class per model with `user` and `record`; `authorize @listing` calls `update?` for the update action; `policy_scope(Listing)` uses `ListingPolicy::Scope#resolve`; `verify_authorized` / `verify_policy_scoped` after_actions catch forgotten checks.
- **CanCanCan:** central `Ability` with `can :update, Listing, organization_id: ...`; `authorize!`, `can?`, `load_and_authorize_resource`, `accessible_by(current_ability)`.
- Org-scoped RBAC: `users` -> `memberships(role)` -> `organizations`. The role lives on the membership because one person can be an admin in one org and a member in another.
- Tenant scoping: `Current.organization` + scoped queries, or acts_as_tenant (`acts_as_tenant :organization`, `ActsAsTenant.with_tenant` in jobs). Jobs do not inherit the request's tenant.
- **IDOR:** `Order.find(params[:id])` lets anyone read any order by changing the ID. Load through the owner (`current_user.orders.find`) or a policy scope.

## 1. Authentication vs authorization

| | Authentication | Authorization |
|---|---|---|
| Question | Who is making this request? | Is this user allowed to do this to this record? |
| Rails tools | `has_secure_password`, the Rails 8 generator, Devise, tokens, OmniAuth | Pundit, CanCanCan, scoped queries, roles on memberships |
| Where in the request | `before_action :authenticate_user!` | Inside the action: `authorize @record`, `policy_scope` |
| Failure | 401 (API) or redirect to sign-in (HTML) | 403, or 404 when existence itself is secret |

They are separate layers: a perfectly authenticated user can still read data they should not if authorization is missing. Concepts such as sessions vs tokens, OAuth flows and hashing are covered in [authentication concepts](../04-web-fundamentals/06-authentication-authorization.md).

## 2. has_secure_password and bcrypt

```ruby
# Gemfile: gem "bcrypt"; users table: email (unique index), password_digest
class User < ApplicationRecord
  has_secure_password                                   # password, password_confirmation, authenticate
  normalizes :email, with: ->(email) { email.strip.downcase }    # Rails 7.1+

  generates_token_for :email_confirmation, expires_in: 24.hours do
    email                                               # the token dies if the email changes
  end
end

user = User.create!(email: "a@example.com", password: "long-pass-1", password_confirmation: "long-pass-1")
user.password_digest              # "$2a$12$...": algorithm, cost, salt and hash in one string
user.authenticate("wrong")        # => false
user.authenticate("long-pass-1")  # => user
User.authenticate_by(email: "A@example.com ", password: "long-pass-1")   # Rails 7.1+: user or nil

token = user.generate_token_for(:email_confirmation)
User.find_by_token_for(:email_confirmation, token)     # nil when expired, tampered with or stale
```

- **Why bcrypt:** a fast hash like SHA-256 lets an attacker with a leaked table try billions of guesses per second on GPUs. bcrypt is deliberately slow, salts every hash (identical passwords get different digests) and has a cost factor you raise as hardware improves.
- bcrypt only uses the first 72 bytes, so `has_secure_password` validates that maximum.
- `authenticate_by` hashes the password even when no user matches, so response time does not reveal which emails are registered. `User.find_by(email:)&.authenticate(pw)` returns faster for unknown emails.
- Rails 8.0+ also gives `has_secure_password` models `user.password_reset_token` and `User.find_by_password_reset_token!(token)`, short-lived tokens built on `generates_token_for`; the auth generator uses them.

## 3. The Rails 8 authentication generator

`bin/rails generate authentication` (Rails 8.0+) writes plain code into your app instead of hiding it in a gem:

```ruby
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy
  normalizes :email_address, with: ->(e) { e.strip.downcase }
end

class Session < ApplicationRecord             # one row per signed-in browser: user_id, ip_address, user_agent
  belongs_to :user
end

class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end

# app/controllers/concerns/authentication.rb (trimmed), included in ApplicationController
def start_new_session_for(user)
  user.sessions.create!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
    Current.session = session
    cookies.signed.permanent[:session_id] = { value: session.id, httponly: true, same_site: :lax }
  end
end

# app/controllers/sessions_controller.rb (trimmed)
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[new create]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_url, alert: "Try again later." }

  def create
    if (user = User.authenticate_by(params.permit(:email_address, :password)))
      start_new_session_for(user)
      redirect_to after_authentication_url
    else
      redirect_to new_session_path, alert: "Try another email address or password."
    end
  end
end
```

- Every controller requires a session by default (`before_action :require_authentication` in the concern); public actions opt out with `allow_unauthenticated_access`.
- Sessions live in the database and the cookie holds only a signed session ID, so you can list a user's devices, revoke one session, or sign out everywhere by destroying rows.
- A `PasswordsController` and `PasswordsMailer` handle resets by email.
- **Not included:** sign-up (you write a `RegistrationsController` yourself), email confirmation, lockout, OAuth, 2FA, API tokens and authorization. Exact generated code can change between versions; read what your version produces.

## 4. Devise

Devise is the long-time standard and appears in most legacy Rails apps. It is an engine built on **Warden**, a Rack middleware that keeps authentication state in the session.

How a request flows: `before_action :authenticate_user!` calls `warden.authenticate!(scope: :user)`. Warden first looks in the session (`warden.user.user.key`), then runs strategies (password from a sign-in form, remember-me cookie). On success the user is stored in the session and `current_user` returns it; on failure Warden throws, and Devise's failure app redirects HTML requests to the sign-in page or returns 401 for JSON.

| Module | What it adds |
|---|---|
| database_authenticatable | bcrypt-hashed password, sign-in with email and password |
| registerable | Sign up, edit and delete account |
| recoverable | Password reset by emailed token |
| rememberable | "Remember me" cookie |
| validatable | Email format and password length validations |
| confirmable | Must confirm email before signing in (optional grace period) |
| lockable | Lock after N failed attempts; unlock by email or after a delay |
| trackable | Sign-in count, timestamps and IPs |
| timeoutable | Expire sessions after inactivity |
| omniauthable | OmniAuth callbacks for social login |

```ruby
# bin/rails generate devise:install, then: bin/rails generate devise User
class User < ApplicationRecord
  devise :database_authenticatable, :registerable, :recoverable, :rememberable,
         :validatable, :confirmable, :lockable, :trackable
end

# config/routes.rb: /users/sign_in, /users/sign_up, /users/password/new, ...
devise_for :users, controllers: { registrations: "users/registrations" }   # optional custom controller

class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  before_action :configure_permitted_parameters, if: :devise_controller?

  private

  def configure_permitted_parameters
    devise_parameter_sanitizer.permit(:sign_up, keys: %i[first_name last_name])
  end

  def after_sign_in_path_for(resource)
    stored_location_for(resource) || dashboard_path
  end
end
```

- `bin/rails generate devise:views` copies the views into your app to edit; `bin/rails generate devise:controllers users` creates subclasses like `Users::RegistrationsController < Devise::RegistrationsController`, wired with the `controllers:` option above.
- Helpers: `current_user`, `user_signed_in?`, `sign_in(user)`, `sign_out`, `user_session`; route helpers like `new_user_session_path`.
- With Turbo, use Devise 4.9+ and its responder settings (`config.responder.error_status = :unprocessable_entity`, `config.responder.redirect_status = :see_other`), and sign out with `button_to ... method: :delete`.

| | Rails 8 generator | Devise |
|---|---|---|
| What you get | Readable code you own: sessions and password reset | An engine with ten modules, routes, views and mailers |
| Sign-up, confirmation, lockout, OAuth | Build them yourself | Turn on modules |
| Customizing | Edit your code | Override controllers and views, configuration, Warden hooks |
| Best for | New apps with simple needs, learning | Legacy apps, apps needing many standard features fast |

## 5. API and mobile authentication

A mobile app cannot rely on the browser cookie session, so it sends a token in the `Authorization` header. The common design for a monolith's mobile API:

| Token | Lifetime | Server stores | Sent |
|---|---|---|---|
| Access | 15 to 60 minutes | SHA-256 digest (or nothing, for JWT) | On every request |
| Refresh | 30 to 90 days, replaced on each use | Digest, device name, expiry, `revoked_at` | Only to the refresh endpoint |

```ruby
class DeviceSession < ApplicationRecord      # one row per signed-in phone
  belongs_to :user
  # access_token_digest, access_expires_at, refresh_token_digest, refresh_expires_at, device_name, revoked_at

  def self.digest(token) = Digest::SHA256.hexdigest(token)

  def rotate!
    access, refresh = SecureRandom.urlsafe_base64(32), SecureRandom.urlsafe_base64(32)
    update!(access_token_digest: self.class.digest(access), access_expires_at: 15.minutes.from_now,
            refresh_token_digest: self.class.digest(refresh), refresh_expires_at: 60.days.from_now)
    { access_token: access, refresh_token: refresh, expires_in: 15.minutes.to_i }   # plain tokens leave once
  end
end

# POST /api/v1/tokens/refresh
def refresh
  device = DeviceSession.where(revoked_at: nil).where("refresh_expires_at > ?", Time.current)
                        .find_by(refresh_token_digest: DeviceSession.digest(params.require(:refresh_token)))
  return render_error(:unauthorized, "invalid_refresh_token", "Please sign in again") unless device
  render json: device.rotate!                 # the old refresh token stops working immediately
end
```

- **Why two tokens:** a stolen access token works only for minutes; the refresh token travels rarely and can be revoked per device.
- **Revocation:** sign-out sets `revoked_at` on that device; "sign out everywhere" and password changes revoke all of the user's rows. For reuse detection, keep the previous refresh digest: if it is presented again after rotation, someone copied it, so revoke the device session.
- **Digest lookup:** SHA-256 is fine for long random tokens (bcrypt is for low-entropy passwords and cannot be looked up by value).
- **On the device:** Expo SecureStore (iOS Keychain, Android Keystore), never AsyncStorage; refresh once at a time (parallel 401s should wait on a single refresh call, or the second refresh fails after rotation); clear tokens on sign-out. See [state, storage and auth in Expo](../08-react-native-expo/04-state-storage-auth-offline.md).

| | Opaque token in a table | JWT |
|---|---|---|
| Checking a request | DB lookup by digest | Verify the signature; no DB needed |
| Revocation | Immediate: update the row | Hard: wait for expiry, or check a denylist or allowlist (a DB lookup again) |
| Payload | None; the server looks up the user | Readable claims (base64, not encrypted); never put secrets in it |
| Fits | A monolith and its mobile app | Several services verifying tokens independently |

With Devise, **devise-jwt** issues a JWT in the `Authorization` response header on sign-in and revokes it on sign-out:

```ruby
class User < ApplicationRecord
  devise :database_authenticatable, :jwt_authenticatable, jwt_revocation_strategy: JwtDenylist
end

class JwtDenylist < ApplicationRecord
  include Devise::JWT::RevocationStrategies::Denylist   # also available: JTIMatcher, Allowlist
  self.table_name = "jwt_denylist"
end
# config/initializers/devise.rb: config.jwt { |jwt| jwt.secret = ENV.fetch("DEVISE_JWT_SECRET_KEY"); jwt.expiration_time = 15.minutes.to_i }
```

**Doorkeeper** turns your app into an OAuth 2 authorization server. You need it when third parties build clients against your API ("Sign in with YourApp", partner integrations, scoped permissions): authorization code with PKCE for mobile and SPAs, client credentials for server-to-server. Protect endpoints with `before_action :doorkeeper_authorize!` and read `doorkeeper_token.resource_owner_id`. For your own first-party app, simple tokens are usually enough.

## 6. OmniAuth for social login

```ruby
# Gemfile: gem "omniauth-google-oauth2"; gem "omniauth-rails_csrf_protection"
# config/initializers/omniauth.rb (without Devise; with Devise use :omniauthable instead)
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :google_oauth2, ENV.fetch("GOOGLE_CLIENT_ID"), ENV.fetch("GOOGLE_CLIENT_SECRET")
end

# config/routes.rb: get "/auth/:provider/callback", to: "omniauth_callbacks#create"
class OmniauthCallbacksController < ApplicationController
  def create
    auth = request.env["omniauth.auth"]                     # provider, uid, info.email, credentials
    identity = Identity.find_or_initialize_by(provider: auth.provider, uid: auth.uid)
    identity.user ||= User.find_or_create_from_oauth!(auth) # link by email only if the provider verified it
    identity.save!
    start_new_session_for(identity.user)
    redirect_to root_path
  end
end
```

```erb
<%= button_to "Sign in with Google", "/auth/google_oauth2", data: { turbo: false } %>
```

- OmniAuth 2 rejects GET requests to `/auth/:provider` (a login CSRF fix), so the button must POST with a CSRF token; Turbo is disabled because the response redirects to another origin.
- Key identities on `provider` + `uid` (unique index), not on email: emails change, and one user can link several providers.
- Auto-linking a Google login to an existing account by email is only safe when the provider says the email is verified; otherwise attackers can claim accounts.
- Mobile social login usually happens natively: the app gets an ID token from the provider SDK and posts it to your API, which verifies its signature and audience before issuing your own tokens.

## 7. Authorization with Pundit

```ruby
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  after_action :verify_authorized, except: :index        # raises if an action forgot `authorize`
  after_action :verify_policy_scoped, only: :index       # raises if index forgot `policy_scope`
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  private

  def forbidden
    respond_to do |format|
      format.html { redirect_back fallback_location: root_path, alert: "Not allowed." }
      format.json { render json: { error: { code: "forbidden" } }, status: :forbidden }
    end
  end
end

class ListingPolicy < ApplicationPolicy                  # user and record come from ApplicationPolicy
  def show?    = membership.present?
  def update?  = %w[manager admin].include?(membership&.role)
  def destroy? = membership&.role == "admin"

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.where(organization_id: user.memberships.select(:organization_id))
    end
  end

  private

  def membership
    @membership ||= user.memberships.find_by(organization_id: record.organization_id)
  end
end

class ListingsController < ApplicationController
  def index
    @listings = policy_scope(Listing).order(created_at: :desc)
  end

  def update
    @listing = policy_scope(Listing).find(params[:id])   # other orgs' records: 404
    authorize @listing                                   # calls ListingPolicy#update?: 403 if false
    @listing.update!(permitted_attributes(@listing))     # optional: policy decides permitted params
    redirect_to @listing
  end
end
```

- `authorize @listing` infers the query from the action (`update` -> `update?`); `authorize @listing, :publish?` names it.
- In views: `<% if policy(@listing).update? %>`. Hiding a button is UX, not security; the controller check is the security.
- The `verify_*` after_actions run after the action, so they are a safety net that makes tests fail loudly, not a runtime guard.
- Policies are plain Ruby objects, so they are trivial to unit-test.

## 8. CanCanCan, and choosing between them

```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    return unless user
    can :read, Listing, organization_id: user.organization_ids
    can :manage, Listing, organization_id: user.memberships.where(role: :admin).pluck(:organization_id)
  end
end

class ListingsController < ApplicationController
  load_and_authorize_resource            # index: Listing.accessible_by(current_ability); member actions: find + authorize!
  rescue_from CanCan::AccessDenied do
    redirect_to root_path, alert: "Not allowed."
  end
end
# Elsewhere: authorize! :update, @listing   |   in views: can?(:update, @listing)
```

| | Pundit | CanCanCan |
|---|---|---|
| Where rules live | One policy class per model, plain methods | One `Ability` class (can be split) |
| Style | Explicit `authorize` and `policy_scope` calls | `can`/`cannot` DSL plus `load_and_authorize_resource` |
| Listing records | `Policy::Scope#resolve` | `accessible_by(current_ability)` from the same rules |
| As the app grows | Many small files, scales well | The Ability file tends to become huge |
| Forgotten checks | `verify_authorized`, `verify_policy_scoped` | `check_authorization` |

Pick Pundit for new code when rules are contextual (roles per organization, record state); CanCanCan is common in older apps and fine for compact rule sets. Either way, the rule is: every action authorizes, every list is scoped.

## 9. Role-based access with memberships

```ruby
class Organization < ApplicationRecord
  has_many :memberships, dependent: :destroy
  has_many :users, through: :memberships
end

class Membership < ApplicationRecord
  belongs_to :user
  belongs_to :organization
  enum :role, { member: 0, manager: 1, admin: 2, owner: 3 }    # explicit integers never shift
  validates :user_id, uniqueness: { scope: :organization_id }  # plus a unique index on both columns

  def at_least?(role) = Membership.roles.fetch(self.role) >= Membership.roles.fetch(role.to_s)
end

class Current < ActiveSupport::CurrentAttributes
  attribute :user, :organization, :membership
end

class ApplicationController < ActionController::Base
  before_action :set_current_organization

  private

  def set_current_organization                             # the org comes from the URL, a header or a subdomain
    Current.membership = current_user.memberships.joins(:organization)
                                     .find_by!(organizations: { slug: params[:org_slug] })  # not a member: 404
    Current.organization = Current.membership.organization
  end
end
```

- The role belongs on the membership, not the user: one person can own one organization and be a plain member of another. Platform staff get a separate flag or table, never a fake membership everywhere.
- Never trust an `organization_id` sent by the client without checking membership; deriving it from the membership lookup does both at once.
- Policies then read `Current.membership.at_least?(:manager)`; with Pundit you can override `pundit_user` to pass a context object holding user, organization and membership.
- `Current` is reset per request. Background jobs do not inherit it: pass `organization_id` explicitly and set the tenant inside the job.

## 10. Multi-tenant scoping

Two common approaches for row-level tenancy (one database, an `organization_id` on every tenant table):

```ruby
# A. Explicit scoping through Current
Current.organization.listings.find(params[:id])

# B. acts_as_tenant gem: automatic scoping
class Listing < ApplicationRecord
  acts_as_tenant :organization                 # belongs_to + a default scope on the current tenant
  validates_uniqueness_to_tenant :sku          # uniqueness within one organization
end

class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action { set_current_tenant(Current.organization) }
end

ActsAsTenant.with_tenant(organization) { Listing.count }   # jobs, console, scripts
```

| | Explicit (`Current.organization.listings`) | acts_as_tenant |
|---|---|---|
| Forgetting the scope | Possible; review and tests must catch it | Hard: queries are scoped automatically |
| Visibility | Every query shows its scope | Magic default scope; `unscoped` or a missing tenant can surprise |
| Jobs and console | Pass the org and scope manually | Wrap work in `with_tenant`; `require_tenant` can raise when none is set |

Postgres row-level security adds a database-enforced layer, and schema-per-tenant (Apartment-style) is another model; trade-offs are in [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md).

## 11. Testing authorization

```ruby
# spec/policies/listing_policy_spec.rb (require "pundit/rspec" in rails_helper)
RSpec.describe ListingPolicy do
  subject { described_class }

  let(:org) { create(:organization) }
  let(:listing) { create(:listing, organization: org) }
  let(:admin) { create(:membership, organization: org, role: :admin).user }
  let(:member) { create(:membership, organization: org, role: :member).user }
  let(:outsider) { create(:user) }

  permissions :update? do
    it { is_expected.to permit(admin, listing) }
    it { is_expected.not_to permit(member, listing) }
    it { is_expected.not_to permit(outsider, listing) }
  end

  it "scopes listings to the user's organizations" do
    create(:listing)                                         # another organization
    expect(Pundit.policy_scope!(member, Listing)).to contain_exactly(listing)
  end
end
```

Add request specs that prove the wiring: another organization's record returns 404, a member trying an admin action gets 403 and nothing changes, and an unauthenticated request gets 401 or a redirect. With CanCanCan, `require "cancan/matchers"` gives `expect(Ability.new(user)).to be_able_to(:update, listing)`.

## 12. Common security mistakes

```ruby
# IDOR: any signed-in user can read any order by editing the URL
@order = Order.find(params[:id])
# Fixed: scope through the owner or a policy scope
@order = current_user.orders.find(params[:id])
@order = policy_scope(Order).find(params[:id])
```

- **IDOR** (insecure direct object reference) is the most common Rails authorization bug, and sequential IDs make it trivial to exploit. UUIDs or `signed_id` make guessing harder but are not authorization.
- **Mass-assigning privileges:** permitting `:role`, `:admin` or `:organization_id` in strong params.
- **Trusting client-supplied tenant IDs** in params, headers or JSON bodies without a membership check.
- **UI-only authorization:** hiding a button while the endpoint stays open.
- **Unscoped writes:** `Listing.where(id: params[:ids]).update_all(...)` instead of going through the scope.
- **Account enumeration:** different messages or timings for "unknown email" and "wrong password" (use `authenticate_by` and neutral reset messages).
- **Tokens in URLs and logs:** query strings end up in logs and analytics; send tokens in headers and add them to `filter_parameters`.
- **Session fixation:** call `reset_session` on sign-in when rolling your own (Devise and the generator handle it).
- **Long-lived tokens nobody can revoke**, and password changes that leave old sessions alive.

## Common pitfalls
- Confusing 401 and 403, or returning 403 for another tenant's record (it confirms the record exists).
- Storing tokens in plain text in the database, or on the phone in AsyncStorage.
- Building JWT auth for a single monolith and then discovering you cannot sign anyone out.
- Forgetting `verify_authorized` / `verify_policy_scoped`, so new actions ship without checks.
- Putting the role on `users` in a multi-organization app.
- Background jobs that assume `Current.organization` is set, or that skip authorization for IDs taken from user input.
- Linking OAuth accounts by unverified email.
- Overriding Devise controllers wholesale instead of using its hooks, which makes upgrades painful.

## How to talk about this in an interview
- Lead with the two-layer framing: authentication identifies, authorization decides, and you check both on every request. Then give the Rails tools for each.
- Org-scoped access control in the multi-tenant migration is your strongest authorization story: memberships with roles, deriving the organization from membership, scoping every query, and testing cross-tenant access. Say clearly which parts you designed and which you implemented.
- For the Coloc resident app, explain how a mobile client authenticates against the Rails API and how tokens are stored and revoked; if the real setup was simpler than access plus refresh tokens, describe it honestly and say what you would improve.
- The PureLogics LMS (DRF and React) had role-based dashboards; mention it as earlier RBAC experience in another framework, mapping it to Pundit concepts.

> **Fill in:** which authentication the Colochousing app uses (Devise, custom tokens, devise-jwt), and what the resident app stores on the phone.

> **Fill in:** how org-scoped access control worked in the migration: roles, where the current organization came from, and how you tested isolation.

> **Fill in:** an authorization or IDOR-style bug you found or fixed in a legacy monolith, and how you prevented it from coming back.

## Interview questions

**Q1. What is the difference between authentication and authorization?**
Authentication establishes who the user is (password, session cookie, token); authorization decides what that user may do with a specific resource (policies, roles, scoped queries). Authentication failures give 401 or a sign-in redirect; authorization failures give 403, or 404 when even existence is secret.
*Follow-up:* Which Rails tools for each? Devise, the Rails 8 generator or tokens for authN; Pundit, CanCanCan and scoped queries for authZ.

**Q2. How does `has_secure_password` work, and why bcrypt?**
It needs a `password_digest` column and the bcrypt gem, adds virtual `password` and `password_confirmation` attributes with validations, stores a salted bcrypt hash, and `authenticate(pw)` returns the user or false. bcrypt is slow and salted with a tunable cost, so leaked hashes resist brute force, unlike SHA-256.
*Follow-up:* What does `authenticate_by` add? A timing-safe lookup that hashes even when the user does not exist, preventing email enumeration.

**Q3. What does the Rails 8 authentication generator give you, and what is missing?**
User and database-backed Session models, `Current`, an `Authentication` concern that requires login by default, sign-in and sign-out, password reset by email, and a rate-limited login. Sign-up, email confirmation, lockout, OAuth, 2FA, API tokens and authorization are left to you.
*Follow-up:* Why database sessions? You can list and revoke sessions per device and sign out everywhere.

**Q4. How does Devise work, and what are its main modules?**
Devise is an engine on top of Warden, a Rack middleware that stores the authenticated user in the session and runs strategies (password, remember-me) when there is none. `authenticate_user!` calls `warden.authenticate!`, and the failure app redirects or returns 401. Modules: database_authenticatable, registerable, recoverable, rememberable, validatable, confirmable, lockable, trackable, timeoutable, omniauthable.
*Follow-up:* How do you add a field to sign-up? `devise_parameter_sanitizer.permit(:sign_up, keys: [:first_name])` in a before_action for Devise controllers.

**Q5. When would you choose Devise over the Rails 8 generator?**
Devise when you need many standard features quickly (confirmation, lockout, OAuth) or the app already uses it; the generator when needs are simple and you want code you fully own and understand.
*Follow-up:* Biggest Devise risk? Heavy controller overrides that break on upgrades; prefer configuration and hooks.

**Q6. How do you secure an API consumed by a mobile app?**
HTTPS only; sign-in returns a short-lived access token and a long-lived refresh token, both stored as digests per device; the access token goes in the `Authorization` header and expires in minutes; refresh rotates both tokens, and reuse of an old refresh token revokes the device; sign-out and password changes revoke rows; the app keeps tokens in SecureStore. Add rate limiting on sign-in and refresh, authorization on every endpoint, and a minimum app version.
*Follow-up:* What happens when two requests hit an expired token at once? The app should queue them behind a single refresh call.

**Q7. JWT or opaque tokens, and how do you revoke a JWT?**
Opaque tokens need a DB lookup but revoke instantly; JWTs verify by signature without a lookup but stay valid until expiry. Revoke JWTs with short expiry plus a denylist, a JTI matcher or an allowlist (devise-jwt supports all three), which reintroduces a lookup.
*Follow-up:* Is a JWT payload private? No, it is base64-encoded JSON; anyone can read it.

**Q8. When do you need Doorkeeper?**
When your app must act as an OAuth 2 provider: third-party clients, delegated and scoped access, authorization code with PKCE, client credentials between services. For your own mobile app talking to your own API, simple tokens are usually enough.
*Follow-up:* Which grant fits a mobile app? Authorization code with PKCE.

**Q9. How does social login with OmniAuth work, and what can go wrong?**
The user POSTs to `/auth/:provider`, is redirected to the provider, and returns to your callback where `request.env["omniauth.auth"]` holds provider, uid and profile; you find or create an identity keyed on provider and uid and sign the user in. Risks: GET-based request phase (login CSRF), linking accounts by unverified email, and keying on email instead of uid.
*Follow-up:* Why omniauth-rails_csrf_protection? It enforces the POST plus CSRF token that OmniAuth 2 requires.

**Q10. How do Pundit policies and scopes work?**
A policy is a plain class per model initialized with `user` and `record`, with predicate methods like `update?`. `authorize @record` calls the predicate matching the action and raises `Pundit::NotAuthorizedError` when false. `policy_scope(Model)` calls `Policy::Scope#resolve`, which returns only records the user may see, for example those in the user's organizations.
*Follow-up:* How do you ensure nobody forgets? `after_action :verify_authorized` and `verify_policy_scoped`, which raise in tests.

**Q11. Pundit or CanCanCan?**
Pundit keeps rules in small per-model policy classes with explicit calls and scales well with contextual rules; CanCanCan centralizes rules in an Ability DSL with automatic loading, which is quick for small apps but grows into a huge file. Both can derive list queries from the rules.
*Follow-up:* CanCanCan's equivalent of `policy_scope`? `Model.accessible_by(current_ability)`.

**Q12. Design access control for users who belong to several organizations with different roles.**
Users, organizations and memberships with a role enum and a unique index on user and organization. Resolve the current organization from the URL or header through the user's memberships (404 if not a member), keep it in `Current`, scope every query by it, and check roles in policies. Platform admins are a separate concept.
*Follow-up:* Why not a `role` column on users? One person can hold different roles in different organizations.

**Q13. How do you enforce tenant isolation in a Rails app?**
Every tenant table has `organization_id`; queries go through `Current.organization` associations or acts_as_tenant's automatic scope; uniqueness is scoped per tenant; jobs receive the organization ID and set the tenant; tests assert cross-tenant access fails. Postgres row-level security is an extra database-enforced layer.
*Follow-up:* Main risk with acts_as_tenant? Work outside a request (jobs, console) running without a tenant or with `unscoped`.

**Q14. What is IDOR, and how do you prevent it in Rails?**
An insecure direct object reference: the app loads a record by an ID from the request without checking ownership, so changing `/orders/41` to `/orders/42` exposes someone else's order. Prevent it by loading through the owner (`current_user.orders.find`), a policy scope or the tenant, plus a policy check for the action, and test it.
*Follow-up:* Do UUIDs fix it? They make guessing harder, but leaked IDs still work; they are not authorization.

**Q15. How do you test authorization?**
Unit-test each policy across roles (Pundit's `permissions`/`permit` matchers or plain method calls) and test scopes return only allowed records; then request specs proving another tenant's record returns 404, forbidden actions return 403 without side effects, and unauthenticated requests are rejected.
*Follow-up:* Why both levels? Policy specs prove the rules; request specs prove every controller actually applies them.

**Q16. How do you stop account enumeration and brute force on login?**
Same error message and similar timing for unknown email and wrong password (`authenticate_by`), neutral password-reset responses, rate limiting by IP and account (`rate_limit` or rack-attack), and lockable accounts or increasing delays.
*Follow-up:* Why not lock accounts immediately? Attackers can lock out real users; combine rate limits with notifications.

## Further reading
- [Securing Rails Applications](https://guides.rubyonrails.org/security.html)
- [ActiveModel::SecurePassword](https://api.rubyonrails.org/classes/ActiveModel/SecurePassword/ClassMethods.html)
- [Devise](https://github.com/heartcombo/devise) and [Pundit](https://github.com/varvet/pundit)
