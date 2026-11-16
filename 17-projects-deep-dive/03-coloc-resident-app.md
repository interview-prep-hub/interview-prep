# Coloc Resident App: React Native, Expo and EAS

> **Tier:** T1 (know cold) · **Related:** [React Native fundamentals](../08-react-native-expo/01-react-native-fundamentals.md), [Expo fundamentals](../08-react-native-expo/02-expo-fundamentals.md), [Navigation and Expo Router](../08-react-native-expo/03-navigation-expo-router.md), [State, storage, auth and offline](../08-react-native-expo/04-state-storage-auth-offline.md), [Push notifications](../08-react-native-expo/05-push-notifications.md), [Payments and split bills](../08-react-native-expo/06-payments-and-split-bills.md), [EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md), [Payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md), [Colochousing web and API](02-colochousing-web-and-api.md)
> **On your resume:** built features across the Coloc resident app (React Native, Expo, TypeScript), live on the App Store and Google Play for 400+ residents: events, tasks, split bills, rent payments, push notifications and EAS releases.

## Quick revision

| Item | What to remember |
|---|---|
| Where | Colochousing, a Devntech client, during your Devntech role (Nov 2025 to present) |
| What | Features across the resident app of a Belgian co-living platform |
| Stack | React Native, Expo, TypeScript; the backend it calls is presumably the Colochousing Rails REST API (confirm) |
| Status | Live on the App Store and Google Play |
| Audience | 400+ residents: the app's audience, not a number you grew |
| Areas on your resume | Events, tasks, split bills, rent payments, push notifications, EAS releases |

- **"Built features across"** means you contributed features to an app the team owns; be ready to name exactly which ones.
- **Push in one breath (the typical Expo flow; confirm yours):** ask permission, get an Expo push token (`getExpoPushTokenAsync({ projectId })`), send it to the backend, store it per device, let a backend job send through Expo's push service (which forwards to APNs and FCM), check receipts later, and delete tokens that come back `DeviceNotRegistered`.
- **EAS:** Build (cloud builds from profiles in `eas.json`), Submit (uploads to App Store Connect and Google Play), Update (over-the-air JavaScript and asset updates on a channel, gated by `runtimeVersion`).
- **OTA limits:** an update can change JavaScript and assets only. A new native module, permission or SDK upgrade needs a new store build.
- **Rent and store rules:** rent pays for a real-world service, so store rules point to a regular payment provider rather than in-app purchase; Apple and Google in-app purchase rules target digital content consumed in the app.
- **Money (best practice):** integer cents everywhere; split remainders deterministically; settle-up = net balances, then greedy matching of debtors and creditors.
- **Auth token (best practice):** `expo-secure-store` (Keychain on iOS, Keystore-backed on Android), never AsyncStorage.
- **Builds:** Expo Go cannot load custom native code, so teams use development builds (`expo-dev-client`); Expo Go also lost Android remote push in SDK 53.
- **Store releases (the usual flow):** version plus build number, TestFlight or a Play testing track, store review, phased (iOS) or staged (Android) rollout, crash monitoring after release.

> **Fill in:** the features you built yourself vs those you only touched, the Expo SDK version, and whether the app uses Expo Router or React Navigation.

## The 30-second pitch

> "I build features for Coloc, the resident app of a Belgian co-living platform. It's React Native with Expo and TypeScript, live on the App Store and Google Play for 400+ residents. I've worked across events, tasks, split bills, rent payments and push notifications, and on the app's releases with EAS."

> **Fill in:** one sentence on the feature you are proudest of and what made it tricky (for example getting bill splits to always add up, or notification taps opening the right screen).

> **Fill in:** which parts of the EAS releases you handled (cloud builds, store submissions, over-the-air updates) so you can extend the last sentence precisely.

## Domain primer

### What a resident app does
Typical responsibilities of a co-living resident app (compare with what Coloc really does):
- **Events:** community events per house or city: a list, details, RSVPs, reminders.
- **Tasks:** shared house to-dos or chores: assignment, rotation, due dates, completion.
- **Split bills:** shared expenses between housemates (groceries, supplies, outings): who paid, who owes what, running balances, settling up.
- **Rent payments:** what is due, what was paid, and a way to pay.
- **Push notifications:** reminders and updates for all of the above, plus announcements.

