# Authentication and Authorization

> **Tier:** T1 (know cold) · **Related:** [How the web works](01-how-the-web-works.md), [Web security and OWASP](07-web-security-owasp.md), [REST API design](04-rest-api-design.md), [Rails authentication and authorization](../05-ruby-on-rails/11-authentication-authorization.md), [DRF auth and permissions](../10-python-django/05-drf-auth-permissions-advanced.md), [Mobile state, storage and auth](../08-react-native-expo/04-state-storage-auth-offline.md), [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md)
> **On your resume:** org-scoped access control in the Colochousing multi-tenant platform migration, role-based dashboards in the PureLogics LMS, and login flows for the Rails web app and the Coloc mobile app at Devntech.

## Quick revision
- **Authentication (authN)** = who are you? **Authorization (authZ)** = what may you do? Failed authN returns 401; failed authZ returns 403 (or 404 to hide the resource).
- Hash passwords with a slow, salted, adaptive algorithm: **argon2id** or **bcrypt** (scrypt and PBKDF2 are acceptable). Never plaintext, never reversible encryption, never a fast hash such as MD5 or SHA-256 alone.
- A **salt** is random per password and stored with the hash: it defeats precomputed tables and makes equal passwords hash differently. bcrypt embeds the salt and cost in its output.
- **Server sessions:** a random ID in an HttpOnly cookie, data in Redis or the database; revocation is instant. **Cookie sessions:** signed and encrypted data inside the cookie; no store, hard to revoke.
- **JWT** = base64url `header.payload.signature`. Signed, not encrypted: anyone can read the claims. Standard claims: `iss`, `sub`, `aud`, `exp`, `iat`, `nbf`, `jti`.
- **HS256** = HMAC with one shared secret. **RS256/ES256** = a private key signs and a public key (published as a JWKS) verifies, which suits several services or third parties.
- Verify strictly: an algorithm allowlist (reject `alg: none`), signature, `exp`, `iss`, `aud`.
- **The revocation problem:** a stateless JWT stays valid until it expires. Use short-lived access tokens (5 to 15 minutes) with server-stored refresh tokens, a `jti` denylist, or a token version on the user.
- **Refresh token rotation:** every refresh returns a new refresh token and invalidates the old one; reuse of an old token signals theft, so revoke the whole token family.
- Browser storage: an HttpOnly, Secure, SameSite cookie (safe from XSS theft, needs CSRF defense) or memory plus a refresh cookie; `localStorage` is readable by any injected script. Mobile: Keychain/Keystore via `expo-secure-store`, never AsyncStorage.
- **OAuth 2.0** is delegated authorization. Roles: resource owner, client, authorization server, resource server. Access tokens carry **scopes**.
- **Authorization code + PKCE** for SPAs and mobile apps (and recommended for all clients); **client credentials** for service-to-service; device code for TVs and CLIs. Implicit and password grants are deprecated.
- **PKCE:** the client sends `code_challenge = BASE64URL(SHA256(code_verifier))` first and the `code_verifier` when redeeming the code, so an intercepted code is useless. `state` protects the callback from CSRF.
- **OpenID Connect** adds identity: the `openid` scope, an **ID token** (a JWT about the user, meant for the client), the userinfo endpoint and discovery. SAML 2.0 is the older XML-based enterprise SSO protocol.
- MFA: **TOTP** (6 digits from an HMAC of a shared secret and the current 30-second window), **passkeys/WebAuthn** (phishing-resistant), SMS (weakest). **Magic links:** single-use, short-lived, stored hashed.
- **API keys** for server-to-server integrations: high-entropy, shown once, stored as a hash, prefixed, scoped, rotatable and revocable.
- **RBAC** = permissions through roles; **ABAC** = rules over attributes of user, resource and context; **ReBAC** = permissions from relationships ("member of this house").
- Multi-tenant authZ: users belong to organizations through **memberships** with a role; every query is scoped to the current org; deny by default; least privilege.
- Classic bugs: IDOR (broken object-level authorization), admin endpoints protected only in the UI, session fixation, CSRF with cookie auth, user enumeration, unchecked JWT `aud` and `exp`.

## 1. Authentication vs authorization

| | Authentication | Authorization |
|---|---|---|
| Question | Who are you? | What are you allowed to do with this resource? |
| When | at login; afterwards every request proves it with a session or token | on every action, for every resource |
| Mechanisms | passwords, OAuth/OIDC, magic links, MFA, API keys | roles, policies, ownership checks, token scopes |
| Failure | 401 Unauthorized | 403 Forbidden, or 404 to hide existence |
| Rails examples | `has_secure_password`, the Rails 8 generator, Devise | Pundit, CanCanCan, scoped queries |

## 2. Storing passwords

