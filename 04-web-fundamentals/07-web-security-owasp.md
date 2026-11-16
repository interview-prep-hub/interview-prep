# Web Security and the OWASP Top 10

> **Tier:** T1 (know cold) · **Related:** [Authentication and authorization](06-authentication-authorization.md), [How the web works](01-how-the-web-works.md), [REST API design](04-rest-api-design.md), [Rails caching, performance and security](../05-ruby-on-rails/14-caching-performance-security.md), [Browser DOM and events](../03-languages/javascript/06-browser-dom-events.md), [Git](../14-devops-and-tools/01-git.md), [GitHub and workflows](../14-devops-and-tools/02-github-and-workflows.md)
> **On your resume:** the legacy Rails e-commerce platform at CrystalCommerce (700+ stores, 8M+ customers), the Colochousing platform for a Belgian client (EU personal data, rental applications, rent payments), third-party integrations such as PriceCharting, and REST APIs at Devntech.

## Quick revision
- **OWASP Top 10 (2021):** A01 Broken Access Control, A02 Cryptographic Failures, A03 Injection (includes XSS), A04 Insecure Design, A05 Security Misconfiguration, A06 Vulnerable and Outdated Components, A07 Identification and Authentication Failures, A08 Software and Data Integrity Failures, A09 Security Logging and Monitoring Failures, A10 Server-Side Request Forgery. OWASP published a newer edition in 2025; check owasp.org for its list.
- **XSS** = the attacker's script runs in your page. Stored (saved and shown to others), reflected (echoed from the request), DOM-based (client JS writes untrusted data into the DOM).
- Prevent XSS: auto-escaping templates (ERB `<%= %>`, JSX), no `raw`/`html_safe`/`dangerouslySetInnerHTML` on user data, allowlist sanitizing for rich text, URL scheme checks, CSP as a second layer.
- **CSP** limits where scripts and other resources may come from; use nonces or hashes instead of `'unsafe-inline'`, and roll out with `Content-Security-Policy-Report-Only`.
- **CSRF** = another site makes the victim's browser send an authenticated request (cookies attach automatically). Defenses: SameSite cookies, CSRF tokens (Rails `protect_from_forgery`), Origin and Fetch Metadata checks, no state changes on GET.
- XSS runs code inside your origin and defeats CSRF defenses; CSRF only forges requests and cannot read responses.
- **SQL injection:** never interpolate input into SQL; use bound parameters or the ORM, and allowlist identifiers such as sort columns.
- **NoSQL injection:** JSON bodies can smuggle operators (`{"$ne": null}`), so validate types with a schema. **Command injection:** never pass input to a shell; use array-form `system` or `execFile`.
- **Clickjacking:** forbid framing with `frame-ancestors 'none'` (CSP) and `X-Frame-Options: DENY`.
- **CORS misconfiguration:** reflecting any origin with credentials lets any site read authenticated responses; CORS is not CSRF protection.
- **Mass assignment:** allowlist writable attributes; never accept `role` or `organization_id` from clients.
- **Open redirect:** redirect only to relative paths or allowlisted hosts.
- **File uploads:** check type by content, cap size, rename, store privately, serve from another domain or as attachments, send `nosniff`.
- **SSRF:** the server fetches an attacker-chosen URL (internal services, cloud metadata at `169.254.169.254`); allowlist destinations, block private ranges after DNS resolution, re-check redirects.
- **Secrets:** environment variables or a secret store; never committed, logged or bundled into frontend code. A leaked key must be rotated; rewriting git history is not enough.
- HTTPS everywhere with **HSTS**; headers: CSP, HSTS, `X-Content-Type-Options: nosniff`, `frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`.
- **Brute force:** rate limit per account and per IP, MFA, generic error messages, alerts.
- **Webhooks:** verify an HMAC over the raw body with a constant-time comparison and a timestamp window.
- **Dependencies:** bundler-audit, Brakeman, `npm audit`, Dependabot, committed lockfiles, regular upgrades.
- **GDPR** (EU clients): lawful basis, minimization, retention limits, access and erasure rights, breach notification within 72 hours, processor agreements, no personal data in logs.

## 1. The OWASP Top 10 (2021)