> **Fill in:** what "tasks" means in Coloc (chores, maintenance requests, onboarding steps) and what split bills cover if rent is all-in.

### Mobile concepts you must be able to explain

| Concept | Meaning |
|---|---|
| Store review | Apple and Google review each binary (and its metadata) before release |
| Version vs build number | The user-facing version (1.4.0) vs the internal build number that must increase on every upload |
| OTA update | A new JavaScript bundle and assets downloaded by installed apps, with no store review, if the native layer is compatible |
| Runtime version | The label that says which native layer an update is compatible with |
| Push token | An address for one app install on one device; it can change and can die |
| Deep link | A URL that opens a specific screen, used by notification taps |
| Development build | Your own debug build of the app with `expo-dev-client`, able to run custom native code |
| Secure storage | Keychain or Keystore-backed storage for secrets such as tokens |

### Payment vocabulary
Payment provider, checkout session or payment intent, webhook, idempotency key, refund, chargeback. Payment methods you may meet in Belgium include cards, Bancontact (the Belgian domestic card scheme) and SEPA Direct Debit (bank debits in the euro area, common for recurring payments); these are examples, not claims about Coloc.

> **Fill in:** the payment provider and payment methods the app actually uses.

## Typical architecture

> A typical setup for an Expo app on a Rails API, to adapt to what you actually saw. It is not a description of the real Coloc codebase.

```text
Coloc app (Expo, TypeScript)
  screens: Events | Tasks | Bills | Rent | Profile
  data:    API client + server-state cache; auth token in SecureStore
  push:    expo-notifications (permission, token, tap -> deep link)
        |  HTTPS + bearer token                          ^ notification
        v                                                |
Colochousing Rails API (/api/v1) -- Sidekiq jobs --> Expo push service --> APNs / FCM
        |                         \--> payment provider API
        |                               <-- webhook: payment succeeded
        v
    PostgreSQL

EAS Build (binaries) -> EAS Submit (stores); EAS Update (OTA bundles per channel)
```

- The server owns the truth: balances, payment status and task state are computed and stored by the API; the app displays them and caches them.
- Native capabilities (notifications, secure storage, browser sessions) come from Expo modules, which is why a development build matters for anything Expo Go cannot load.

> **Fill in:** the data-fetching and state approach (TanStack Query, Redux Toolkit, Zustand, plain context), the navigation library, and the crash-reporting tool.

## Bullet-by-bullet

The bullet is long; interviewers pick it apart feature by feature.

| Part of the bullet | What they ask | Study |
|---|---|---|
| Events | RSVP flow, list performance, reminders | [RN fundamentals](../08-react-native-expo/01-react-native-fundamentals.md), [performance](../08-react-native-expo/07-performance-and-animations.md) |
| Tasks | Assignment, rotation, optimistic updates, offline | [state and offline](../08-react-native-expo/04-state-storage-auth-offline.md) |
| Split bills | Rounding, balances, settle-up, where calculations live | [payments and split bills](../08-react-native-expo/06-payments-and-split-bills.md) |
| Rent payments | Provider, flow, webhooks, why not in-app purchase | [payments and subscriptions](../13-architecture-and-practices/08-payments-and-subscriptions.md) |
| Push notifications | Token lifecycle, sending, tap handling, permissions | [push notifications](../08-react-native-expo/05-push-notifications.md) |
| EAS releases | Profiles, submit, OTA updates, rollback, versioning | [EAS](../08-react-native-expo/08-eas-build-submit-update.md) |
| "Live on the App Store and Google Play for 400+ residents" | Store review, release process, crashes in production | Part D below |

How to answer honestly:
- Lead with the features you owned end to end; then list areas you contributed to.
- For each feature know: the screens, the API endpoints it called, one edge case, one bug you fixed.
- Say "400+ residents use the app" as a fact about the product, not a growth claim.