**Encryption** is reversible: whoever gets the key (and keys leak along with databases) gets every password. **Hashing** is one-way, but general-purpose hashes like SHA-256 are designed to be fast, so a GPU can test billions of guesses per second. Password hashing functions are deliberately slow, tunable through a **cost factor**, and in argon2's case **memory-hard**, which makes every guess expensive.

- **Salt:** a random value per password, stored next to the hash. Two users with the same password get different hashes, and precomputed rainbow tables become useless. bcrypt output such as `$2a$12$<salt><hash>` already contains the algorithm, cost and salt.
- **Pepper (optional):** an extra secret kept outside the database (an environment variable or secret store) and mixed in, so a database-only leak is not enough.
- **bcrypt:** cost 12 is a common choice; it only uses the first 72 bytes of input, which is why Rails' `has_secure_password` validates the maximum length. **argon2id** is OWASP's first recommendation for new systems; Django defaults to PBKDF2 and supports Argon2.

```ruby
# Gemfile: gem "bcrypt"
class User < ApplicationRecord
  has_secure_password # password= stores a bcrypt digest in password_digest and adds #authenticate
end

user = User.authenticate_by(email: params[:email], password: params[:password]) # Rails 7.1+, nil on failure
```

```ts
import bcrypt from "bcrypt";

const hash = await bcrypt.hash(password, 12);     // cost 12; a random salt is generated and embedded
const ok = await bcrypt.compare(candidate, hash); // re-hashes the candidate with the embedded salt and cost
```

**Login hygiene:**
- Return the same message for an unknown email and a wrong password ("Invalid email or password"), with similar timing. Rails' `authenticate_by` is designed to take the same time even when the email does not exist.
- Rate limit and slow down attempts per account and per IP; check new passwords against breached-password lists; prefer long passphrases over complexity rules, as NIST guidance does.
- Rehash on the next successful login when you raise the cost or change algorithms.
- Password reset tokens are random, single-use, stored hashed, expire within 15 to 60 minutes, die when the password changes, and the reset form never reveals whether an email exists.

## 3. Sessions: server-side and cookie-based

1. The user logs in; the server verifies the password.
2. The server creates a session with a new random ID and sends `Set-Cookie: session_id=...; HttpOnly; Secure; SameSite=Lax`.
3. Every later request carries the cookie; the server loads the session and the user.
4. Logout deletes the session server-side and clears the cookie.

With **server-side sessions** (Redis or a table), you can list a user's devices, expire sessions on idle and absolute timeouts, and "log out everywhere" by deleting rows. The Rails 8 authentication generator works this way: a `Session` record per login, its ID kept in a signed cookie. With **cookie sessions** (Rails' default `CookieStore`), the data travels in an encrypted cookie: no store to run, but you cannot kill one stolen session before it expires unless you add a server-side version check.

**Always issue a fresh session ID at login** (Rails `reset_session`; Django's `login()` rotates the key for you). Otherwise an attacker who planted a known session ID before login shares the authenticated session: session fixation.

## 4. JWT: structure, signing and verification

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiJ9.<signature>

header     {"alg": "HS256", "typ": "JWT"}
payload    {"sub": "42", "org_id": "org_7", "role": "member", "iss": "api.example.com",
            "aud": "mobile-app", "iat": 1793999100, "exp": 1794000000}
signature  HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)
```

- The three parts are base64url-encoded, not encrypted: paste a token into any decoder and the claims are visible, so never put secrets or sensitive personal data inside. (Encrypted JWTs, JWE, exist but are rarely needed.)
- The selling point is self-containment: any service holding the key can verify the token without a database lookup. That is also the weakness: nothing can easily invalidate it early.
- Tokens travel on every request, so keep them small and avoid embedding permissions that change often.

| | HS256 (HMAC) | RS256 / ES256 (asymmetric) |
|---|---|---|
| Keys | one shared secret signs and verifies | private key signs, public key verifies |
| Who can mint tokens | anyone holding the secret, including every verifier | only the holder of the private key |
| Distribution | the secret must be copied to every verifier | public keys published at a JWKS URL, rotated by key ID (`kid`) |
| Fits | one backend that issues and verifies its own tokens | identity providers, many microservices, third-party verifiers |

**Verification checklist:** allow only the algorithm you expect (reject `alg: none`, and never let a server that accepts both HS256 and RS256 treat a public key as an HMAC secret), check the signature, `exp` (with a few seconds of clock-skew leeway), `nbf`, `iss` and `aud`, then load fresh authorization data for sensitive actions.

```ts
import { SignJWT, jwtVerify } from "jose";

const secret = new TextEncoder().encode(process.env.JWT_SECRET!);

export async function issueAccessToken(userId: string, orgId: string) {
  return new SignJWT({ org_id: orgId })
    .setProtectedHeader({ alg: "HS256" })
    .setSubject(userId)
    .setIssuer("api.example.com")
    .setAudience("mobile-app")
    .setIssuedAt()
    .setExpirationTime("15m")
    .sign(secret);
}