| Category | Example | Mitigation |
|---|---|---|
| A01 Broken Access Control | changing `/bills/981` to `/bills/982` shows another resident's bill; members can call an admin endpoint | deny by default, scope queries to the user and org, policies on every action, cross-tenant tests |
| A02 Cryptographic Failures | MD5 password hashes, personal data over HTTP, unencrypted backups | TLS everywhere, bcrypt or argon2, encrypted sensitive columns (Rails `encrypts`), managed keys |
| A03 Injection | `where("email = '#{params[:email]}'")`; a listing title that injects a script | bound parameters, ORM APIs, context-aware output escaping, input validation |
| A04 Insecure Design | unlimited OTP attempts, a coupon redeemable many times, a reset flow that reveals accounts | threat modeling, abuse cases, limits and secure defaults designed in |
| A05 Security Misconfiguration | debug pages in production, default credentials, a public bucket, permissive CORS, verbose errors | hardened defaults, infrastructure as code, configuration reviews, security headers |
| A06 Vulnerable and Outdated Components | an old Rails or Nokogiri with a known CVE; an abandoned npm package | dependency inventory, bundler-audit and `npm audit`, Dependabot, regular upgrades |
| A07 Identification and Authentication Failures | credential stuffing, session IDs not rotated at login, no MFA for admins | rate limiting, MFA, secure sessions, breached-password checks |
| A08 Software and Data Integrity Failures | unverified webhooks, a compromised CI pipeline or package, `Marshal.load` on user data | signatures and checksums, lockfiles, protected CI secrets, safe formats such as JSON |
| A09 Security Logging and Monitoring Failures | brute force goes unnoticed; no audit trail of admin actions | log auth events and admin actions, alert on anomalies, retain logs, incident runbooks |
| A10 Server-Side Request Forgery | "import image from URL" fetches the cloud metadata endpoint | allowlists, blocking internal ranges, isolated egress |

OWASP also maintains an API Security Top 10 (2023 edition, led by broken object-level authorization) and a Mobile Top 10, both useful for API and React Native work.

## 2. Cross-site scripting (XSS)

| Type | How the script gets in | Example |
|---|---|---|
| Stored | saved in the database and rendered for other users | a listing title containing a `<script>` tag, rendered without escaping |
| Reflected | echoed from the current request | a search page printing `?q=<script>...</script>` unescaped |
| DOM-based | client-side code writes untrusted data into the DOM | `el.innerHTML = location.hash.slice(1)` |

Once a script runs in your origin it can read JS-accessible tokens, act as the user (change their email, make payments), log keystrokes or show a fake login form on your real domain.

```erb
<%= listing.title %>                <%# escaped by default: safe %>
<%= raw listing.title %>            <%# not escaped: XSS if a user wrote the title %>
<%= listing.title.html_safe %>      <%# same danger: html_safe means "trust me" %>
<%= sanitize listing.description, tags: %w[p br strong em ul ol li a], attributes: %w[href] %>
```

```tsx
import DOMPurify from "dompurify";

export function Listing({ title, descriptionHtml, website }: { title: string; descriptionHtml: string; website: string }) {
  return (
    <article>
      <h2>{title}</h2> {/* React escapes text: safe */}
      <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(descriptionHtml) }} /> {/* sanitize first */}
      <a href={safeUrl(website)}>Website</a> {/* escaping does not stop javascript: URLs */}
    </article>
  );
}

function safeUrl(raw: string): string {
  try {
    const url = new URL(raw);
    return url.protocol === "https:" || url.protocol === "http:" ? url.toString() : "#";
  } catch {
    return "#";
  }
}
```

