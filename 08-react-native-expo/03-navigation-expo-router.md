# Navigation and Expo Router

> **Tier:** T1 (know cold) · **Related:** [React Native fundamentals](01-react-native-fundamentals.md), [State, storage, auth and offline](04-state-storage-auth-offline.md), [Push notifications](05-push-notifications.md), [Routing on the web](../06-react/08-routing.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md)
> **On your resume:** the screens of the Coloc resident app (events, tasks, split bills, rent payments) reached through tabs, deep links and push notification taps.

## Quick revision
- Mobile navigation patterns: **stack** (push and pop, back gesture), **tabs** (parallel top-level sections, each often with its own stack), **drawer** (side menu), **modal** (a focused task presented over the current screen).
- Navigation state is a tree of navigators: root stack, then tabs, then a stack per tab. The Android back button pops the innermost stack first.
- **React Navigation** is the underlying library (`NavigationContainer`, native stack, bottom tabs, drawer). **Expo Router** adds file-based routing on top and makes every screen URL-addressable.
- Expo Router: files in `app/` are routes. `_layout.tsx` defines the navigator for its folder, `index.tsx` is the folder's default route, `(group)` folders organize without adding a URL segment, `[id].tsx` is dynamic, `[...rest].tsx` catches everything, `+not-found.tsx` handles unmatched URLs.
- Navigate with `<Link href>` or `useRouter()` / `router`: `push`, `replace`, `back`, `dismissAll`, `setParams`.
- Read params with `useLocalSearchParams<{ id: string }>()`. Params are **strings** (arrays for catch-all); convert and validate them.
- Typed routes (`experiments.typedRoutes` in the app config) type-check every `href`.
- Modals: a stack screen with `presentation: 'modal'` (or `formSheet`, `transparentModal`).
- Auth: `Stack.Protected guard={...}` in recent versions (Expo Router v5, SDK 53+), or a `<Redirect>` in a group layout. Keep the splash screen up while restoring the session.
- After logout, protected screens must leave the history: guards do it automatically; otherwise use `router.replace` plus `router.dismissAll()`, and clear cached user data.
- Deep links: a custom `scheme` (`myapp://events/42`) maps straight to routes. Universal links (iOS, `associatedDomains` + `apple-app-site-association`) and app links (Android, intent filters with `autoVerify` + `assetlinks.json`) use `https://` URLs.
- Push tap: put an internal path in the notification `data` (`{ url: '/bills/12' }`) and `router.push` it from the response listener, plus the last response for cold starts.
- Pass IDs in params, not whole objects; fetch details by ID (the query cache makes it instant).
- Performance: native stack transitions, lazy tabs (default), `freezeOnBlur`, prefetch data before navigating, keep screen mount work light.

## 1. Navigation concepts

| Pattern | Behaves like | Use for |
|---|---|---|
| Stack | A history you push onto and pop off, with native slide transitions and the back gesture | Drilling down: list, then detail, then edit |
| Tabs | Parallel sections; switching keeps each tab's state and scroll position | Top-level areas: Home, Events, Bills, Profile |
| Drawer | A side menu sliding over the content | Many sections, admin-style apps; less common in modern iOS design |
| Modal | A screen presented over the current context, dismissed back to where you were | Focused tasks: create a task, pay a bill, pick a date |

Differences from web routing: there is no address bar, every stack keeps its own history, screens below the top one stay mounted (their state survives), transitions are native animations, and Android adds a hardware or gesture back action that must do something sensible on every screen.

## 2. React Navigation basics

Expo Router generates this for you, but you should recognize the underlying API (many existing apps use it directly):

```tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator, type NativeStackScreenProps } from '@react-navigation/native-stack';

type RootStackParamList = {
  Events: undefined;
  EventDetail: { eventId: number };
};

const Stack = createNativeStackNavigator<RootStackParamList>();

function EventsScreen({ navigation }: NativeStackScreenProps<RootStackParamList, 'Events'>) {
  return <EventList onOpen={(id) => navigation.navigate('EventDetail', { eventId: id })} />;
}

function EventDetailScreen({ route }: NativeStackScreenProps<RootStackParamList, 'EventDetail'>) {
  return <EventDetail id={route.params.eventId} />;
}

export default function App() {
  return (
    <NavigationContainer linking={{ prefixes: ['myapp://'], config: { screens: { EventDetail: 'events/:eventId' } } }}>
      <Stack.Navigator>
        <Stack.Screen name="Events" component={EventsScreen} />
        <Stack.Screen name="EventDetail" component={EventDetailScreen} options={{ title: 'Event' }} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

Key ideas: navigators are components, screens receive `navigation` and `route` (or use `useNavigation` and `useRoute`), params are typed through a param list, and deep linking needs a manual `linking` config that maps URLs to screens. React Navigation 7 also offers a static configuration API that infers these types for you.

## 3. Expo Router: routes are files

```text
app/
  _layout.tsx             root Stack, providers, auth gate
  login.tsx               /login
  (tabs)/
    _layout.tsx           Tabs navigator
    index.tsx             /            (home tab)
    events/
      _layout.tsx         Stack inside the Events tab
      index.tsx           /events
      [id].tsx            /events/42
    bills.tsx             /bills
    profile.tsx           /profile
  bills/
    [billId].tsx          /bills/12    (pushed on the root stack, above the tabs)
  new-task.tsx            /new-task    (presented as a modal)
  help/
    [...slug].tsx         /help/payments/sepa  (catch-all, slug = ['payments', 'sepa'])
  +not-found.tsx          any unmatched URL
```

| Convention | Meaning |
|---|---|
| `_layout.tsx` | Wraps the routes in its folder: a `Stack`, `Tabs`, `Drawer` or `Slot` (render the child without a navigator) |
| `index.tsx` | The default route of a folder |
| `(name)/` | A group: organizes files and layouts without adding a URL segment |
| `[param].tsx` | A dynamic segment, read with `useLocalSearchParams` |
| `[...param].tsx` | Catch-all: the param is an array of segments |
| `+not-found.tsx` | Rendered when no route matches (bad deep links land here) |

Because every screen has a URL, deep links, push notification taps and web support come almost for free, and the folder tree documents the app's structure.

## 4. Layouts

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import * as SplashScreen from 'expo-splash-screen';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '../lib/query-client';
import { SessionProvider, useSession } from '../lib/session';

SplashScreen.preventAutoHideAsync(); // keep the native splash until we know who the user is

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <SessionProvider>
        <RootNavigator />
      </SessionProvider>
    </QueryClientProvider>
  );
}

function RootNavigator() {
  const { user, isLoading } = useSession();

  useEffect(() => {
    if (!isLoading) SplashScreen.hideAsync();
  }, [isLoading]);

  if (isLoading) return null; // the splash screen is still visible

  return (
    <Stack screenOptions={{ headerBackTitle: 'Back' }}>
      <Stack.Protected guard={!!user}>
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="bills/[billId]" options={{ title: 'Bill' }} />
        <Stack.Screen name="new-task" options={{ presentation: 'modal', title: 'New task' }} />
      </Stack.Protected>
      <Stack.Protected guard={!user}>
        <Stack.Screen name="login" options={{ headerShown: false }} />
      </Stack.Protected>
    </Stack>
  );
}
```

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#2f6fed' }}>
      <Tabs.Screen name="index" options={{ title: 'Home', tabBarIcon: ({ color, size }) => <Ionicons name="home" color={color} size={size} /> }} />
      <Tabs.Screen name="events" options={{ title: 'Events', headerShown: false, tabBarIcon: ({ color, size }) => <Ionicons name="calendar" color={color} size={size} /> }} />
      <Tabs.Screen name="bills" options={{ title: 'Bills', tabBarIcon: ({ color, size }) => <Ionicons name="wallet" color={color} size={size} /> }} />
      <Tabs.Screen name="profile" options={{ title: 'Profile', tabBarIcon: ({ color, size }) => <Ionicons name="person" color={color} size={size} /> }} />
    </Tabs>
  );
}
```

```tsx
// app/(tabs)/events/_layout.tsx: a stack inside the Events tab, so details keep the tab bar visible
import { Stack } from 'expo-router';

