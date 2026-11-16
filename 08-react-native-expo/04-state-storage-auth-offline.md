# State, Storage, Auth and Offline

> **Tier:** T1 (know cold) · **Related:** [TanStack Query](../06-react/04-data-fetching-tanstack-query.md), [State management in React](../06-react/03-state-management.md), [Forms and validation](../06-react/05-forms-and-validation.md), [Rails API development](../05-ruby-on-rails/10-api-development.md), [Rails auth](../05-ruby-on-rails/11-authentication-authorization.md), [Web auth concepts](../04-web-fundamentals/06-authentication-authorization.md), [Navigation](03-navigation-expo-router.md)
> **On your resume:** the Coloc resident app (React Native, Expo, TypeScript) on top of the Colochousing Rails 7 REST API: the sessions and data behind events, tasks, split bills and rent payments.

## Quick revision
- Split state by owner: **server state** (bills, events: TanStack Query), **client state** (UI, preferences: `useState`, context, Zustand), **form state** (React Hook Form), **persisted state** (storage), **secrets** (SecureStore), **navigation state** (the router).
- TanStack Query works unchanged in React Native; wire `focusManager` to `AppState` and `onlineManager` to NetInfo so data refreshes on resume and pauses offline.
- Context is fine for rarely changing values (session, theme). For frequently changing shared state, use Zustand with selectors so only subscribers re-render.
- **AsyncStorage:** unencrypted, async, strings, works in Expo Go. **expo-secure-store:** Keychain / Keystore-backed, async, small values: tokens. **MMKV:** synchronous, very fast, JSI-based, optional encryption, needs a development build.
- Tokens: short-lived **access token in memory**, long-lived **refresh token in SecureStore**, rotated on every refresh and revocable on the server. Never AsyncStorage for tokens.
- API client: attach `Authorization: Bearer`, on 401 refresh **once** (single-flight promise shared by parallel requests), retry, and sign out if the refresh fails.
- Logout: revoke on the server, delete stored tokens, `queryClient.clear()`, unregister the push token, reset navigation.
- Biometrics (`expo-local-authentication`) only unlock something already on the device; they are not server authentication.
- OAuth on mobile: authorization code flow with **PKCE** through the system browser (`expo-auth-session`); the backend verifies and issues its own tokens.
- Offline basics: persist the query cache, show cached data with a "last updated" hint, optimistic updates with rollback, paused mutations that resume when back online, idempotency keys so retries do not duplicate.
- Forms: React Hook Form's `Controller` wraps `TextInput` (`onChangeText`), Zod schema via `zodResolver`, `handleSubmit` from a `Pressable`.
- `AppState` (`active`, `background`, `inactive` on iOS) drives refetch on foreground, locking after inactivity and pausing work.
- Security: no secrets in the bundle, HTTPS only, consider certificate pinning carefully, assume rooted or jailbroken devices exist, and enforce everything on the server.

## 1. Kinds of state in a mobile app

| State | Example | Where it lives |
|---|---|---|
| Server state | Bills, events, tasks, the profile | TanStack Query cache (optionally persisted) |
| Client / UI state | Selected tab filter, a sheet being open | `useState`, context or Zustand |
| Form state | The "new bill" form while typing | React Hook Form |
| Persisted preferences | Language, onboarding seen, last house | AsyncStorage or MMKV |
| Secrets | Refresh token | expo-secure-store |
| Navigation state | Current screen and params | Expo Router / React Navigation |

Most bugs come from mixing these: copying server data into a client store (now it goes stale), or keeping a token in a place built for preferences. More background in [React state management](../06-react/03-state-management.md).

## 2. Server state with TanStack Query in React Native

The hooks are identical to the web. Mobile has no browser focus or online events, so connect them yourself, once, at startup:

```ts
// lib/query-client.ts
import { AppState, Platform, type AppStateStatus } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { QueryClient, focusManager, onlineManager } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000, retry: 2 },
  },
});

// Pause queries and mutations while offline, resume on reconnect
onlineManager.setEventListener((setOnline) =>
  NetInfo.addEventListener((state) => setOnline(!!state.isConnected)),
);

// Treat "app came to the foreground" like "window regained focus": stale queries refetch
AppState.addEventListener('change', (status: AppStateStatus) => {
  if (Platform.OS !== 'web') focusManager.setFocused(status === 'active');
});
```

Everything else (query keys, invalidation, optimistic updates, infinite lists) is covered in [Data fetching and TanStack Query](../06-react/04-data-fetching-tanstack-query.md). Mobile-specific: refetch on screen focus with `useFocusEffect` when a screen must be fresh, and pick `staleTime` per resource (bills change rarely, task completion often).

## 3. Client state: Context vs Zustand

- **Context** suits values that rarely change: the session, theme, current house. Every consumer re-renders when the value changes, so keep context values small and memoized.
- **Zustand** suits shared state that changes often or is read in many places. Components subscribe to slices with selectors and re-render only when their slice changes. Its `persist` middleware saves state to storage.

```ts
// stores/preferences.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

type Preferences = {
  language: 'en' | 'fr' | 'nl';
  billsFilter: 'all' | 'unpaid';
  setLanguage: (language: Preferences['language']) => void;
  setBillsFilter: (filter: Preferences['billsFilter']) => void;
};

export const usePreferences = create<Preferences>()(
  persist(
    (set) => ({
      language: 'en',
      billsFilter: 'unpaid',
      setLanguage: (language) => set({ language }),
      setBillsFilter: (billsFilter) => set({ billsFilter }),
    }),
    { name: 'preferences', storage: createJSONStorage(() => AsyncStorage) },
  ),
);

// In a component: subscribe to one field only
const billsFilter = usePreferences((s) => s.billsFilter);
```

Never put tokens in a persisted Zustand store backed by AsyncStorage: that writes them to unencrypted storage.

## 4. Storage: AsyncStorage vs SecureStore vs MMKV

| | AsyncStorage | expo-secure-store | MMKV (`react-native-mmkv`) |
|---|---|---|---|
| Backing | Unencrypted key-value storage in the app sandbox | iOS Keychain; Android values encrypted with keys in the Android Keystore | Memory-mapped files in C++, accessed through JSI |
| Encrypted | No | Yes | Optional (with an encryption key) |
| API | Async, string values | Async (`getItemAsync`, `setItemAsync`, `deleteItemAsync`), strings | Synchronous; strings, numbers, booleans, buffers |
| Speed | Slowest | Slow per call (crypto), fine for a handful of reads | Fastest |
| Size | Fine for moderate data (Android has historically capped total size at a few MB) | Small values only (historically about 2 KB per item) | Large data is fine |
| Expo Go | Yes | Yes | No, needs a development build |
| Use for | Non-sensitive preferences, persisted query cache | Refresh tokens, other credentials | Hot-path state, large persisted stores |

Decision rule: sensitive means SecureStore; large or hot means MMKV (keep its encryption key in SecureStore if the data is sensitive); simple and non-sensitive means AsyncStorage; relational data you query offline means `expo-sqlite`.

```ts
import * as SecureStore from 'expo-secure-store';

await SecureStore.setItemAsync('auth.refreshToken', token, {
  // Readable only while the device is unlocked, never migrated to another device by backups
  keychainAccessible: SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
});
const stored = await SecureStore.getItemAsync('auth.refreshToken'); // string | null
await SecureStore.deleteItemAsync('auth.refreshToken');
```

Two platform quirks to know: Keychain items can **survive an uninstall** on iOS, so a reinstalled app may find an old token (clear secure items on the first launch after install, using a flag in AsyncStorage, which is wiped on uninstall). On Android, restoring a backup to a new device can bring back encrypted values that can no longer be decrypted, so handle read failures by treating the user as logged out (the expo-secure-store docs describe backup configuration).