1. **Escape output for its context.** ERB and JSX escape text by default; the bugs come from the escape hatches (`raw`, `html_safe`, `<%==`, `dangerouslySetInnerHTML`) and from writing into attributes, URLs or inline scripts.
2. **Sanitize rich text with an allowlist** (Rails `sanitize`, Action Text's built-in sanitizing, DOMPurify on the client); never with regular expressions.
3. **Avoid dangerous DOM sinks:** `innerHTML`, `outerHTML`, `document.write`, `eval`, `new Function`, string arguments to `setTimeout`, jQuery `.html()`. Use `textContent`.
4. **Validate URL schemes** for user-supplied links: only `http` and `https`.
5. **Add CSP and HttpOnly cookies** as damage limiters, never as the primary fix.

## 3. Content Security Policy (CSP)

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-4AEemGb0xJptoIGFP3Nd';
  style-src 'self'; img-src 'self' https://images.example-cdn.com data:; connect-src 'self' https://api.example.com;
  object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'
```

- Each directive allowlists sources for one resource type; `default-src` is the fallback.
- Inline scripts and inline event handlers are blocked unless a script carries the per-response nonce (`<script nonce="...">`) or matches a hash. That is what neutralizes most injected markup. `'unsafe-inline'` and `'unsafe-eval'` undo much of the protection.
- `'strict-dynamic'` lets nonce-approved scripts load their own dependencies, which simplifies policies for bundled apps.
- **Rollout:** start with `Content-Security-Policy-Report-Only` and a reporting endpoint (`report-to` or `report-uri`), fix violations (third-party widgets, inline handlers, analytics), then enforce. Rails ships a CSP DSL in `config/initializers/content_security_policy.rb` with a per-request nonce generator that helpers such as `javascript_tag nonce: true` use.
- CSP is defense in depth: it does not replace escaping, and `frame-ancestors` also covers clickjacking.

## 4. Cross-site request forgery (CSRF)

```html
<!-- on https://evil.example, opened by a logged-in user -->
<form action="https://app.example.com/account/email" method="post" id="f">
  <input type="hidden" name="email" value="attacker@evil.example">
</form>
<script>document.getElementById("f").submit();</script>
```

If the browser attaches the session cookie, the server sees an authenticated request it cannot distinguish from a real one. The attacker never reads the response; changing the email (then resetting the password) is enough.

**Defenses, layered:**
1. **SameSite=Lax or Strict** session cookies: cross-site POSTs arrive without the cookie. A strong baseline, but not complete alone: a compromised sibling subdomain counts as same-site, and state-changing GET endpoints stay exposed.
2. **Synchronizer tokens:** the server stores a random token in the session and embeds it in forms and a meta tag; state-changing requests must echo it, and the attacker's page cannot read it because of the same-origin policy. Rails: `protect_from_forgery with: :exception` (the default in `ActionController::Base`), `form_with` adds `authenticity_token`, and `csrf_meta_tags` exposes it to JavaScript, where Turbo and Rails UJS send it as `X-CSRF-Token`.
3. **Origin and Fetch Metadata checks:** reject state-changing requests whose `Origin` is foreign or whose `Sec-Fetch-Site` is `cross-site`.
4. **Never change state on GET.**

```ts
// Custom fetch in a Rails page: send the token from csrf_meta_tags
const token = document.querySelector<HTMLMetaElement>('meta[name="csrf-token"]')?.content ?? "";
await fetch("/bills", {
  method: "POST",
  headers: { "Content-Type": "application/json", "X-CSRF-Token": token },
  body: JSON.stringify({ bill: { title: "Internet", amount_cents: 4500 } }),
});
```

APIs authenticated with an `Authorization` header are not exposed to CSRF, because browsers never add that header on their own; that is why `ActionController::API` does not include forgery protection. Move tokens into cookies for an SPA and CSRF is back.

| | XSS | CSRF |
|---|---|---|
| What the attacker does | runs their JavaScript inside your origin | makes the victim's browser send a request to your site |
| Can read responses | yes | no |
| Requires | an injection point rendered without escaping | cookie-based auth and a state-changing endpoint without checks |
| Beats the other's defenses | yes: XSS can read CSRF tokens | no |
| Main defenses | escaping, sanitizing, CSP, HttpOnly cookies | SameSite cookies, CSRF tokens, Origin checks |

## 5. Injection: SQL, NoSQL and command

```ruby
# Vulnerable: input becomes part of the SQL text
User.where("email = '#{params[:email]}'")          # email: ' OR '1'='1  returns every user
Listing.order("#{params[:sort]} #{params[:dir]}")  # identifiers cannot be bound as parameters

# Safe: values travel separately from the SQL; identifiers come from an allowlist
User.where(email: params[:email])
User.where("email = ? AND status = ?", params[:email], "active")
Listing.order(SORTABLE.fetch(params[:sort], :created_at) => (params[:dir] == "asc" ? :asc : :desc))
```

```ts
// node-postgres: $1 placeholders with values passed separately
await pool.query("SELECT * FROM users WHERE email = $1", [email]);

// Drizzle: the sql template tag turns ${} values into bound parameters;
// sql.raw() inserts text verbatim, so use it only for allowlisted identifiers
await db.execute(sql`SELECT * FROM users WHERE email = ${email}`);
```

ORM escape hatches (`find_by_sql`, string conditions, `Arel.sql`, raw `pluck` expressions) need the same care, and a least-privilege database user limits the blast radius. Brakeman flags many of these patterns in Rails code.

**NoSQL injection** happens when a JSON body is passed straight into a query:

```ts
// Vulnerable: req.body.password can be {"$ne": null}, which matches any document
const user = await User.findOne({ email: req.body.email, password: req.body.password });

// Safe: validate shapes and types first, then compare a password hash outside the query
const { email, password } = z.object({ email: z.string().max(254), password: z.string().min(8) }).parse(req.body);
const found = await User.findOne({ email });
const ok = found ? await bcrypt.compare(password, found.passwordHash) : false;
```

Mongoose's `sanitizeFilter` option and middleware that strips `$`-prefixed keys add another layer.

**Command injection:**

```ruby
system("convert #{params[:file]} out.png")   # the shell parses ; | && $(...): "x.png; rm -rf ~" runs
system("convert", params[:file], "out.png")  # array form: no shell, arguments passed as-is
```

In Node, use `execFile("convert", [inputPath, "out.png"])` or `spawn` instead of `exec` with an interpolated string. Even without a shell, reject inputs that start with `-`, which a program may parse as an option (argument injection).

## 6. Clickjacking

The attacker loads your page in an invisible iframe over a decoy button, so the user unknowingly clicks "Delete account" or "Confirm payment" on your site. Forbid framing with `Content-Security-Policy: frame-ancestors 'none'` (or `'self'`, or a list of partner origins) plus the legacy `X-Frame-Options: DENY` or `SAMEORIGIN`; Rails sends `X-Frame-Options: SAMEORIGIN` by default. Embeddable widgets need an explicit allowlist.

## 7. CORS misconfiguration

- Reflecting whatever `Origin` arrives together with `Access-Control-Allow-Credentials: true`: any website can read a logged-in user's API responses.
- Allowing the `null` origin, which sandboxed iframes and local files send.
- Sloppy matching: `endsWith("example.com")` accepts `evilexample.com`, and `startsWith("https://example.com")` accepts `https://example.com.evil.net`.
- Believing CORS blocks requests: simple cross-site POSTs still execute; CORS only controls reading responses, so CSRF defenses are still required.

Use an exact allowlist per environment, credentials only where needed, and `Vary: Origin`. Mechanics: [How the web works](01-how-the-web-works.md).

## 8. Mass assignment

```ruby
# Vulnerable: permit! (or permitting :role) lets a user send user[role]=admin or user[organization_id]=7
current_user.update(params.require(:user).permit!)

# Safe: allowlist what this user may change; set sensitive fields on the server
current_user.update(params.require(:user).permit(:name, :phone))
# Rails 8.0+ alternative: current_user.update(params.expect(user: [:name, :phone]))
```

A 2012 mass-assignment exploit against GitHub's Rails app is a big reason strong parameters became the Rails default. Elsewhere: DRF serializers with explicit `fields` and `read_only_fields`; in TypeScript, parse bodies with a zod schema (`z.object` strips unknown keys by default) and never spread `req.body` into an ORM update.

## 9. Open redirects

`redirect_to params[:return_to]` lets a phishing email link to `https://app.example.com/login?return_to=https://evil.example/login`: the link looks like yours, and after login the user lands on a copy of your site. Accept only relative paths that start with a single `/` (reject `//evil.example`, which browsers treat as another host, and backslash tricks), or match an allowlist of hosts. Rails 7.0+ new-app defaults raise on redirects to other hosts unless you pass `allow_other_host: true`, and `url_from(params[:return_to])` returns nil for foreign URLs. Open redirects also help attackers steal OAuth codes when redirect URIs are matched loosely.

## 10. File upload risks

| Risk | Example | Mitigation |
|---|---|---|
| Stored XSS | an uploaded `.html` or `.svg` with script, served from your domain | serve user files from a separate domain, force `Content-Disposition: attachment` for risky types, send `nosniff`, sanitize or rasterize SVGs |
| Disguised type | a script renamed `photo.jpg` | check magic bytes, allowlist types, ignore the extension and the client's `Content-Type` |
| Path traversal | a filename like `../../config/database.yml` | random storage keys; never use the client filename as a path |
| Denial of service | multi-gigabyte uploads, decompression bombs, giant images | size limits at the proxy and app, pixel limits, processing in background jobs |
| Parser exploits | crafted images aimed at image libraries | keep ImageMagick or libvips updated, restrict formats |
| Data exposure | public buckets, guessable URLs | private buckets, short-lived presigned URLs issued after an authorization check |

Direct uploads to S3 with presigned URLs keep large files off app servers (Active Storage supports this; see [Action Cable, Mailer and Active Storage](../05-ruby-on-rails/15-actioncable-mailer-activestorage.md)). Scan files that users share with each other.

## 11. Server-side request forgery (SSRF)

Features that fetch URLs on the server (sending webhooks to customer URLs, "import from URL", link previews, HTML-to-PDF rendering, image proxies) can be pointed at `http://169.254.169.254/` (cloud metadata, sometimes with credentials), `http://localhost:6379` (Redis) or `http://10.0.0.5/admin`, directly or through redirects and DNS names that resolve to private addresses.

- Allowlist destination hosts when the feature allows it.
- Otherwise resolve the hostname, reject private, loopback, link-local and metadata ranges, and connect to the IP you checked (so a second DNS answer cannot switch it, known as DNS rebinding).
- Do not follow redirects automatically, or re-validate every hop; allow only `http` and `https`; set timeouts and response size limits.
- Run fetchers with restricted network egress, and on AWS require IMDSv2, whose session tokens stop simple GET-based SSRF from reading metadata.

## 12. Secrets management

- **Where secrets live:** environment variables injected by the platform (Heroku config vars, Render or Fly secrets, Kamal secrets), a secret manager (AWS Secrets Manager or SSM Parameter Store, Vault, Doppler), Rails encrypted credentials (`config/credentials.yml.enc` is committed; `master.key` or `RAILS_MASTER_KEY` never is), and CI secrets in GitHub Actions.
- **Never:** commit `.env` files (commit a `.env.example` instead), log secrets (filter parameters, scrub headers), put them in URLs, or ship them in frontend or mobile bundles. Anything prefixed `NEXT_PUBLIC_` or `EXPO_PUBLIC_` is public by design, and app binaries can be unpacked.
- **Least privilege:** scoped API keys and IAM roles, separate keys per environment, rotation on a schedule and when people leave.
- **Detection:** GitHub secret scanning and push protection, gitleaks in a pre-commit hook or CI.
- **When a key leaks:** revoke or rotate it at the provider first, because bots scan public repositories within minutes; check the provider's logs for misuse; then clean history with `git filter-repo` if needed. Rewriting history alone is not enough: forks, clones and caches keep the old commits ([Git](../14-devops-and-tools/01-git.md)).

## 13. HTTPS, HSTS and security headers

Serve everything over HTTPS, redirect HTTP to HTTPS and mark cookies `Secure`. `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` tells browsers to use only HTTPS for your domain for two years, removing the downgrade window after the first visit; `preload` gets the domain into browser preload lists, so make sure every subdomain supports HTTPS first. Rails `config.force_ssl = true` redirects, sends HSTS and marks cookies secure. TLS internals live in [Networking essentials](../01-cs-fundamentals/06-networking.md).

| Header | Typical value | Protects against |
|---|---|---|
| `Content-Security-Policy` | see section 3 | XSS, injected resources, framing (`frame-ancestors`) |
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains` | protocol downgrade, cookies sent over HTTP |
| `X-Content-Type-Options` | `nosniff` | browsers guessing types, such as running an upload as a script |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | clickjacking in older browsers |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | leaking full URLs with IDs or tokens to other sites |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` | unwanted use of powerful browser features |
| `Cross-Origin-Opener-Policy` | `same-origin` | cross-window attacks through `window.opener` |

`X-XSS-Protection` is obsolete; current guidance is to omit it or set it to `0` and rely on CSP. The MDN HTTP Observatory grades a site's headers.

## 14. Rate limiting and brute-force protection

- Protect login, password reset, OTP verification, signup and expensive search endpoints.
- Limit per account (stops guessing one user's password) and per IP (stops one source spraying many accounts). Prefer growing delays over hard lockouts, which let attackers lock real users out.
- Add MFA, breached-password checks, a CAPTCHA or device challenge after repeated failures, generic error messages and alerts on failure spikes.
- Keep counters in a shared store (Redis) so limits hold across servers.

```ruby
class SessionsController < ApplicationController
  # Rails 7.2+; older apps use Rack::Attack
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { redirect_to new_session_url, alert: "Try again later." }
end
```

## 15. Webhook HMAC verification

```ruby
class Webhooks::PaymentsController < ActionController::API
  TOLERANCE_SECONDS = 300

  def create
    payload   = request.raw_post                              # the exact bytes that were signed
    timestamp = request.headers["X-Webhook-Timestamp"].to_i
    signature = request.headers["X-Webhook-Signature"].to_s

    expected = OpenSSL::HMAC.hexdigest("SHA256", ENV.fetch("PAYMENTS_WEBHOOK_SECRET"), "#{timestamp}.#{payload}")
    fresh    = (Time.now.to_i - timestamp).abs <= TOLERANCE_SECONDS
    return head(:unauthorized) unless fresh && ActiveSupport::SecurityUtils.secure_compare(expected, signature)

    event = JSON.parse(payload)
    ProcessPaymentEventJob.perform_later(event["id"], payload) # the job dedupes on the event id
    head :ok
  end
end
```

- Verify the **raw body**: re-serialized JSON may differ by a single space and fail the check, or worse, you verify something other than what you process.
- Compare in **constant time** (`secure_compare`, `crypto.timingSafeEqual`) so response timing does not leak the expected signature.
- Reject old **timestamps** to stop replays, dedupe by event ID, and respond quickly while a job does the work.
- Support two active secrets during rotation. Providers such as Stripe, GitHub and Shopify define their own header names and signing strings and ship verification helpers; use them when they exist. If the controller inherits from `ActionController::Base`, skip forgery protection for this action, since the provider has no CSRF token. Webhook design is covered in [REST API design](04-rest-api-design.md).

## 16. Dependency auditing and supply chain

- **Ruby:** `bundle exec bundler-audit check --update` checks `Gemfile.lock` against known advisories; **Brakeman** statically analyzes Rails code for SQL injection, XSS, mass assignment and unsafe redirects, and is part of new apps and their default CI from Rails 7.2.
- **JavaScript:** `npm audit` (`--omit=dev` to focus on production dependencies) or `pnpm audit`; judge severity and whether the vulnerable code path is reachable. **Python:** `pip-audit`.
- **GitHub:** Dependabot alerts, security update PRs and grouped version updates ([GitHub and workflows](../14-devops-and-tools/02-github-and-workflows.md)); code scanning with CodeQL; secret scanning.
- **Hygiene:** commit lockfiles and install them frozen in CI (`npm ci`, Bundler's frozen or deployment mode), review lockfile diffs, prefer maintained packages, watch for typosquats, pin GitHub Actions to versions or commit SHAs, and enable 2FA on registry accounts. Popular npm packages have been hijacked in recent years, so treat every update as code you ship.
- Upgrade little and often; a jump across several Rails majors is far riskier ([Legacy codebases and upgrades](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md)).

## 17. PII and GDPR basics for EU clients

- **Scope:** GDPR applies to personal data of people in the EU, wherever the processing company sits, so work on a Belgian platform falls under it through the client.
- **Personal data** is anything that identifies a person: name, email, phone, address, IP address, device IDs, location, photos, rental applications, payment details. Special categories (health, religion, ethnicity) need extra protection.
- **Roles:** the client is usually the *controller*; hosting, email and analytics vendors (and often your employer) are *processors* bound by a Data Processing Agreement.
- **Principles:** a lawful basis (contract, consent, legitimate interest, legal obligation), purpose limitation, data minimization, accuracy, storage limitation and security.
- **Rights:** access (an export), rectification, erasure (delete or anonymize, including backups on a schedule), portability and objection.
- **Breaches** must be reported to the supervisory authority within 72 hours of discovery, and to users when the risk to them is high. Fines can reach 4% of global annual turnover.
- **Transfers** outside the EU need a legal mechanism (an adequacy decision or Standard Contractual Clauses), which affects hosting regions and remote access to production data.
- **Cookies:** under ePrivacy rules, non-essential cookies (analytics, ads) need opt-in consent; strictly necessary ones (sessions) do not.
- **Developer habits:** keep personal data out of logs and error trackers (Rails `filter_parameters`, Sentry scrubbing), never copy production data to laptops or staging without anonymizing it, restrict and audit production access, encrypt sensitive columns, run retention jobs, and implement account deletion end to end. Apple requires in-app account deletion for apps that support account creation, and Google Play has a similar requirement.

## Common pitfalls
- `raw`, `html_safe` or `dangerouslySetInnerHTML` on user content; relying on CSP or a WAF instead of fixing escaping.
- Disabling CSRF protection to "make a form work", or for JSON endpoints that use cookie auth.
- String-built SQL, especially for `ORDER BY` and column names.
- Reflecting origins in CORS with credentials allowed.
- `permit!`, or spreading request bodies into model updates.
- `redirect_to params[:return_to]` without validation.
- Serving uploads from the main domain with sniffable types.
- Fetching user-supplied URLs from the server without SSRF protection.
- Committing secrets, or shipping them in `NEXT_PUBLIC_` or `EXPO_PUBLIC_` variables; rewriting history without rotating.
- Ignoring bundler-audit and Dependabot alerts until a major upgrade is forced.
- Logging tokens, passwords or personal data.

## How to talk about this in an interview
- Connect each category to real code. CrystalCommerce is a large legacy Rails e-commerce platform: strong parameters, escaping in ERB, CSRF protection, Brakeman findings and dependency upgrades all apply. Colochousing serves EU residents, so personal data, rental applications and rent payments raise GDPR and access-control questions, and org-scoped access control in the platform migration is A01 in practice.
- Integrations such as PriceCharting bring secrets management and, if webhooks are involved, signature verification.
- Present security as habits rather than heroics: escape by default, allowlist input, scope every query, least privilege, audit dependencies, never log secrets.
- If you have not done something (a CSP rollout, a penetration test), say so and describe how you would approach it.

> **Fill in:** a security issue you found or fixed (an unscoped query, missing strong parameters, an XSS in a view, a vulnerable gem) and how you verified the fix.

> **Fill in:** how secrets were managed on your Devntech projects (Rails credentials, environment variables, a secret manager) and who could access production data.

> **Fill in:** any GDPR-related work on Colochousing (account deletion, data export, filtering personal data from logs).

## Interview questions

**Q1. Walk me through the OWASP Top 10.**
The 2021 list: broken access control, cryptographic failures, injection (including XSS), insecure design, security misconfiguration, vulnerable and outdated components, identification and authentication failures, software and data integrity failures, security logging and monitoring failures, and SSRF. For each, have one example and one mitigation ready; access control and injection matter most for typical web apps. OWASP published a newer edition in 2025.
*Follow-up:* Which do you see most in Rails apps? Broken access control (unscoped `find`) and outdated dependencies.

**Q2. What is XSS, what types exist, and how do you prevent it in Rails and React?**
Untrusted input executes as script in your origin: stored, reflected or DOM-based. Prevent it with auto-escaping (ERB `<%= %>`, JSX), avoiding `raw`, `html_safe` and `dangerouslySetInnerHTML` on user data, sanitizing rich text with an allowlist, validating URL schemes, avoiding DOM sinks like `innerHTML`, and adding CSP and HttpOnly cookies as backup layers.
*Follow-up:* Does React make you immune? No: `dangerouslySetInnerHTML`, `javascript:` URLs and direct DOM manipulation still introduce XSS.

**Q3. What is a Content Security Policy, and how would you roll one out?**
A response header that allowlists sources for scripts, styles, images, connections and frames, and blocks inline scripts without a nonce or hash. Roll out in report-only mode with a reporting endpoint, fix violations (inline handlers, third-party scripts), switch to nonces, then enforce.
*Follow-up:* Why avoid `'unsafe-inline'`? It allows exactly the injected inline scripts CSP is meant to stop.

**Q4. What is CSRF, and how does Rails protect against it?**
A malicious site makes the victim's browser send an authenticated request, relying on cookies being attached automatically. Rails embeds a per-session authenticity token in forms and meta tags and rejects state-changing requests without a valid token (`protect_from_forgery`); SameSite cookies and Origin checks add layers.
*Follow-up:* Why don't Rails API-only controllers have it? Header-based tokens are not sent automatically, so CSRF does not apply unless you authenticate with cookies.

**Q5. XSS vs CSRF: what is the difference?**
XSS runs attacker code inside your origin, so it can read data and act as the user, and it can read CSRF tokens. CSRF only makes the browser send a request and cannot read responses. XSS is fixed with escaping, sanitizing and CSP; CSRF with SameSite cookies, tokens and Origin checks.
*Follow-up:* Which is worse? XSS, because it defeats CSRF protection and exposes everything the page can see.

**Q6. Does SameSite make CSRF tokens unnecessary?**
It blocks most CSRF for cookies marked Lax or Strict, but not all: sibling subdomains are same-site, state-changing GETs remain exposed, cookies that need `SameSite=None` get no protection, and defaults vary across browsers. Keep tokens (or Origin checks) for defense in depth.
*Follow-up:* What does Lax allow? Cookies on top-level navigations with safe methods, such as following a link.

**Q7. What is SQL injection, and how do you prevent it, including in ORDER BY?**
Input is concatenated into SQL and changes the query's meaning. Use bound parameters or ORM methods (`where(email: x)`, `where("email = ?", x)`, `$1` placeholders). Identifiers such as sort columns cannot be bound, so map them through an allowlist.
*Follow-up:* Which Rails methods are risky? String conditions with interpolation, `find_by_sql`, `order` or `pluck` with raw input, and `Arel.sql`.

**Q8. Give examples of NoSQL injection and command injection.**
NoSQL: a JSON body sends `{"password": {"$ne": null}}` into a MongoDB query and matches any document; validate input types with a schema and never query by plaintext password. Command: `system("convert #{file}")` lets `x.png; rm -rf ~` run through the shell; use array-form `system` or `execFile` and reject option-like inputs.
*Follow-up:* What is argument injection? Input beginning with `-` interpreted as a flag by the called program.

**Q9. What is SSRF, and how do you defend against it?**
Making the server fetch an attacker-chosen URL, such as cloud metadata or internal admin services. Allowlist destinations; otherwise resolve DNS, block private, loopback and link-local ranges, connect to the checked IP, re-validate redirects, restrict schemes, limit time and size, and isolate egress (IMDSv2 on AWS).
*Follow-up:* Which features are at risk? Webhook senders, URL imports, link previews and HTML-to-PDF renderers.

**Q10. What is clickjacking?**
Your page is loaded in an invisible iframe over a decoy, so users click your buttons without knowing. Forbid framing with CSP `frame-ancestors 'none'` (or a partner allowlist) and `X-Frame-Options: DENY` or `SAMEORIGIN`.
*Follow-up:* What if you must be embeddable? List the allowed parent origins in `frame-ancestors`.

**Q11. Is CORS a security feature? What are common misconfigurations?**
It relaxes the same-origin policy for chosen origins; the SOP is the protection, and CORS is a controlled opening. Misconfigurations: reflecting any origin with credentials, allowing `null`, sloppy suffix or prefix matching, and assuming CORS stops cross-site POSTs (it does not, so CSRF defenses remain necessary).
*Follow-up:* Why `Vary: Origin`? So caches do not serve one origin's CORS headers to another.

**Q12. What is mass assignment?**
Binding request parameters directly to model attributes, letting clients set fields such as `role`, `admin` or `organization_id`. Prevent it with allowlists: Rails strong parameters (`permit`, or `params.expect` in Rails 8.0+), DRF serializer fields, zod schemas, and set sensitive fields server-side.
*Follow-up:* Why is `permit!` dangerous? It permits every attribute, recreating the vulnerability.

**Q13. What is an open redirect, and why does it matter?**
An endpoint redirects to a URL taken from the request without validation, so attackers send links on your trusted domain that land users on phishing pages; it can also leak OAuth codes. Allow only relative paths or allowlisted hosts; Rails 7.0+ defaults raise on cross-host redirects.
*Follow-up:* Why is `//evil.example` dangerous? Browsers treat it as a protocol-relative URL to another host.

**Q14. How do you secure file uploads?**
Allowlist types and verify them by content, cap sizes, generate storage keys instead of using client filenames, store privately with short-lived presigned URLs, serve from a separate domain or as attachments with `nosniff`, process images in background jobs with updated libraries, and scan shared files.
*Follow-up:* Why are SVGs risky? They can contain scripts that run when served inline from your origin.

**Q15. How do you manage secrets? What do you do if one is committed?**
Keep them in platform environment variables, a secret manager or Rails encrypted credentials, scoped per environment, never logged or bundled into clients, and scanned for in pushes. If one leaks, revoke and rotate it immediately, check for misuse, then clean history; rewriting history alone does not help because copies already exist.
*Follow-up:* Is `EXPO_PUBLIC_API_KEY` secret? No. Public env vars are compiled into the bundle, so only put publishable keys there.

**Q16. Which security headers do you set, and why?**
CSP (XSS and framing), HSTS (force HTTPS), `X-Content-Type-Options: nosniff` (no type sniffing), `X-Frame-Options` or `frame-ancestors` (clickjacking), `Referrer-Policy` (limit URL leakage) and `Permissions-Policy` (disable unused browser features). Skip the obsolete `X-XSS-Protection`.
*Follow-up:* Risk of HSTS preload? Every subdomain must support HTTPS, and removal from preload lists is slow.

**Q17. How do you protect login endpoints from brute force and credential stuffing?**
Rate limit per account and per IP with growing delays, offer MFA, check breached passwords, return generic errors with uniform timing, challenge suspicious clients after repeated failures, and alert on spikes. Rails 7.2+ `rate_limit` or Rack::Attack implement the limits.
*Follow-up:* Why not lock accounts after five failures? Attackers can then lock out real users at will.

**Q18. How do you verify webhooks?**
Compute an HMAC (usually SHA-256) of the timestamp and the raw body with the shared secret, compare it to the signature header in constant time, reject stale timestamps, dedupe by event ID, respond fast and process in a job. Use the provider's SDK helper when it has one.
*Follow-up:* Why the raw body? Parsing and re-serializing JSON can change bytes, breaking or bypassing verification.

**Q19. How do you keep dependencies secure?**
Commit lockfiles and install them frozen in CI, run bundler-audit and `npm audit` (and Brakeman for Rails code), enable Dependabot alerts and grouped update PRs, review changelogs and lockfile diffs, prefer maintained packages, pin CI actions, and upgrade regularly instead of in big risky jumps.
*Follow-up:* How do you prioritize a flood of alerts? By severity, reachability of the vulnerable code, and exposure (production dependencies first).

**Q20. What does GDPR mean for you as a developer?**
Collect only what you need for a stated purpose, protect it (encryption, access control, no personal data in logs), support access, export and deletion requests, apply retention limits, keep production data out of unsafe environments, and help the team meet the 72-hour breach notification duty. Data processing agreements and EU transfer rules shape which vendors and regions you can use.
*Follow-up:* What counts as personal data? Anything that can identify a person, including emails, IP addresses and device IDs.

## Further reading
- [OWASP Top 10](https://owasp.org/Top10/) and [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [Rails guide: Securing Rails Applications](https://guides.rubyonrails.org/security.html)