export const unstable_settings = { initialRouteName: 'index' }; // a deep link to /events/42 still gets the list underneath

export default function EventsLayout() {
  return <Stack />;
}
```

Where a screen lives decides how it behaves: `events/[id].tsx` inside the tab keeps the tab bar, while `bills/[billId].tsx` at the root covers the tabs. `unstable_settings` is, as the name says, marked unstable, so check the docs for your version.

## 5. Navigating and reading params

```tsx
import { Link, Stack, useLocalSearchParams, useRouter } from 'expo-router';
import { ActivityIndicator, Pressable, Text } from 'react-native';

export function EventCard({ event }: { event: { id: number; title: string } }) {
  return (
    <Link href={{ pathname: '/events/[id]', params: { id: String(event.id) } }} asChild>
      <Pressable accessibilityRole="link">
        <Text>{event.title}</Text>
      </Pressable>
    </Link>
  );
}

// app/(tabs)/events/[id].tsx
export default function EventScreen() {
  const { id } = useLocalSearchParams<{ id: string }>(); // always a string: convert and validate
  const eventId = Number(id);
  const { data: event, isPending } = useEvent(eventId); // TanStack Query hook keyed by id
  const router = useRouter();

  if (isPending) return <ActivityIndicator />;
  return (
    <>
      <Stack.Screen options={{ title: event?.title ?? 'Event' }} />
      <Pressable onPress={() => router.push('/new-task')}>
        <Text>Add a task for this event</Text>
      </Pressable>
    </>
  );
}
```

| Call | Effect |
|---|---|
| `router.push(href)` | Add a screen on top of the current stack |
| `router.replace(href)` | Swap the current screen (after login, after submitting a form) |
| `router.back()` | Go back one screen; check `router.canGoBack()` when the screen may be the first one |
| `router.dismissAll()` | Pop back to the first screen of the current stack (close a modal flow) |
| `router.setParams({ filter: 'unpaid' })` | Update the current route's params without navigating |

- `useLocalSearchParams` returns the params of the route this component belongs to. `useGlobalSearchParams` follows the focused URL anywhere and re-renders on every change, so prefer the local version.
- Query strings work too: `/bills?status=unpaid` gives `{ status: 'unpaid' }`.
- **Typed routes:** set `"experiments": { "typedRoutes": true }` in the app config and run the dev server; Expo Router generates types so a misspelled `href` fails type-checking.

## 6. Modals and sheets

Declare the presentation on the stack screen: `presentation: 'modal'` slides a card up on iOS; `formSheet` gives a sheet; `transparentModal` lets the previous screen show through (for custom dialogs); `fullScreenModal` covers everything. Inside the modal, `router.back()` or `router.dismiss()` closes it. On Android a modal is a regular screen with a different animation, and the back button closes it. For a quick confirmation, `Alert.alert` is simpler than a route.

## 7. Auth flows and protected routes

**Pattern A: protected groups** (Expo Router v5, SDK 53+; shown in the root layout above). When `guard` is false, those screens cannot be reached; when the session flips, Expo Router moves to the first available screen and removes the guarded screens from history, so logging out cannot be undone with the back button.

**Pattern B: redirect in a layout** (works in every version and is common in existing code):

```tsx
// app/(app)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useSession } from '../../lib/session';