## 5. Authentication against a Rails API

A typical token design (adapt it to what your backend does):

1. `POST /api/v1/auth/login` with email and password returns `{ access_token, refresh_token, user }`.
2. The **access token** is short-lived (minutes) and sent as `Authorization: Bearer <token>`. Keep it in memory.
3. The **refresh token** is long-lived, random and opaque. The server stores only its hash, per device, so it can revoke one device. Store it in SecureStore.
4. `POST /api/v1/auth/refresh` exchanges a refresh token for new tokens and **rotates** the refresh token. If an already-used refresh token shows up again, the server revokes that whole token family (someone copied it).
5. `DELETE /api/v1/auth/session` revokes the device's refresh token on logout.

Why not cookies: React Native has no browser cookie model to rely on, bearer tokens are simple to attach and revoke, and CSRF does not apply to a bearer header. On the Rails side this is often Devise with devise-jwt, or `has_secure_password` with a custom token model; see [Rails API development](../05-ruby-on-rails/10-api-development.md) and [Rails authentication](../05-ruby-on-rails/11-authentication-authorization.md).

### Going deeper: what the server must do
- Hash refresh tokens at rest, like passwords: a database leak should not leak sessions.
- Bind refresh tokens to a device record (name, platform, last used) so users can see and revoke sessions.
- Rotate on use and detect reuse; expire unused sessions; rate-limit login and refresh endpoints.
- Return `401` for expired or invalid access tokens and `403` for "authenticated but not allowed", so the client knows when refreshing makes sense.

## 6. An API client with token refresh

```ts
// lib/api.ts
import * as SecureStore from 'expo-secure-store';

const API_URL = process.env.EXPO_PUBLIC_API_URL;
const REFRESH_KEY = 'auth.refreshToken';

type Tokens = { access_token: string; refresh_token: string };
type ApiInit = Omit<RequestInit, 'headers'> & { headers?: Record<string, string> };

let accessToken: string | null = null; // memory only, short-lived
let refreshInFlight: Promise<string | null> | null = null;
let onSessionExpired: () => void = () => {};

export const setSessionExpiredHandler = (fn: () => void) => {
  onSessionExpired = fn;
};

export async function saveTokens(tokens: Tokens) {
  accessToken = tokens.access_token;
  await SecureStore.setItemAsync(REFRESH_KEY, tokens.refresh_token);
}

export async function clearTokens() {
  accessToken = null;
  await SecureStore.deleteItemAsync(REFRESH_KEY);
}

async function refreshAccessToken(): Promise<string | null> {
  const refreshToken = await SecureStore.getItemAsync(REFRESH_KEY);
  if (!refreshToken) return null;
  const res = await fetch(`${API_URL}/api/v1/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refresh_token: refreshToken }),
  });
  if (!res.ok) return null; // expired or revoked: a new login is needed
  const tokens = (await res.json()) as Tokens;
  await saveTokens(tokens); // the server rotated the refresh token
  return tokens.access_token;
}

export async function api(path: string, init: ApiInit = {}, isRetry = false): Promise<Response> {
  const res = await fetch(`${API_URL}${path}`, {
    ...init,
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
      ...init.headers,
      ...(accessToken ? { Authorization: `Bearer ${accessToken}` } : {}),
    },
  });
  if (res.status !== 401 || isRetry) return res;

  // Single flight: ten parallel 401s share one refresh request
  refreshInFlight ??= refreshAccessToken().finally(() => {
    refreshInFlight = null;
  });
  const newToken = await refreshInFlight;
  if (!newToken) {
    await clearTokens();
    onSessionExpired(); // the session provider sets user = null and the auth guard redirects
    return res;
  }
  return api(path, init, true); // retry the original request once
}
```

Why single-flight matters: with rotation, two parallel refreshes would send the same refresh token twice; the second looks like reuse and the server revokes the session, logging the user out at random. The same logic works as an Axios response interceptor if the project uses Axios.

## 7. Session provider, logout and revocation

```tsx
// lib/session.tsx
import { createContext, useCallback, useContext, useEffect, useMemo, useState, type ReactNode } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { api, clearTokens, saveTokens, setSessionExpiredHandler } from './api';