> **Fill in:** for each of the six areas, mark "built", "contributed" or "not mine", with one concrete detail for each "built".

## Design deep-dive

> Typical designs to adapt. Where the real app works differently, describe the real one and explain why it was built that way.

### Part A: push notifications end to end

```ts
// Client: get a token and hand it to the API (pattern from the Expo push setup guide).
import * as Device from "expo-device";
import * as Notifications from "expo-notifications";
import Constants from "expo-constants";
import { Platform } from "react-native";

export async function registerForPushAsync(): Promise<string | null> {
  if (!Device.isDevice) return null;                 // Expo push tokens need a physical device

  if (Platform.OS === "android") {
    await Notifications.setNotificationChannelAsync("default", {
      name: "Default",
      importance: Notifications.AndroidImportance.DEFAULT,
    });
  }

  const { status: existing } = await Notifications.getPermissionsAsync();
  let finalStatus = existing;
  if (existing !== "granted") {
    finalStatus = (await Notifications.requestPermissionsAsync()).status;
  }
  if (finalStatus !== "granted") return null;        // respect the choice; explain the value later

  const projectId = Constants.expoConfig?.extra?.eas?.projectId ?? Constants.easConfig?.projectId;
  const { data } = await Notifications.getExpoPushTokenAsync({ projectId });
  return data;                                       // "ExponentPushToken[...]": POST it to the API
}
```

```ruby
# Server (illustrative, assuming a Rails backend with Sidekiq): a job sends through Expo's push API.
# Expo's push service needs the app's FCM and APNs credentials, which are configured in EAS.
require "net/http"

class SendPushJob
  include Sidekiq::Job
  sidekiq_options queue: :push, retry: 5

  EXPO_PUSH_URL = URI("https://exp.host/--/api/v2/push/send")

  def perform(user_id, title, body, data = {})
    tokens = PushToken.where(user_id: user_id, disabled_at: nil).pluck(:token)
    return if tokens.empty?

    messages = tokens.map { |token| { to: token, title: title, body: body, data: data } }
    response = Net::HTTP.post(EXPO_PUSH_URL, messages.to_json, "Content-Type" => "application/json")
    raise "Expo push failed with #{response.code}" unless response.is_a?(Net::HTTPSuccess)
    # The response holds one ticket per message. Save the ticket IDs and check their receipts
    # in a later job; disable any token whose ticket or receipt reports DeviceNotRegistered.
  end
end
```

The token lifecycle, which is where real bugs hide:
1. **Register on every launch after login,** because tokens can change (reinstall, restore onto a new phone). The API upserts by token (unique index) and updates `user_id` and `last_seen_at`, which also handles a shared device changing hands.
2. **Unregister on logout** (`DELETE /api/v1/push_token`), or the next person on that phone receives the previous user's notifications.
3. **Disable dead tokens** when Expo reports `DeviceNotRegistered` (usually the app was uninstalled), and prune tokens not seen for a long time.
4. **Respect permissions:** if the user denies or later revokes permission, delivery silently stops; show the value of notifications in context and link to settings instead of nagging.
5. **Preferences live on the server** (events, bills, rent), so jobs filter recipients before sending.
6. **Keep environments apart:** development builds and production builds should not share tokens or credentials.

Taps and deep links: attach a target in `data` (for example `{ url: "/bills/42" }`), handle `Notifications.addNotificationResponseReceivedListener` while the app runs, and `Notifications.getLastNotificationResponseAsync()` for a cold start, then navigate with the router you use. Decide what the app shows in the foreground with `Notifications.setNotificationHandler` (its options changed across SDK versions, so check the docs for yours).

> **Fill in:** which push service the app really uses (Expo push service, FCM directly, a third-party service), what triggers notifications, and how tokens are cleaned up.

### Part B: split bills that always add up