export default function AppLayout() {
  const { user, isLoading } = useSession();
  if (isLoading) return null; // splash still visible
  if (!user) return <Redirect href="/login" />;
  return <Stack />;
}
```

Details that interviewers probe:
- **Restoring the session:** read the stored refresh token, get a fresh access token, then render. Show the splash (not a login flash) meanwhile.
- **Logout:** revoke the refresh token on the API, delete it from SecureStore, clear the query cache (`queryClient.clear()`), unregister the push token, set the session to null and let the guard redirect, or call `router.replace('/login')` after `router.dismissAll()`.
- **Deep link while logged out:** remember the requested path, send the user to login, then `router.replace(pendingPath)`.
- **Client guards are UX, not security.** The Rails API must authenticate and authorize every request (for example `before_action :authenticate_user!` plus a policy check), because anyone can call the API directly. See [Rails authentication and authorization](../05-ruby-on-rails/11-authentication-authorization.md).

## 8. Deep links, universal links and app links

**Custom scheme.** Set `"scheme": "myapp"` in the app config and `myapp://events/42` opens `app/(tabs)/events/[id].tsx`, because the group is not part of the URL. Schemes are easy, but any app can claim the same scheme, and links shared by email or chat are not clickable everywhere.

**Universal links (iOS) and app links (Android)** use normal `https://app.example.com/events/42` URLs: if the app is installed, it opens; otherwise the website does. Both require proving that you own the domain.

```json
{
  "expo": {
    "scheme": "myapp",
    "ios": { "associatedDomains": ["applinks:app.example.com"] },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [{ "scheme": "https", "host": "app.example.com", "pathPrefix": "/events" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

The website then serves two files over HTTPS, without redirects:

```json
{
  "applinks": {
    "details": [{ "appIDs": ["ABCDE12345.com.example.coliving"], "components": [{ "/": "/events/*" }] }]
  }
}
```

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.coliving",
      "sha256_cert_fingerprints": ["AB:CD:EF:..."]
    }
  }
]
```

The first is `https://app.example.com/.well-known/apple-app-site-association` (no file extension, served as JSON; `ABCDE12345` is the Apple Team ID). The second is `https://app.example.com/.well-known/assetlinks.json`. A Rails app can serve both from `public/.well-known/`.

Testing:

```bash
npx uri-scheme open "myapp://events/42" --ios
xcrun simctl openurl booted "https://app.example.com/events/42"
adb shell am start -W -a android.intent.action.VIEW -d "myapp://events/42" com.example.coliving
```

Classic failures: the Android fingerprint is taken from the upload key instead of the **Play App Signing** key shown in Play Console; the AASA file is behind a redirect or authentication; the Team ID or bundle ID is wrong; iOS caches association files, so a fix may need a reinstall; deep links are tested in Expo Go, which only knows `exp://`.

Treat incoming URLs as untrusted input: validate IDs, check permissions on the server, and never perform an action (such as paying) just because a link was opened.

## 9. Opening a screen from a push notification tap

Send an internal path in the notification's `data`, then navigate when the user taps it, whether the app was running or killed:

```tsx
// lib/use-notification-navigation.ts, called once from the root layout after the session is ready
import { useEffect } from 'react';
import * as Notifications from 'expo-notifications';
import { router, type Href } from 'expo-router';

function openFromNotification(notification: Notifications.Notification) {
  const url = notification.request.content.data?.url;
  // Only accept internal paths, never arbitrary URLs from a payload
  if (typeof url === 'string' && url.startsWith('/')) router.push(url as Href);
}

export function useNotificationNavigation(enabled: boolean) {
  useEffect(() => {
    if (!enabled) return;
    let cancelled = false;

    // Cold start: the tap launched the app before any listener existed
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (!cancelled && response) openFromNotification(response.notification);
    });

    // Warm start: the app was running or in the background
    const sub = Notifications.addNotificationResponseReceivedListener((response) => {
      openFromNotification(response.notification);
    });

    return () => {
      cancelled = true;
      sub.remove();
    };
  }, [enabled]);
}
```