type User = { id: number; name: string };
type SessionValue = {
  user: User | null;
  isLoading: boolean;
  signIn: (email: string, password: string) => Promise<void>;
  signOut: () => Promise<void>;
};

const SessionContext = createContext<SessionValue | null>(null);

export function SessionProvider({ children }: { children: ReactNode }) {
  const queryClient = useQueryClient();
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    setSessionExpiredHandler(() => setUser(null));
    // Restore: the first call has no access token, gets a 401, refreshes from SecureStore and retries
    api('/api/v1/me')
      .then(async (res) => setUser(res.ok ? ((await res.json()) as User) : null))
      .catch(() => setUser(null)) // offline at launch: you may prefer a cached profile here
      .finally(() => setIsLoading(false));
  }, []);

  const signIn = useCallback(async (email: string, password: string) => {
    const res = await api('/api/v1/auth/login', { method: 'POST', body: JSON.stringify({ email, password }) });
    if (!res.ok) throw new Error(res.status === 401 ? 'Wrong email or password' : 'Login failed');
    const body = await res.json();
    await saveTokens(body);
    setUser(body.user);
  }, []);

  const signOut = useCallback(async () => {
    await api('/api/v1/auth/session', { method: 'DELETE' }).catch(() => undefined); // revoke on the server
    await clearTokens();
    queryClient.clear(); // the next person on this phone must not see cached data
    setUser(null); // protected routes redirect to login
  }, [queryClient]);

  const value = useMemo(() => ({ user, isLoading, signIn, signOut }), [user, isLoading, signIn, signOut]);
  return <SessionContext.Provider value={value}>{children}</SessionContext.Provider>;
}

export function useSession() {
  const ctx = useContext(SessionContext);
  if (!ctx) throw new Error('useSession must be used inside SessionProvider');
  return ctx;
}
```

Logout checklist: revoke the refresh token server-side, delete local tokens, clear the query cache and persisted user data, unregister this device's push token ([Push notifications](05-push-notifications.md)), cancel scheduled local notifications, and let the router remove protected screens from history ([Navigation](03-navigation-expo-router.md)).

## 8. Biometric unlock

```ts
import * as LocalAuthentication from 'expo-local-authentication';

export async function unlockWithBiometrics(): Promise<boolean> {
  const [hasHardware, isEnrolled] = await Promise.all([
    LocalAuthentication.hasHardwareAsync(),
    LocalAuthentication.isEnrolledAsync(),
  ]);
  if (!hasHardware || !isEnrolled) return false; // fall back to the password screen
  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: 'Unlock the app',
    cancelLabel: 'Use password',
  });
  return result.success;
}
```

Biometrics prove that the device owner is present; they do not authenticate with your server. Typical use: after 5 minutes in the background, show a lock screen that calls `unlockWithBiometrics()` before revealing data. For stronger binding, SecureStore can require authentication to read an item (`requireAuthentication: true`), which ties the refresh token to the user's biometrics. iOS needs a Face ID usage description, set through the module's config plugin, so test in a development build.

## 9. OAuth and social login

Mobile apps are public clients: they cannot keep a client secret. The standard is the **authorization code flow with PKCE** in the system browser (never an embedded WebView, which could capture passwords):

```tsx
import * as AuthSession from 'expo-auth-session';
import * as WebBrowser from 'expo-web-browser';

WebBrowser.maybeCompleteAuthSession(); // finishes the browser redirect when the app reopens