```ts
// Amounts are integer cents. The first `remainder` members pay one extra cent,
// so shares always sum to the total. Use a stable member order (for example by id).
export function splitEvenly(totalCents: number, memberIds: string[]): Record<string, number> {
  if (!Number.isInteger(totalCents) || totalCents < 0 || memberIds.length === 0) {
    throw new Error("invalid split");
  }
  const base = Math.floor(totalCents / memberIds.length);
  const remainder = totalCents % memberIds.length;
  return Object.fromEntries(memberIds.map((id, i) => [id, base + (i < remainder ? 1 : 0)]));
}

// splitEvenly(1000, ["a", "b", "c"]) -> { a: 334, b: 333, c: 333 }
```

- **Never floats for money:** `0.1 + 0.2 !== 0.3` in JavaScript. Store and compute cents (or minor units) as integers, format only for display.
- **Uneven splits** (by shares or percentages) use the same idea: compute exact shares, floor them, then hand out the leftover cents by largest fractional part (the largest remainder method).
- **Balances:** for each member, total paid minus total owed. Balances across a house sum to zero; assert it in tests.
- **Settle-up suggestions:** split members into debtors and creditors, then repeatedly match the largest debtor with the largest creditor for the smaller of the two amounts. It needs at most n - 1 transfers. Finding the true minimum number of transfers is NP-hard in general, so greedy is the practical choice.
- **Where it runs:** the server computes and stores balances so every device agrees; the app may compute a preview while a user edits a bill, and the server's numbers win.

> **Fill in:** how Coloc splits (equal only, shares, custom amounts), where balances are computed, and whether settle-up suggestions exist.

### Part C: rent payments without in-app purchase
Apple's App Store Review Guidelines (the rules on goods and services consumed outside the app) require apps that sell physical goods or real-world services to use payment methods other than in-app purchase, and Google Play's payments policy makes a similar distinction. Rent pays for housing, so it goes through a normal payment provider, without the store's commission.

A typical safe flow:
1. The app asks the API to pay a specific rent invoice. The **server** decides the amount (never trust an amount from the client) and creates a payment with the provider, using the invoice ID as an idempotency key so a double tap does not create two payments.
2. The app opens the provider's payment UI: a hosted checkout page in an in-app browser session (`expo-web-browser`) or the provider's native SDK (which needs a development build).
3. The provider calls the API's **webhook**. The API verifies the signature, records the event ID (to ignore duplicates), marks the invoice paid and enqueues a "payment received" notification.
4. The app refreshes the invoice from the API. The browser redirect or SDK callback only means "the user finished the flow", not "the money arrived".

> **Fill in:** how rent payments work in Coloc: in-app checkout, a payment link, direct debit, or showing the status of bank transfers, and which provider handles it.

### Part D: releasing with EAS

```json
{
  "cli": { "appVersionSource": "remote" },
  "build": {
    "development": { "developmentClient": true, "distribution": "internal" },
    "preview": { "distribution": "internal", "channel": "preview" },
    "production": { "channel": "production", "autoIncrement": true }
  },
  "submit": { "production": {} }
}
```

```bash
eas build --profile production --platform all      # cloud builds for iOS and Android
eas submit --platform ios --latest                 # upload the latest iOS build to App Store Connect
eas submit --platform android --latest             # upload the latest Android build to Google Play
eas update --branch production --message "Fix bill rounding"   # OTA update for compatible builds
```

- **Channels and branches:** each build is tied to a channel (from its profile); a channel points at a branch; you publish updates to a branch. That is how a preview build gets preview updates and production gets production updates.
- **Runtime version:** an update only reaches builds with the same `runtimeVersion`. Policies such as `appVersion` tie it to the app version; recent SDKs also offer a `fingerprint` policy computed from the native project. A native change means a new runtime version and a new store build.
- **A store release:** bump the version (build numbers can auto-increment), build, test on TestFlight or a Play testing track, update store listings and privacy details (Apple's privacy labels, Google Play's Data safety form), submit for review, then release gradually: phased release on the App Store, staged rollout on Google Play. Watch crash and error rates before going to 100%.
- **Rolling back a bad OTA update:** republish the last good update to the branch, or point the channel back to a known-good branch; EAS also supports rolling back to the update embedded in the build. The exact CLI commands have changed over time, so check the EAS Update docs for your version. Prevent it next time with a preview channel and a gradual rollout.
- **Common rejection reasons:** crashes on the reviewer's device, a login the reviewer cannot get past (provide a demo account), missing in-app account deletion when the app lets users create accounts (an Apple requirement), vague permission prompts, and payment flows that break the rules above.