Pass `enabled = !!user` so taps received while logged out wait until after login, and hedge on API names when you present this: recent SDKs also offer a hook and a synchronous variant for the last response. The full push setup is in [Push notifications](05-push-notifications.md).

## 10. Params vs global state

- **Pass IDs and small primitives**, not whole objects. Params end up in a URL as strings: objects get serialized, go stale when the data changes, and make deep links impossible because a link cannot carry the object.
- **Fetch details by ID.** With TanStack Query the detail screen reads `['events', id]`, and you can seed it from the list cache through `placeholderData`, so it renders instantly while revalidating.
- **Global state** (context or Zustand) is for app-wide facts: the session, the current house, user preferences. Do not use it to pass "the selected bill" between two screens; that breaks on deep links and restarts.
- **Returning a result** from a picker or modal: update shared state or the query cache, then `router.back()`, rather than inventing callback params (functions cannot be serialized into a URL).

## 11. Navigation performance

- **Native stack:** Expo Router's `Stack` uses React Navigation's native stack, so transitions run as native animations, not on the JS thread.
- **Lazy tabs:** a tab screen mounts on first visit (the default), keeping startup light.
- **`freezeOnBlur`:** an opt-in screen option (backed by react-native-screens) that stops re-rendering screens that are not focused, useful when a global store updates often.
- **Light mount work:** a screen that parses data or renders hundreds of rows while its transition plays makes the animation stutter. Render a skeleton first, paginate, and defer non-urgent work.
- **Prefetch on press:** `queryClient.prefetchQuery(eventQuery(id))` right before `router.push` makes the detail screen appear with data.
- **Refresh on focus:** use `useFocusEffect` (with `useCallback`) to refetch when a screen regains focus, instead of effects that run only on mount.
- **Avoid stacking duplicates:** pushing the same screen repeatedly from notifications or buttons builds deep stacks that waste memory; use `replace` or check the current route.

### Going deeper: unsaved changes
To stop the user from losing a half-filled form, React Navigation's `usePreventRemove(hasUnsavedChanges, ({ data }) => ...)` intercepts back gestures, the back button and `router.back()`. Show an `Alert` and, if the user chooses to discard, dispatch `data.action` through `useNavigation()`. Because Expo Router screens are React Navigation screens, the hook works there too.

## Common pitfalls
- Treating `useLocalSearchParams` values as numbers: they are strings, and `'42' === 42` is false.
- Passing whole objects as params, so the detail screen shows stale data and deep links break.
- A login screen that flashes before the session is restored, because the app renders before the stored token is checked; keep the splash visible.
- Logging out with `router.push('/login')`, which leaves protected screens in the history behind it.
- Forgetting `queryClient.clear()` on logout, so the next user on the same phone briefly sees the previous user's bills.
- Putting a detail route in the wrong folder, so the tab bar disappears (or stays) unexpectedly.
- Deep links that open the detail screen with no way back, because the stack has no initial route below it.
- Universal links that never verify because of a redirect, a wrong Team ID or the upload key fingerprint.
- Navigating to any URL found in a notification payload without validating it.
- Heavy work in a screen's first render, making every transition janky.

## How to talk about this in an interview
- Describe structure first: "Tabs for the main areas, a stack per tab for drilling into details, modals for focused tasks such as creating a task or paying, and an auth gate at the root."
- Show you think about the edges: session restore without flashing the login screen, logout that clears history and caches, deep links and notification taps on cold start.
- For the Coloc resident app, connect navigation to its features (events, tasks, split bills, rent payments) and to push notifications that open the right screen. Do not claim a specific library unless you used it.

> **Fill in:** whether the resident app uses Expo Router or React Navigation directly, and how its tabs and stacks are organized.

> **Fill in:** how notification taps open a screen in the app (payload format, cold start handling) and any bug you fixed there.

> **Fill in:** whether the app supports universal links or only a custom scheme, and who maintains the domain files.

## Interview questions