export async function verifyAccessToken(token: string) {
  const { payload } = await jwtVerify(token, secret, {
    algorithms: ["HS256"], // explicit allowlist
    issuer: "api.example.com",
    audience: "mobile-app",
  });
  return payload; // exp and nbf are checked automatically
}
```

In Ruby the `jwt` gem does the same (`JWT.encode(payload, secret, "HS256")`, and `JWT.decode` with the algorithm, `iss` and `aud` options); in DRF, `djangorestframework-simplejwt` provides access and refresh tokens.

## 5. The revocation problem, refresh tokens and rotation

A user logs out, changes their password or loses their role, or a token is stolen, and a stateless JWT keeps working until `exp`. The options, usually combined:
- **Short-lived access tokens** (5 to 15 minutes) limit the damage window.
- **Refresh tokens** are long-lived, stored server-side (hashed), revocable, and only ever sent to the token endpoint.
- **A denylist** of `jti` values in Redis until their expiry; precise, but it reintroduces a lookup per request.
- **A token version** (or `tokens_valid_after` timestamp) on the user record: bump it on password change or "log out everywhere"; tokens carrying an older version are rejected.

**Rotation with reuse detection:**
1. Login returns an access token (15 minutes) and refresh token RT1 (say 30 days), stored hashed with a family ID.
2. When the access token expires, the client sends RT1 to the refresh endpoint; the server marks RT1 as used and returns a new access token and RT2.
3. If RT1 ever shows up again (an attacker, or a buggy client), the server cannot tell who is legitimate, so it revokes the whole family and both parties must log in again.

Clients must avoid racing refreshes: when several requests get a 401 at once, only one refresh should run.

```ts
let refreshing: Promise<string> | null = null;

function getFreshAccessToken(): Promise<string> {
  // every request that hits a 401 at the same moment shares this one refresh call
  refreshing ??= refreshTokens().finally(() => { refreshing = null; });
  return refreshing;
}

export async function apiFetch(path: string, init: RequestInit = {}): Promise<Response> {
  const send = (token: string) => {
    const headers = new Headers(init.headers);
    headers.set("Authorization", `Bearer ${token}`);
    return fetch(`${API_URL}${path}`, { ...init, headers });
  };
  const res = await send(await getAccessToken());
  if (res.status !== 401) return res;
  return send(await getFreshAccessToken()); // retry once; if the refresh itself fails, log the user out
}
```

## 6. Where to store tokens

| Option | Can injected JS read it? | CSRF risk | Survives reload | Notes |
|---|---|---|---|---|
| HttpOnly, Secure, SameSite cookie | no (it can still send requests while the page is open) | yes: SameSite plus CSRF tokens | yes | best default for browser apps on your own domain |
| In-memory variable | harder (not persisted, not global) | no | no: restore it from a refresh cookie | common SPA setup: short access token in memory, refresh token in an HttpOnly cookie |
| `localStorage` / `sessionStorage` | yes, any script on the page | no | `localStorage` yes | avoid for long-lived tokens |
| Mobile: `expo-secure-store` (Keychain / Keystore) | sandboxed by the OS | not applicable | yes | never AsyncStorage, which is unencrypted |

No browser option survives XSS completely: injected code can always act as the user while the page is open. HttpOnly only stops it from carrying the token away. So fix XSS first ([Web security and OWASP](07-web-security-owasp.md)). A **backend-for-frontend (BFF)** goes one step further: the browser holds only a session cookie, and the BFF keeps the OAuth tokens on the server.

## 7. Cookies vs sessions vs JWT

These are not three versions of the same thing: a **cookie** is a browser storage and transport mechanism, a **session** is server-side state, and a **JWT** is a token format. You can put a session ID in a cookie, or a JWT in a cookie, or a JWT in an `Authorization` header.

| | Session ID in a cookie | Signed cookie session | JWT in an `Authorization` header |
|---|---|---|---|
| State lives in | the server store | the cookie | the token itself |
| Per-request cost | a session lookup (Redis or DB) | decrypt and verify | verify a signature, no lookup |
| Revocation | instant: delete the record | hard | hard: short expiry, refresh tokens, denylist |
| CSRF exposure | yes, the browser attaches it | yes | no, your code attaches the header |
| XSS exposure | not readable if HttpOnly | same | readable if stored where JS can reach it |
| Mobile, cross-domain, third parties | awkward: cookies are per domain | awkward | easy: any client can send a header |
| Size | a small ID | up to about 4 KB | grows with claims, sent on every request |
| Typical use | server-rendered Rails or Django apps, same-domain SPAs | Rails default | mobile apps, third-party clients, service-to-service |

A good interview answer: for a web app on your own domain, sessions in HttpOnly cookies are simpler and safer; for mobile apps, cross-domain APIs and services, use short-lived bearer tokens with rotating refresh tokens.

## 8. OAuth 2.0: roles, scopes and flows

OAuth 2.0 is a framework for **delegated authorization**: a user lets an app access their data on another service without giving it their password ("let this app read my calendar"). On its own it says nothing reliable about *who* the user is; OpenID Connect adds that.

- **Roles:** the *resource owner* (the user), the *client* (your app), the *authorization server* (issues tokens: Google, Auth0, Keycloak or your own) and the *resource server* (the API that accepts access tokens).
- **Tokens:** a short-lived *access token* presented to the API (a JWT or an opaque string) and a long-lived *refresh token* sent only to the authorization server.
- **Scopes** limit what a token may do (`calendar.read`, `bills:write`); the consent screen lists them and the resource server enforces them.
- **Client types:** confidential clients (server-side code that can keep a secret) vs public clients (SPAs and mobile apps; anything shipped to a device can be extracted).

| Flow | Use it for | Notes |
|---|---|---|
| Authorization code + PKCE | web apps, SPAs, mobile apps | the default; confidential clients also send their secret |
| Client credentials | service-to-service calls, scheduled jobs, no user | the client authenticates with its own credentials and gets a scoped token |
| Device authorization | smart TVs, CLIs | the user approves on another device with a short code |
| Refresh token | renewing access without a new login | rotate refresh tokens for public clients |
| Implicit (deprecated) | old SPAs | returned tokens in the URL fragment, where history, logs and referrers leak them; no refresh tokens |
| Resource owner password (deprecated) | legacy first-party apps | the app sees the user's password, which defeats the purpose |

RFC 9700 (the OAuth 2.0 Security Best Current Practice, published January 2025) formalizes these recommendations, and the OAuth 2.1 draft folds them into the core specification.

## 9. Authorization code with PKCE, step by step

```text
1. The app creates a random code_verifier and code_challenge = BASE64URL(SHA256(code_verifier)).