> **Fill in:** your real release flow: who triggers builds, how often you ship store releases vs OTA updates, how versions are bumped, and one release that taught you something.

### Part E: tasks, rotation and optimistic updates
Two questions come up for a tasks feature: "how are chores assigned?" and "why does ticking a task feel instant?"

- **Rotation:** order the house members by a stable key and assign `members[(period + taskOffset) % members.length]` for each period (week, month). Store the generated assignments instead of recomputing them on the fly, so history stays correct when someone moves in or out, and regenerate only future periods.
- **Optimistic updates:** update the cache immediately, send the request, roll back if it fails, then refetch. With TanStack Query v5 (a common choice; check what the app uses) it looks like this:

```ts
import { useMutation, useQueryClient } from "@tanstack/react-query";

type Task = { id: string; title: string; done: boolean };

export function useToggleTask(houseId: string) {
  const queryClient = useQueryClient();
  const queryKey = ["tasks", houseId];

  return useMutation({
    mutationFn: (task: Task) => api.patch(`/tasks/${task.id}`, { done: !task.done }), // api: the app's HTTP client
    onMutate: async (task) => {
      await queryClient.cancelQueries({ queryKey });               // an in-flight refetch must not overwrite us
      const previous = queryClient.getQueryData<Task[]>(queryKey);
      queryClient.setQueryData<Task[]>(queryKey, (old) =>
        old?.map((t) => (t.id === task.id ? { ...t, done: !t.done } : t)),
      );
      return { previous };
    },
    onError: (_error, _task, context) => {
      if (context?.previous) queryClient.setQueryData(queryKey, context.previous); // roll back
    },
    onSettled: () => queryClient.invalidateQueries({ queryKey }),  // resync with the server's truth
  });
}
```

- On the server, make the endpoint idempotent by sending the desired state (`done: true`) rather than "toggle", so a retried request cannot flip the task back.

> **Fill in:** how tasks are assigned in Coloc and whether any screen uses optimistic updates.

### Going deeper
- **Crash monitoring:** tools such as Sentry or Firebase Crashlytics; upload source maps so stack traces point at your TypeScript, and tag events with the app version and update ID so a spike can be tied to a specific OTA update.
- **Offline and flaky networks:** cache server state for instant screens, use optimistic updates for small actions (marking a task done) with rollback on failure, and make retried requests idempotent on the server.
- **Performance:** long event or bill lists need virtualized lists (`FlatList` or a faster list library) with stable keys and memoized rows.

> **Fill in:** the crash-monitoring tool, if any, and anything you did for offline behavior or list performance.

## Likely interview questions

**Q1. Which features did you build in the Coloc app?**
Name two or three features you owned, each with: the screens, the endpoints, one edge case and one bug or decision. Then list the areas you contributed to. Never claim the whole app.
> **Fill in:** your features with those four details each.

**Q2. How do push notifications work in your app?**
Walk the chain: permission, push token, token sent to the backend and stored per device, a backend event enqueues a job (a Sidekiq job if the backend is the Rails app), the job calls the push service, the push service delivers through APNs or FCM, the tap opens a deep link, and receipts clean up dead tokens. Mention logout unregistering and per-category preferences.
> **Fill in:** the real push service, triggers and cleanup.

**Q3. How do you release with EAS?**
Profiles in `eas.json` (development, preview, production), `eas build` for binaries, `eas submit` for the stores, `eas update` for OTA fixes, `runtimeVersion` for compatibility, channels and branches to separate preview from production, then store review and a gradual rollout.
> **Fill in:** your actual flow and cadence.

**Q4. Which payment provider did you use, and why doesn't rent go through Apple's in-app purchase?**
Name the real provider and why it suits (European payment methods, recurring payments, hosted checkout). Then the rule: store purchase systems are for digital goods used in the app; rent is a real-world service, so the guidelines require another payment method.
> **Fill in:** the provider and the reason it was chosen.