**Q1. Stack, tabs, drawer or modal: when do you use each?**
A stack for drilling down with back navigation (list, then detail), tabs for parallel top-level sections whose state should persist, a drawer for many secondary sections, and a modal for a focused, dismissible task such as creating a task or paying a bill. Real apps nest them: tabs inside a root stack, a stack inside each tab.
*Follow-up:* What does Android back do with nesting? It pops the innermost stack first, then outer navigators, then leaves the app.

**Q2. How does Expo Router map files to routes?**
Every file under `app/` is a route: `index.tsx` is the folder's default, `[id].tsx` is a dynamic segment, `[...slug].tsx` is a catch-all and `+not-found.tsx` handles unmatched URLs. `_layout.tsx` defines the navigator (Stack, Tabs, Drawer, Slot) for its folder, and `(group)` folders organize routes and layouts without changing URLs.
*Follow-up:* What do you gain over manual React Navigation config? Automatic deep linking, URLs for every screen, typed routes and a folder tree that documents the app.

**Q3. What is a layout route, and what goes in the root layout?**
A `_layout.tsx` renders a navigator around the routes in its folder. The root layout holds app-wide providers (query client, session, theme), splash screen control, the auth gate and the root stack with screen options for modals and full-screen routes.
*Follow-up:* What is `Slot`? A layout that renders the matched child without adding a navigator.

**Q4. How do you keep the tab bar visible on a detail screen?**
Nest a stack inside the tab: `app/(tabs)/events/_layout.tsx` returns a `Stack`, with `index.tsx` for the list and `[id].tsx` for the detail. Screens pushed inside that stack keep the tab bar; routes outside the tabs folder are pushed on the root stack and cover it.
*Follow-up:* How do you hide the tab bar for one nested screen? Move that route out to the root stack, which is cleaner than toggling tab bar visibility.

**Q5. How do you read route params, and what type are they?**
`useLocalSearchParams<{ id: string }>()` in the screen; values are strings, or string arrays for catch-all routes, because they come from a URL. Convert (`Number(id)`) and validate them, and handle invalid values with a not-found state.
*Follow-up:* Local vs global search params? Local belongs to this route; global follows the focused URL and re-renders on every navigation change.

**Q6. What is the difference between push, replace, back and dismissAll?**
`push` adds a screen on top; `replace` swaps the current one, as after login or after a submit; `back` pops one screen; `dismissAll` returns to the first screen of the current stack, which closes a multi-step modal flow. Choosing wrongly leaves screens in history that the user can return to.
*Follow-up:* When is `router.back()` unsafe? When the screen was opened by a deep link and nothing is below it; check `router.canGoBack()` and fall back to `replace`.

**Q7. How do you implement an auth flow with protected routes?**
Keep the splash visible while restoring the session from SecureStore, then render the root stack with `Stack.Protected guard={!!user}` around the app screens and `guard={!user}` around login (Expo Router v5+). In older code, a group layout returns `<Redirect href="/login" />` when there is no user. The API still checks every request, because client guards are only UX.
*Follow-up:* How do you return to the screen a deep link asked for? Store the pending path, then `router.replace` to it after login.

**Q8. After logout, the user presses back and sees the previous user's screen. How do you fix it?**
Logout must remove protected screens from history: with protected groups that happens when the guard flips; otherwise call `router.dismissAll()` and `router.replace('/login')` instead of `push`. Also clear the query cache and any persisted user state, revoke the refresh token on the server and unregister the push token.
*Follow-up:* Why clear the cache? Otherwise the next user on the same device can see cached data before refetches finish.

**Q9. How does deep linking work in an Expo app? Custom scheme vs universal links?**
A custom scheme (`myapp://events/42`) is set in the app config, and Expo Router maps the path to the matching route. Universal links (iOS) and app links (Android) use `https://` URLs for a domain you own, verified through `apple-app-site-association` and `assetlinks.json`: they open the app when installed and the website otherwise, and no other app can claim them.
*Follow-up:* How do you test? `npx uri-scheme open`, `xcrun simctl openurl` and `adb shell am start -d`, in a development build rather than Expo Go.