2. It opens the system browser at:
   https://auth.example.com/authorize?response_type=code&client_id=mobile-app
     &redirect_uri=myapp://callback&scope=openid%20profile%20bills:read&state=af0ifjsldkj
     &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256

3. The user logs in (and consents) on the authorization server.

4. The browser is redirected to myapp://callback?code=SplxlOBeZQQYbYS6WxSbIA&state=af0ifjsldkj
   and the app checks that state matches what it sent.

5. The app redeems the code:
   POST https://auth.example.com/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=myapp://callback
   &client_id=mobile-app&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk

6. The server checks BASE64URL(SHA256(code_verifier)) == code_challenge and returns
   {"access_token": "...", "refresh_token": "...", "id_token": "...", "token_type": "Bearer", "expires_in": 900}

7. The app calls the API with Authorization: Bearer <access_token>.
```

- **Why PKCE:** a public client has no secret, and the code passes through the browser and a redirect that another app could intercept (a malicious app can register the same custom URL scheme; codes can also leak into logs). The verifier never leaves the app, so an intercepted code cannot be redeemed.
- **`state`** ties the callback to a login the app actually started, preventing CSRF on the callback (for example, an attacker logging you into their account). OIDC's **`nonce`** does the same for the ID token.
- **On mobile:** use the system browser (`ASWebAuthenticationSession` on iOS, Custom Tabs on Android), never an embedded WebView, which could read the user's credentials and which providers such as Google block. `expo-auth-session` handles the redirect and PKCE for you; keep tokens in SecureStore. Verified universal links or app links make safer redirect URIs than custom schemes.

## 10. OpenID Connect, SSO and SAML

**OpenID Connect (OIDC)** is an identity layer on top of OAuth 2.0. Request the `openid` scope (plus `profile`, `email`) and the token response also contains an **ID token**: a JWT for the client describing the login (`iss`, `sub`, `aud` = your client ID, `exp`, `iat`, `nonce`, and optionally `email` and `name`). A userinfo endpoint returns profile claims, and `/.well-known/openid-configuration` advertises all endpoints and the JWKS URL. "Sign in with Google" and "Sign in with Apple" are OIDC.

| | ID token | Access token |
|---|---|---|
| Audience | the client app | the resource server (API) |
| Purpose | tells the client who logged in | grants access to an API with scopes |
| Format | always a JWT | JWT or opaque |
| Validated by | the client: signature via JWKS, `iss`, `aud`, `exp`, `nonce` | the API |

Do not send ID tokens to your API as access tokens, and key users on the provider's `sub` (stable and unique per issuer), not on email, which can change or be unverified. For iOS apps that offer third-party social login, Apple's App Store rules require Sign in with Apple or an equivalent privacy-focused option; check the current guideline.

**SSO** means one login session at an identity provider gives access to many apps. In enterprises that is often **SAML 2.0**: the identity provider (Okta, Microsoft Entra ID, Google Workspace) posts a signed XML assertion through the browser to the service provider. Use maintained libraries (ruby-saml, OmniAuth strategies) or an identity provider rather than hand-checking XML signatures, which are notoriously easy to verify wrongly. OIDC is the modern JSON-based alternative, and SCIM handles automatic user provisioning.

## 11. MFA, TOTP, passkeys and magic links

- **Factors:** something you know (password), have (phone, security key) or are (biometrics). MFA combines at least two.
- **TOTP (RFC 6238):** the server creates a random secret and shows it as a QR code (`otpauth://totp/App:sara@example.com?secret=BASE32SECRET&issuer=App`). The authenticator app computes an HMAC of the secret and the current 30-second time step, truncated to 6 digits; the server computes the same and accepts the current step plus or minus one for clock drift. Store the secret encrypted, rate limit attempts, reject reuse of a code within its window, and issue hashed one-time backup codes.
- **SMS codes** beat nothing but are exposed to SIM swapping and interception.
- **WebAuthn and passkeys:** a key pair per site, bound to the site's origin; the private key never leaves the device (or the user's synced keychain), so phishing pages cannot capture anything reusable.
- **Magic links:** email a link with a random, single-use token that is stored hashed and expires in about 15 minutes. Email security becomes account security, and corporate link scanners may "click" links, so land on a confirmation page that POSTs, or email a short code instead. Rails 7.1+ `generates_token_for` creates signed, purpose-scoped tokens that can expire and become invalid when chosen attributes change.