export function useOidcLogin() {
  const discovery = AuthSession.useAutoDiscovery('https://accounts.example.com');
  const [request, response, promptAsync] = AuthSession.useAuthRequest(
    {
      clientId: 'resident-app',
      redirectUri: AuthSession.makeRedirectUri({ scheme: 'myapp' }),
      scopes: ['openid', 'profile', 'email'], // PKCE is on by default
    },
    discovery,
  );
  // When response?.type === 'success': send response.params.code and request.codeVerifier to your
  // backend, which exchanges them with the provider and returns your own access and refresh tokens.
  return { request, response, promptAsync };
}
```

Sign in with Google is often done with the native Google Sign-In library instead, and Sign in with Apple with `expo-apple-authentication`. Either way, the backend verifies the provider's ID token (signature, audience, expiry) and then issues its own session tokens. Apple's guidelines expect Sign in with Apple, or an equivalent privacy-focused option, when you offer third-party logins (check the current guidelines). Provider setup differs a lot, so confirm details in each provider's docs.

## 10. Offline-first basics

You rarely need a full sync engine; these steps cover most apps:

1. **Persist the query cache** so a cold start offline shows the last known data:

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';
import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';

const persister = createAsyncStoragePersister({ storage: AsyncStorage });

export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <PersistQueryClientProvider
      client={queryClient} // set gcTime at least as long as maxAge, or entries are dropped before saving
      persistOptions={{ persister, maxAge: 24 * 60 * 60 * 1000 }}
      onSuccess={() => queryClient.resumePausedMutations().then(() => queryClient.invalidateQueries())}
    >
      {children}
    </PersistQueryClientProvider>
  );
}
```

2. **Show connectivity:** `useNetInfo()` from `@react-native-community/netinfo` drives an "Offline, showing saved data" banner; `isInternetReachable` distinguishes "on Wi-Fi" from "actually online".
3. **Optimistic updates** for low-risk actions (marking a task done): update the cache in `onMutate`, roll back in `onError`, invalidate in `onSettled`.
4. **Queue mutations:** while offline, TanStack Query pauses mutations and resumes them on reconnect. To survive an app restart, register the mutation function with `queryClient.setMutationDefaults(['tasks', 'complete'], { mutationFn })` so persisted paused mutations know what to run, as in the provider above.
5. **Idempotency:** generate a UUID per action (`Crypto.randomUUID()` from `expo-crypto`) and send it as an `Idempotency-Key` header; the Rails API stores processed keys, so a retried request does not create a second task or bill.
6. **Conflicts:** last write wins is acceptable for preferences; for shared data, let the server validate and return a conflict (`409`) that the app resolves by refetching.
7. **Never queue money:** payments and settlements require connectivity and an immediate server answer.

## 11. Forms with React Hook Form and Zod

`TextInput` is not a DOM input, so wrap it with `Controller` and map `onChangeText`:

```tsx
import { Controller, useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Pressable, Text, TextInput, View } from 'react-native';

const schema = z.object({
  title: z.string().min(2, 'Give the bill a name'),
  amount: z.string().regex(/^\d+([.,]\d{1,2})?$/, 'Enter an amount like 12.50'),
});
type FormValues = z.infer<typeof schema>;

export function NewBillForm({ onSubmit }: { onSubmit: (values: FormValues) => Promise<void> }) {
  const { control, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormValues>({
    resolver: zodResolver(schema),
    defaultValues: { title: '', amount: '' },
  });

  return (
    <View style={{ gap: 12, padding: 16 }}>
      <Controller
        control={control}
        name="title"
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput accessibilityLabel="Title" value={value} onChangeText={onChange} onBlur={onBlur} />
        )}
      />
      {errors.title && <Text style={{ color: 'crimson' }}>{errors.title.message}</Text>}
      <Controller
        control={control}
        name="amount"
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput accessibilityLabel="Amount" keyboardType="decimal-pad" value={value} onChangeText={onChange} onBlur={onBlur} />
        )}
      />
      {errors.amount && <Text style={{ color: 'crimson' }}>{errors.amount.message}</Text>}
      <Pressable accessibilityRole="button" disabled={isSubmitting} onPress={handleSubmit(onSubmit)}>
        <Text>{isSubmitting ? 'Saving...' : 'Save bill'}</Text>
      </Pressable>
    </View>
  );
}
```