**Q10. What do the apple-app-site-association and assetlinks.json files do, and what typically goes wrong?**
They prove domain ownership: the AASA file lists Team ID plus bundle ID and the paths the app handles; `assetlinks.json` lists the package name and the SHA-256 fingerprint of the signing certificate. Typical failures: using the upload key fingerprint instead of the Play App Signing key, serving the file through a redirect or with the wrong content type, and a wrong Team ID.
*Follow-up:* Where do they live? Under `/.well-known/` on the domain, over HTTPS; a Rails app can serve them from `public/.well-known/`.

**Q11. How do you open a specific screen when a push notification is tapped, including from a killed state?**
Include an internal path in the notification data (`{ url: '/bills/12' }`). Add a response listener with `addNotificationResponseReceivedListener` for taps while the app runs, and read the last notification response on startup for cold starts. Validate the path and wait until the session is ready.
*Follow-up:* Why validate? The payload is input; you do not want to navigate to arbitrary URLs or screens the user may not access.

**Q12. Should you pass a whole object as a navigation param?**
No. Params are serialized into the URL as strings, objects go stale, and a deep link cannot carry them. Pass the ID and fetch by ID; with TanStack Query the data is usually already cached, and you can seed the detail from the list with `placeholderData`.
*Follow-up:* When is global state right? App-wide facts like the session or current house, not the selected item.

**Q13. How do typed routes help?**
With `experiments.typedRoutes` enabled, Expo Router generates types for every route, so `href` values in `Link` and `router.push` are checked at compile time: a renamed file or a misspelled path fails type-checking instead of reaching `+not-found` in production.
*Follow-up:* Do they validate param values? No, only the shape of the path; you still convert and validate params at runtime.

**Q14. React Navigation or Expo Router?**
Expo Router is built on React Navigation and adds file-based routing, automatic deep links, URLs for every screen and typed routes, which suits new Expo apps. React Navigation directly gives full manual control and is common in older or non-Expo apps. The concepts (navigators, options, focus events) are shared, so the knowledge transfers.
*Follow-up:* Can you use React Navigation APIs inside Expo Router? Yes, hooks such as `useNavigation`, `useFocusEffect` and `usePreventRemove` work.

**Q15. How do you make navigation feel fast?**
Use the native stack, keep tabs lazy, keep first renders light (skeletons, pagination, deferred work), prefetch the next screen's data on press, avoid re-rendering unfocused screens (`freezeOnBlur`, store selectors), and avoid pushing duplicate screens.
*Follow-up:* How do you refresh a list when returning to it? `useFocusEffect` to refetch or invalidate, or rely on TanStack Query's refetch on mount with a sensible `staleTime`.

**Q16. A deep link opens the detail screen, but there is no back button. Why, and how do you fix it?**
The deep link built a stack containing only the detail screen. Give the nested stack an initial route (in Expo Router, `unstable_settings = { initialRouteName: 'index' }` in that folder's layout) so the list sits underneath, or render your own back action that calls `router.replace('/events')` when `router.canGoBack()` is false.
*Follow-up:* Does the same apply to notification taps? Yes, they are deep links into the app.

**Q17. How do you prevent losing unsaved form data when the user navigates away?**
Track a dirty flag (React Hook Form's `formState.isDirty`) and use `usePreventRemove` to intercept back actions with a confirmation dialog; dispatch the original action if the user discards. For long forms, also persist a draft so an app restart does not lose it.
*Follow-up:* Does it catch the Android back button? Yes, it intercepts every attempt to remove the screen, including the back gesture and button.

## Further reading
- [Expo Router introduction](https://docs.expo.dev/router/introduction/)
- [Linking and deep links in Expo](https://docs.expo.dev/linking/overview/)
- [React Navigation docs](https://reactnavigation.org/docs/getting-started)