## 12. API keys for server-to-server integrations

API keys suit partner integrations and internal services where full OAuth would be overkill. Treat them like passwords that machines use:
- Generate at least 32 random bytes; add a prefix (`sk_live_`, `sk_test_`) so keys are recognizable in logs and by secret scanners.
- Show the key once; store only a hash. A fast SHA-256 is fine here because keys are long and random, unlike human passwords.
- Scope keys (read-only vs write, specific resources), allow several active keys for zero-downtime rotation, revoke instantly, record last use, and rate limit per key.
- Send keys in a header, never in URLs. Stronger alternatives: HMAC request signing (the secret never travels), mutual TLS, or OAuth client credentials with short-lived tokens.

```ruby
class ApiKey < ApplicationRecord
  belongs_to :organization

  def self.issue!(organization:, name:, scopes:)
    token = "sk_live_#{SecureRandom.urlsafe_base64(32)}"
    key = create!(organization: organization, name: name, scopes: scopes,
                  token_digest: Digest::SHA256.hexdigest(token), last_four: token[-4..])
    [key, token] # hand the plain token to the user once; only the digest is stored
  end

  def self.authenticate(token)
    return if token.blank?
    find_by(token_digest: Digest::SHA256.hexdigest(token), revoked_at: nil)
  end
end
```

## 13. Authorization models: RBAC, ABAC and ReBAC

| | RBAC | ABAC | ReBAC |
|---|---|---|---|
| Decides by | the user's roles | attributes of user, resource, action and environment | relationships between user and resource |
| Example | admins can delete listings | a resident may edit a bill they created within 24 hours if the house plan includes bill splitting | a user can see a bill if they are a member of the bill's house |
| Strengths | simple, auditable, easy admin UI | fine-grained and contextual | natural for sharing, orgs and hierarchies |
| Weaknesses | role explosion; cannot express ownership alone | harder to reason about and audit | needs a relationship graph, and an engine at scale |
| Tools | role enums, Rolify, Django groups | policy code (Pundit, CASL), engines such as OPA or Cedar | Zanzibar-style systems such as OpenFGA or SpiceDB |

Real apps mix them: roles inside an organization (RBAC), ownership and state rules in policies (ABAC-style), all scoped by membership (the simplest ReBAC). The principles do not change: deny by default, grant the least privilege needed, enforce on the server (a hidden button is not a permission check), keep rules in one place, and test permissions like any other feature.

## 14. Ownership checks and org-scoped access

```ruby
# Vulnerable (IDOR): any logged-in user can read any bill by changing the id in the URL
@bill = Bill.find(params[:id])

# Fixed: reach the record through something the user may access (404 when it is not theirs)
@bill = current_user.visible_bills.find(params[:id])

# Or load it and ask a policy (Pundit): BillPolicy#show? decides, and a denial raises
@bill = Bill.find(params[:id])
authorize @bill
```

A typical multi-tenant model:

```text
users --< memberships >-- organizations
memberships: user_id, organization_id, role (owner | admin | member), unique (user_id, organization_id)
every tenant-owned table: organization_id NOT NULL, indexed
```

1. Authenticate the user.
2. Resolve the current organization from the URL (`/orgs/:slug/...`), subdomain or a header, never blindly from a body field.
3. Load the membership for (user, org); no membership means 404, which also avoids confirming that the org exists.
4. Check the role's permission for the action, plus resource rules such as ownership or state.
5. Scope every query by organization (`current_org.bills.find(id)`), and set `organization_id` on create from the membership, never from params.
6. Add defense in depth: helpers or default scopes that make unscoped queries awkward, Postgres row-level security where it fits, and tests that attempt cross-org access.