**Q5. A bill of 10.00 is shared by three people. Who pays what?**
Work in cents: 1000 / 3 = 333 remainder 1, so one person pays 334 and the others 333, chosen by a stable rule. For settle-up, compute each member's net balance and greedily match the largest debtor with the largest creditor.
*Follow-up:* Why not floats? Binary floating point cannot represent most decimal fractions exactly, so totals drift.

**Q6. How do you manage push tokens over time?**
Register on every launch, upsert by token, unregister on logout, disable on `DeviceNotRegistered`, prune stale tokens, and keep development and production apart.

**Q7. A bad OTA update reached users. What do you do?**
Republish the last good update (or roll back to the embedded one) so apps pick up the fix on their next update check, confirm with crash monitoring, then find out why the preview channel or rollout did not catch it. If the bug is native, an OTA cannot fix it: ship a store build, using an expedited review request if it is severe.

**Q8. Where do you store the auth token, and why?**
`expo-secure-store`, backed by the iOS Keychain and Android Keystore, because AsyncStorage is unencrypted. Clear it on logout, handle expiry with refresh, and never put tokens in logs or crash reports.
> **Fill in:** the real token storage and refresh behavior.

**Q9. Walk me through a store release. What gets apps rejected?**
Version and build bump, build, TestFlight or a Play testing track, metadata and privacy forms, review, phased or staged rollout, monitoring. Rejections: crashes, reviewers blocked at login, missing account deletion, unclear permission prompts, payment rule violations.

**Q10. How do you find out about crashes in production?**
A crash and error reporting tool with source maps and release tags, plus store dashboards (App Store Connect, Google Play Console vitals). Then reproduce on a development build.
> **Fill in:** the tool the team uses, or say honestly that you relied on store dashboards and user reports.

**Q11. Expo Go vs a development build vs a production build?**
Expo Go is a sandbox app with a fixed set of native modules, for quick experiments. A development build is your app with `expo-dev-client`, any native code you need and fast refresh. A production build is the optimized, signed binary you ship. Push notifications on Android and custom native modules need a development build.

**Q12. How does tapping a notification open the right screen?**
Put a route in the notification's `data`, listen for notification responses while the app runs, check the last response on a cold start, and navigate. Validate the route before navigating, because data can be stale (for example, a deleted bill).

**Q13. What can an OTA update change, and what can it not?**
It can change JavaScript, styles, images and other assets. It cannot add native modules, change permissions, upgrade the Expo SDK or React Native, or change app icons and native config. Those need a new build and a new runtime version.

**Q14. How do you keep the app and API compatible as both change?**
Additive API changes, versioned endpoints, a minimum-supported-version check with a forced-update screen, and feature flags so the server can hide a feature an old app cannot handle. See [the API side](02-colochousing-web-and-api.md).

**Q15. How would you build the events feature from scratch?**
Endpoints for listing (paginated, filtered by house) and RSVPs (idempotent PUT per user), a cached list screen with pull to refresh, optimistic RSVP toggling, a reminder job on the server some time before the event, and a deep link from the reminder to the event.

**Q16. What was the hardest problem you solved in the app?**
Use a STAR story from [the story bank](../18-behavioral-and-hr/02-star-stories-bank.md): symptom, investigation, root cause, fix, prevention.
> **Fill in:** a real story from the app.

## Fill-in checklist
- [ ] Which of events, tasks, split bills, rent payments, push and releases you built, contributed to, or did not touch.
- [ ] Expo SDK version, navigation library, data-fetching and state approach.
- [ ] Push service, notification triggers, token cleanup, tap handling.
- [ ] How bills are split, where balances are computed, whether settle-up exists.
- [ ] Payment provider, payment methods and the rent payment flow.
- [ ] Auth token storage and refresh behavior.
- [ ] Your release flow: profiles, who builds, how often, OTA policy, how rollbacks would happen.
- [ ] Crash and error monitoring tool.
- [ ] One bug story and one release story.
- [ ] Where the 400+ residents figure comes from.