Keep amounts as strings in the form and convert to integer cents in one tested function ([Payments and split bills](06-payments-and-split-bills.md)). Map Rails validation errors (`422` with field errors) back onto fields with `setError`. More in [Forms and validation](../06-react/05-forms-and-validation.md).

## 12. AppState: foreground, background and auto-lock

```ts
import { useEffect, useRef } from 'react';
import { AppState } from 'react-native';

const LOCK_AFTER_MS = 5 * 60 * 1000;

export function useAutoLock(lock: () => void) {
  const backgroundedAt = useRef<number | null>(null);

  useEffect(() => {
    const sub = AppState.addEventListener('change', (status) => {
      if (status === 'background') backgroundedAt.current = Date.now();
      if (status === 'active' && backgroundedAt.current !== null) {
        if (Date.now() - backgroundedAt.current > LOCK_AFTER_MS) lock(); // show the biometric lock screen
        backgroundedAt.current = null;
      }
    });
    return () => sub.remove();
  }, [lock]);
}
```

States: `active` (foreground), `background`, and `inactive` on iOS (transitions such as the app switcher or an incoming call). The focus manager wiring in section 2 already refetches stale queries when the app becomes active. JS does not keep running in the background (apart from specific background tasks), so save drafts when the app goes to the background.

## 13. Security notes for a mobile client

- **No secrets in the bundle.** Assume anyone can read your JS and `EXPO_PUBLIC_*` values. Payment secret keys and third-party API secrets stay on the server.
- **HTTPS only.** iOS App Transport Security and Android's cleartext defaults block plain HTTP in production builds; do not weaken them.
- **Certificate pinning** protects against intercepting proxies with trusted rogue certificates, but a pinned certificate that rotates unexpectedly bricks every installed app until users update. If you pin, pin public keys, ship backup pins and have a remote way to disable it. Expo has no built-in pinning, so it means a community library or native configuration through a config plugin.
- **Jailbroken and rooted devices exist.** Detection libraries are easy to bypass, so treat every client as untrusted: authorization, amount calculations and validation happen on the Rails API. App attestation services (Play Integrity on Android, App Attest on iOS) raise the bar for high-risk actions.
- **Do not log tokens or personal data**, including in crash reports and analytics breadcrumbs.
- **Sensitive screens:** consider hiding content in the app switcher snapshot (`expo-screen-capture` can block screenshots where that is justified).

## Common pitfalls
- Storing tokens in AsyncStorage or a persisted store, which is plain text on disk.
- Refreshing in parallel from several failing requests, so rotation logs the user out randomly.
- Retrying a failed refresh forever, or retrying `403` responses that a refresh cannot fix.
- Forgetting `queryClient.clear()` and the push token on logout.
- The login screen flashing at startup because the session is restored after the first render.
- Trusting the iOS Keychain to be empty after a reinstall.
- Copying query data into Zustand or context, where it never refreshes.
- Offline retries without idempotency keys, creating duplicate tasks or bills.
- Treating biometrics as authentication, or blocking users without biometrics instead of falling back to a password.

## How to talk about this in an interview
- Start from the split: "Server data lives in TanStack Query or a similar cache, client state stays local or in a small store, tokens live in secure storage, and the API client owns refresh logic."
- For auth, walk through the flow end to end: login, where each token lives, refresh with rotation and single flight, logout with server revocation. Name the security reason behind each choice.
- Connect to the Coloc resident app and its Rails 7 REST API without inventing details; the exact token scheme is yours to fill in.

> **Fill in:** how the resident app authenticates against the Rails API (token type, lifetimes, refresh flow) and where tokens are stored on the device.

> **Fill in:** which state tools the app uses (TanStack Query, context, Zustand, Redux, plain hooks) and why.