```ts
import { and, eq } from "drizzle-orm";
// db and the memberships table come from your Drizzle setup

type Role = "owner" | "admin" | "member";
type Action = "bill:create" | "member:invite" | "org:delete";

const allowed: Record<Action, Role[]> = {
  "bill:create": ["owner", "admin", "member"],
  "member:invite": ["owner", "admin"],
  "org:delete": ["owner"],
};

export async function requirePermission(userId: string, orgId: string, action: Action) {
  const [membership] = await db
    .select()
    .from(memberships)
    .where(and(eq(memberships.userId, userId), eq(memberships.organizationId, orgId)))
    .limit(1);
  if (!membership) throw new NotFoundError();          // do not reveal that the org exists
  if (!allowed[action].includes(membership.role as Role)) throw new ForbiddenError();
  return membership;                                   // callers scope queries with membership.organizationId
}
```

In oRPC or Hono this check typically lives in middleware that puts the membership into typed context for every procedure. More patterns in [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md) and the [multi-tenant platform migration deep dive](../17-projects-deep-dive/04-multi-tenant-platform-migration.md).

## 15. Common vulnerabilities

- **IDOR / broken object-level authorization:** changing an ID returns someone else's data. Fix with scoped queries and policies; test with two users and two orgs.
- **Broken function-level authorization:** admin endpoints hidden in the UI but callable directly. Check roles on the server for every endpoint.
- **Session fixation:** an attacker plants a known session ID before login. Rotate the session ID at login.
- **CSRF with cookie auth:** browsers attach cookies to cross-site requests. Use SameSite cookies plus CSRF tokens for state-changing requests. Bearer tokens in headers are not sent automatically, so they are not exposed to CSRF (but are exposed to XSS if stored in JS-readable storage).
- **Credential stuffing and brute force:** rate limiting, MFA, breached-password checks, alerting on spikes.
- **User enumeration** through different login, signup or reset messages, or response timing.
- **JWT mistakes:** `alg: none`, unchecked `aud`, `iss` or `exp`, long-lived tokens without revocation, signing secrets committed to the repo, sensitive data in payloads.
- **OAuth mistakes:** no `state` or PKCE, redirect URIs matched by prefix instead of exactly (combined with an open redirect, codes get stolen), unvalidated ID tokens, email used as the identity key.
- **Weak password reset:** guessable or long-lived tokens, tokens that survive use, reset links built from the request's `Host` header.
- **Mass assignment** of `role` or `organization_id` from request bodies.

## 16. Implementations in your stacks

- **Rails:** `has_secure_password`, the Rails 8 authentication generator (sessions and password resets, no sign-up flow), Devise (with devise-jwt for token APIs), OmniAuth for OAuth/OIDC login, Doorkeeper to act as an OAuth provider, and Pundit or CanCanCan for authorization. See [Rails authentication and authorization](../05-ruby-on-rails/11-authentication-authorization.md).
- **Django and DRF:** `django.contrib.auth`, `SessionAuthentication`, `TokenAuthentication`, `djangorestframework-simplejwt`, permission classes and object-level `has_object_permission`. See [DRF auth and permissions](../10-python-django/05-drf-auth-permissions-advanced.md).
- **Node and TypeScript:** `jose` for JWTs, `bcrypt` or `argon2` packages, framework or oRPC middleware that puts the user and membership into typed context, or libraries such as Auth.js and Better Auth for complete flows. See [Production APIs](../09-nodejs-backend/06-production-apis.md).
- **Mobile:** tokens in SecureStore, a single-flight refresh, biometric unlock of stored credentials. See [Mobile state, storage and auth](../08-react-native-expo/04-state-storage-auth-offline.md).

## Common pitfalls
- Encrypting passwords or hashing them with a fast hash.
- Long-lived JWTs with no revocation plan; secrets or personal data inside JWT payloads.
- Tokens in `localStorage` or AsyncStorage.
- Authorization only in the UI, or only on list endpoints but not on show and update.
- `Model.find(params[:id])` without scoping to the user or org.
- Trusting `organization_id` or `role` from the request body.
- Keeping the same session ID across login, or keeping sessions alive after a password change.
- OAuth without PKCE or `state`, or redirect URIs matched loosely.
- Identifying social-login users by email instead of the provider's `sub`.
- Error messages that reveal whether an account exists.
- Hand-rolled crypto instead of vetted libraries.

## How to talk about this in an interview
- **Org-scoped access control** is a resume bullet from the Colochousing migration onto the multi-tenant TypeScript platform: explain the membership and role model, how the current organization was resolved, how every query was scoped, and how cross-org access was tested.
- **Role-based dashboards** in the PureLogics LMS (DRF and React) are an RBAC story: different roles see different dashboards, enforced by DRF permission classes rather than by hiding UI. Subscriptions gating course content add an attribute-based rule on top.
- **The Coloc mobile app** gives you token storage, refresh and logout to talk about: where tokens live, what happens on a 401, and what logout cleans up (tokens, cached data, push tokens).
- Use precise vocabulary (authN vs authZ, IDOR, PKCE, rotation, least privilege) and always mention the server-side check.

> **Fill in:** how authentication works in the Colochousing web app and the Coloc app (Devise or another library, sessions or tokens, where the app stores tokens, how refresh works).

> **Fill in:** how org-scoped access was enforced in the new platform (oRPC middleware, query helpers, roles), and one access-control bug you prevented or fixed.

> **Fill in:** the roles in the PureLogics LMS and how dashboards and permissions differed per role.

## Interview questions

**Q1. What is the difference between authentication and authorization?**
Authentication establishes who the caller is (password, OAuth login, API key); authorization decides whether that identity may perform this action on this resource (roles, policies, ownership). Authentication failures return 401, authorization failures 403 or 404.
*Follow-up:* Which runs more often? Authorization: on every action and every resource, while authentication is proven once and then carried by a session or token.

**Q2. How should passwords be stored? Why not encrypt them, and why not SHA-256?**
Hash them with a slow, salted, adaptive function such as argon2id or bcrypt. Encryption is reversible, so a leaked key exposes every password. SHA-256 is fast, so attackers can test billions of guesses per second; password hashes are deliberately slow and tunable (and argon2 is memory-hard).
*Follow-up:* How do you upgrade the cost later? Rehash with the new parameters on the user's next successful login.

**Q3. What are a salt and a pepper?**
A salt is a random per-password value stored with the hash; it makes identical passwords hash differently and defeats precomputed tables. A pepper is an application-wide secret kept outside the database, so a database dump alone is not enough to start cracking.
*Follow-up:* Does bcrypt need a separate salt column? No, the salt and cost are embedded in the hash string.

**Q4. Cookies vs sessions vs JWT: how do they compare?**
A cookie is a browser transport, a session is server-side state, and a JWT is a self-contained signed token. Session IDs in HttpOnly cookies are simple, instantly revocable and need CSRF protection; JWTs in headers avoid lookups and suit mobile and cross-domain clients but are hard to revoke and must be stored carefully.
*Follow-up:* What would you pick for a Rails web app plus a mobile app? Cookie sessions for the web app and short-lived bearer tokens with rotating refresh tokens for the mobile app, sharing the same authorization layer.

**Q5. What is inside a JWT, is it encrypted, and how do you verify one?**
Three base64url parts: a header (algorithm, type), a payload of claims (`sub`, `iss`, `aud`, `exp`, custom claims) and a signature. It is signed, not encrypted, so anyone can read the payload. Verify with an algorithm allowlist, check the signature, then `exp`, `nbf`, `iss` and `aud`.
*Follow-up:* What is the `alg: none` attack? A forged token declares no signature, and a careless library accepts it.

**Q6. HS256 vs RS256?**
HS256 signs and verifies with one shared secret, so every verifier could also mint tokens. RS256 (or ES256) signs with a private key and verifies with a public key published as a JWKS, so many services can verify without being able to issue.
*Follow-up:* How are keys rotated? Publish new public keys under a new `kid`, sign with the new key, and keep old keys until tokens signed with them expire.

**Q7. How do you log a user out, or revoke access, when you use JWTs?**
Keep access tokens short-lived, store refresh tokens server-side and revoke them on logout, and for immediate effect use a `jti` denylist or a token version on the user that you bump on logout-everywhere or password change. On the client, delete stored tokens.
*Follow-up:* Is that still stateless? Partly not, and that trade-off is the honest answer: revocation needs some state.

**Q8. Explain access and refresh tokens and refresh token rotation.**
The short-lived access token goes to the API on every request; the long-lived refresh token goes only to the token endpoint to get new access tokens. With rotation, each refresh returns a new refresh token and invalidates the old one; if an old one is reused, the server assumes theft and revokes the whole family.
*Follow-up:* How do you stop parallel requests from refreshing at once? A single-flight refresh on the client and a short grace period on the server.

**Q9. Where should a web app and a mobile app store tokens?**
Web: prefer an HttpOnly, Secure, SameSite cookie (or a session cookie with tokens kept server-side in a BFF); if you need a bearer token in JS, keep it in memory and restore it from a refresh cookie; avoid `localStorage` for long-lived tokens. Mobile: Keychain or Keystore through `expo-secure-store`, never AsyncStorage.
*Follow-up:* Does HttpOnly stop XSS? It stops token theft, not the injected script acting as the user while the page is open.