> **Fill in:** how the app behaves offline or on a bad connection, and one bug you fixed in that area.

## Interview questions

**Q1. How do you organize state in a React Native app?**
By owner: server data in a cache such as TanStack Query, local UI state in components, shared client state in context (rarely changing) or Zustand (frequently changing), forms in React Hook Form, preferences in AsyncStorage or MMKV, secrets in SecureStore and navigation state in the router. Each tool then solves the right problem, and server data never goes stale inside a client store.
*Follow-up:* What goes wrong when you mix them? A copied profile in a global store that never updates after editing, or tokens persisted in plain text.

**Q2. How do you set up TanStack Query for React Native?**
Create one `QueryClient`, wrap the app in its provider, connect `onlineManager` to NetInfo so queries pause offline, and connect `focusManager` to `AppState` so stale queries refetch when the app returns to the foreground. Optionally persist the cache to AsyncStorage for offline starts.
*Follow-up:* Why is focus handling needed? Mobile has no window focus event, and users return after minutes or hours.

**Q3. Context or Zustand?**
Context for small, rarely changing values such as the session, theme or current house; every consumer re-renders when the value changes. Zustand for shared state that changes often or is read widely, because components subscribe through selectors and re-render only when their slice changes; it also persists easily.
*Follow-up:* Would you put server data in Zustand? No, use a server-state cache with invalidation.

**Q4. SecureStore vs AsyncStorage vs MMKV?**
AsyncStorage is unencrypted, async and string-based, fine for preferences and cached data, and works in Expo Go. SecureStore uses the Keychain and Keystore, is async and meant for small secrets such as refresh tokens. MMKV is synchronous, very fast through JSI, supports encryption and suits hot or large data, but needs a development build.
*Follow-up:* Encrypted MMKV, where does the key live? In SecureStore.

**Q5. How do you store auth tokens on a device?**
The short-lived access token stays in memory; the long-lived refresh token goes into SecureStore with a restrictive accessibility option, and the server stores only its hash so it can rotate and revoke it. Nothing token-related goes into AsyncStorage, logs or crash reports.
*Follow-up:* Why not keep the access token in SecureStore too? You can, but memory is enough for a token that lives minutes, and it avoids slow secure reads on every request.

**Q6. Walk me through login and token refresh against a Rails API.**
Login posts credentials and receives an access token, a refresh token and the user. Requests carry `Authorization: Bearer <access>`. On a `401`, the client calls the refresh endpoint once with the stored refresh token, saves the rotated tokens and retries the request; if the refresh fails, it clears tokens and sends the user to login. Logout revokes the refresh token on the server.
*Follow-up:* What does rotation protect against? A stolen refresh token: reuse of an old token reveals the theft, and the server revokes the token family.

**Q7. Several requests fail with 401 at the same time. What happens in your client?**
They share one refresh promise (single flight): the first `401` starts the refresh, the others await the same promise, then all retry with the new token. Without it, parallel refreshes send the same rotated token twice and the server treats the second as reuse and revokes the session.
*Follow-up:* How do you avoid infinite loops? Retry each request at most once, and never refresh from the refresh call itself.

**Q8. What should happen when a user logs out?**
Revoke the refresh token on the server, delete local tokens, clear the query cache and persisted user data, unregister the device's push token, cancel scheduled local notifications, and reset navigation so protected screens leave the history. Then the next person using the phone starts clean.
*Follow-up:* What if the revoke call fails offline? Still clear everything locally; the server-side token expires or can be revoked from another session.

**Q9. How would you add biometric unlock, and is it authentication?**
Check `hasHardwareAsync` and `isEnrolledAsync`, then call `authenticateAsync` when the app opens or returns after a timeout, falling back to the password. It proves the device owner is present, which is a local gate, not server authentication. Binding the refresh token to biometrics through SecureStore's `requireAuthentication` makes it stronger.
*Follow-up:* What changes if the user adds a new fingerprint? Keys bound to biometrics can be invalidated, so handle a failed read as "log in again".

**Q10. How does OAuth or social login work in a mobile app, and why PKCE?**
Use the authorization code flow in the system browser: the app sends a code challenge, the provider redirects back to the app's scheme with a code, and the backend exchanges code plus verifier for tokens, verifies the identity and issues its own session tokens. PKCE replaces the client secret a mobile app cannot keep, so an intercepted code is useless without the verifier.
*Follow-up:* Why not a WebView? The app could read the password typed into it, users cannot see the real URL, and providers block embedded views.

**Q11. How would you make the app usable offline?**
Persist the query cache so cold starts show the last data, show an offline banner from NetInfo, use optimistic updates for low-risk actions, let mutations pause and resume (with `setMutationDefaults` so they survive restarts), send idempotency keys, and handle conflicts on the server. Payments always require a live connection.
*Follow-up:* When do you need a real sync engine? When users create and edit lots of shared data offline for long periods; then consider SQLite-based sync solutions.

**Q12. How do you make sure a queued mutation is not applied twice?**
Generate an idempotency key per user action and send it with every attempt; the Rails API records processed keys (for example a unique index on the key per user) and returns the original result for duplicates. The same mechanism protects double taps and network retries.
*Follow-up:* Where else do idempotency keys matter? Payment provider calls such as creating a PaymentIntent.

**Q13. How do you build a validated form in React Native?**
React Hook Form with a Zod schema through `zodResolver`; each `TextInput` is wrapped in `Controller`, mapping `onChangeText` to `onChange`. Submit with `handleSubmit(onSubmit)` from a `Pressable`, show field errors under inputs, disable the button while `isSubmitting`, and map server-side `422` errors back with `setError`.
*Follow-up:* Why not a controlled `useState` per field? It works for tiny forms, but validation, dirty tracking and errors become boilerplate.

**Q14. How do you refetch data when the app returns to the foreground?**
Listen to `AppState` changes and call `focusManager.setFocused(status === 'active')`, so TanStack Query refetches stale queries on resume. For screen-level freshness, use `useFocusEffect` to invalidate when a screen regains focus.
*Follow-up:* What else would you do on resume? Re-check the session, refresh badges and lock the app after a long background period.

**Q15. What security measures matter on the mobile client?**
No secrets in the bundle, HTTPS only, tokens in secure storage and out of logs, careful certificate pinning with backup pins if used, attestation for high-risk actions, and treating the client as untrusted because rooted or jailbroken devices can bypass any check. All authorization and money calculations stay on the server.
*Follow-up:* Why is pinning risky? An unexpected certificate change breaks every installed app until users update.

**Q16. Users who reinstall the app are logged in as a previous account. What is happening?**
iOS Keychain entries can survive an uninstall, so the reinstalled app finds the old refresh token in SecureStore. Store a "has launched" flag in AsyncStorage (wiped on uninstall); if it is missing at startup, delete the secure items before restoring the session.
*Follow-up:* Does Android behave the same? Uninstall usually clears app data, but backup and restore can bring values back, possibly undecryptable.

**Q17. Where would you store the draft of a long form, such as a maintenance request?**
In AsyncStorage or MMKV under a per-user key, saved on change with a debounce and when the app goes to the background, restored when the screen opens, and deleted after a successful submit. Drafts are not secrets, so SecureStore is unnecessary, but clear them on logout.
*Follow-up:* Why save on background? The OS can kill a backgrounded app without warning.

## Further reading
- [TanStack Query: React Native](https://tanstack.com/query/latest/docs/framework/react/react-native)
- [expo-secure-store](https://docs.expo.dev/versions/latest/sdk/securestore/)
- [Authentication with expo-auth-session](https://docs.expo.dev/versions/latest/sdk/auth-session/)
- [expo-local-authentication](https://docs.expo.dev/versions/latest/sdk/local-authentication/)