**Q10. Walk me through the OAuth 2.0 authorization code flow with PKCE.**
The app generates a code verifier and its SHA-256 challenge, then opens the authorization endpoint with client ID, redirect URI, scopes, `state` and the challenge. The user logs in and consents; the server redirects back with a code and the same `state`. The app posts the code and the verifier to the token endpoint; the server checks that the verifier hashes to the challenge and returns access, refresh and (with OIDC) ID tokens.
*Follow-up:* What does PKCE protect against? Interception of the authorization code, since redeeming it requires the verifier that never left the app.

**Q11. Why was the implicit flow deprecated?**
It returned access tokens directly in the redirect URL fragment, where browser history, logs, referrer headers and malicious scripts could capture them, and it could not safely issue refresh tokens. Authorization code with PKCE works for browser apps now and keeps tokens out of URLs.
*Follow-up:* What else is deprecated? The resource owner password grant, because the client handles the user's password.

**Q12. OAuth 2.0 vs OpenID Connect? ID token vs access token?**
OAuth 2.0 delegates access to APIs; OIDC adds authentication on top with the `openid` scope, an ID token, a userinfo endpoint and discovery. The ID token is a JWT for the client describing who logged in; the access token is for the API and carries scopes.
*Follow-up:* Which field identifies the user? The `sub` claim, unique per issuer, not the email.

**Q13. When would you use the client credentials flow?**
For machine-to-machine calls with no user involved: a backend syncing with a partner API, a scheduled job calling an internal service. The client authenticates with its own credentials and receives a short-lived access token limited by scopes.
*Follow-up:* Why not a static API key? Short-lived tokens limit the damage of a leak and centralize scope management, at the cost of more moving parts.

**Q14. RBAC vs ABAC (vs ReBAC)?**
RBAC grants permissions through roles: simple and auditable, but coarse. ABAC evaluates rules over attributes of the user, resource, action and context: fine-grained but harder to audit. ReBAC derives access from relationships, such as membership of a house or organization. Most apps combine roles within an org with ownership and state rules.
*Follow-up:* What is role explosion? Creating ever more specific roles to express rules that really depend on attributes or relationships.

**Q15. How do you implement org-scoped access control in a multi-tenant app?**
Users join organizations through memberships that carry a role. Each request resolves the current org, loads the membership (missing means 404), checks the role's permission and any resource rules, scopes every query by `organization_id`, and sets the org on new records from the membership. Back it with tests that try cross-org access, and optionally database row-level security.
*Follow-up:* Why 404 rather than 403 for another org's resource? It avoids confirming that the resource or org exists.

**Q16. What is IDOR, and how do you prevent it?**
Insecure Direct Object Reference: the API uses an ID from the request without checking that the caller may access that object, so changing `/bills/981` to `/bills/982` returns someone else's bill. Prevent it by loading records through the user's or org's scope, or through policy checks on every action, and by testing with multiple users.
*Follow-up:* Do UUIDs fix it? No. They make guessing harder, but IDs leak, so the authorization check is still required.

**Q17. What is session fixation?**
An attacker gets a victim to use a session ID the attacker knows (through a crafted link or a cookie set on a shared domain); if the app keeps that ID after login, the attacker is now logged in as the victim. Issue a new session ID at login, which Rails' `reset_session` and Django's `login()` do.
*Follow-up:* What else should invalidate sessions? Logout, password changes and privilege changes.

**Q18. Is a token-based API vulnerable to CSRF?**
Not if the token travels in an `Authorization` header that your code adds, because browsers never attach it automatically to cross-site requests. If you store the token in a cookie, the API is exposed to CSRF again and needs SameSite cookies and CSRF tokens.
*Follow-up:* What is the trade-off? Header tokens must be readable by JavaScript, which raises the stakes of XSS.

**Q19. How would you design API keys for partner integrations?**
High-entropy random keys with a recognizable prefix, shown once and stored as a SHA-256 hash, scoped to specific permissions and an organization, sent in a header, rate limited, with last-used tracking, multiple active keys for rotation and instant revocation. For higher security, sign requests with HMAC or use OAuth client credentials.
*Follow-up:* Why is a fast hash acceptable here? The keys are long and random, so brute-forcing a hash is infeasible, unlike human passwords.

**Q20. How does TOTP work, and what makes magic links safe?**
TOTP shares a secret at enrollment (QR code); both sides compute an HMAC of the secret and the current 30-second time step, truncated to six digits, and the server accepts a small drift window. Magic links are safe when the token is random, single-use, short-lived and stored hashed, and when the link lands on a confirmation step so link scanners cannot consume it.
*Follow-up:* What is stronger than TOTP? Passkeys (WebAuthn), which are bound to the site's origin and resist phishing.

## Further reading
- [RFC 6749: OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749), [RFC 7636: PKCE](https://www.rfc-editor.org/rfc/rfc7636) and [RFC 9700: OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [RFC 7519: JSON Web Token](https://www.rfc-editor.org/rfc/rfc7519) and [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) and [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
