# Native Modules, Permissions and Testing

> **Tier:** T1 (know cold) · **Related:** [Expo fundamentals](02-expo-fundamentals.md), [EAS Build, Submit and Update](08-eas-build-submit-update.md), [React testing](../06-react/10-testing.md), [Testing fundamentals](../13-architecture-and-practices/09-testing-fundamentals.md), [Observability and debugging](../14-devops-and-tools/05-observability-and-debugging.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md)
> **On your resume:** the Coloc resident app (React Native, Expo, TypeScript), live on both stores for 400+ residents: the device features, permissions, monitoring and testing practices any store-released app needs.

## Quick revision
- When the Expo SDK is not enough: a maintained community library with a config plugin, then your own config plugin, then a local module written with the **Expo Modules API** (Swift and Kotlin DSL), all used through a development build.
- **iOS permissions:** declare a usage string in Info.plist for each protected resource (missing string = crash when requesting); the system prompt appears **once**, afterwards only Settings can change it.
- **Android permissions:** declare in AndroidManifest; "dangerous" ones are requested at runtime (Android 6+); after two denials (Android 11+) the prompt stops appearing; notifications need a runtime permission from Android 13.
- Flow: explain the value in your UI, request at the moment of need, handle granted, denied and "denied forever" (`canAskAgain === false`, then `Linking.openSettings()`), re-check when the app returns to the foreground, degrade gracefully.
- Expo modules expose permission hooks and methods (`useCameraPermissions`, `requestForegroundPermissionsAsync`); config plugins set usage strings (`["expo-camera", { "cameraPermission": "..." }]`).
- Location: foreground first; background location is a separate request, needs strong justification, store declarations (Google Play) and a task defined with `expo-task-manager`.
- Background work is opportunistic: the OS decides when tasks run (expo-background-task in recent SDKs, minimum intervals around 15 minutes, nothing guaranteed after a force-quit on iOS).
- App lifecycle: `active`, `background`, `inactive` (iOS); the OS can kill a backgrounded app without warning; handle cold starts.
- Crash reporting: `@sentry/react-native` (with its Expo config plugin), source maps uploaded for every build and every EAS Update, releases and environments, release health (crash-free sessions and users).
- Debug a production crash: scope it (platform, app version, update ID, device), read the symbolicated stack and breadcrumbs, classify (JS, native, ANR, out of memory), reproduce in a release build, mitigate (OTA rollback or fix, feature flag), fix the root cause, add a test.
- Testing: Jest with the `jest-expo` preset, React Native Testing Library (`render`, `screen`, `fireEvent`, `waitFor`, queries by role and label), mocks for native modules, end-to-end flows with **Maestro** (YAML) or **Detox** (gray-box, JS).

## 1. When the Expo SDK is not enough

| Need | Option | Example |
|---|---|---|
| A capability Expo covers | Expo SDK module | `expo-camera`, `expo-location`, `expo-notifications` |
| A capability a library covers | Community library, ideally with a config plugin and New Architecture support | Payments SDK, analytics, maps, MMKV |
| A library needs extra native setup | Your own config plugin | Adding a manifest entry or Info.plist key the library does not automate |
| Nothing exists | Local module with the **Expo Modules API** | A vendor's door-lock SDK, a home-screen widget bridge |
| A library has a bug | `patch-package`, plus an upstream fix | A crash fix not yet released |

All of these require a **development build**; Expo Go cannot load your native code.

## 2. The Expo Modules API

Scaffold a module inside the app with `npx create-expo-module@latest --local` (it lands in `modules/`). You describe the module with a small DSL; Expo generates the JSI bindings, so the same definition style works on iOS and Android.

```text
// Swift (modules/battery-saver/ios/BatterySaverModule.swift), illustrative
import ExpoModulesCore

public class BatterySaverModule: Module {
  public func definition() -> ModuleDefinition {
    Name("BatterySaver")

    Function("isLowPowerModeEnabled") { () -> Bool in
      ProcessInfo.processInfo.isLowPowerModeEnabled
    }

    AsyncFunction("fetchDiagnostics") { (verbose: Bool) -> [String: Any] in
      ["lowPower": ProcessInfo.processInfo.isLowPowerModeEnabled, "verbose": verbose]
    }

    Events("onPowerModeChange")
  }
}
```

The Kotlin side mirrors it inside `override fun definition() = ModuleDefinition { ... }` with the same `Name`, `Function`, `AsyncFunction` and `Events` blocks. TypeScript then loads it by name:

```ts
// modules/battery-saver/index.ts
import { requireNativeModule } from 'expo-modules-core';

type BatterySaverModule = {
  isLowPowerModeEnabled(): boolean; // synchronous via JSI
  fetchDiagnostics(verbose: boolean): Promise<Record<string, unknown>>;
};

export default requireNativeModule<BatterySaverModule>('BatterySaver');
```

The DSL also supports `View` (native UI components with `Prop` setters), constants and lifecycle hooks. Keep native modules thin: expose small, typed primitives and keep business logic in TypeScript, where the rest of the team can maintain it.

### Going deeper: TurboModules with Codegen
Without Expo's DSL, a New Architecture module starts from a typed spec that Codegen reads at build time:

```ts
// specs/NativeBatteryLevel.ts: the file name must start with "Native"
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getBatteryLevel(): Promise<number>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('BatteryLevel');
```

Codegen generates the C++ and platform interfaces; you implement them in Objective-C++ or Swift and Kotlin or Java. The Expo Modules API is usually less boilerplate for app teams.

## 3. The permission model: iOS vs Android

| | iOS | Android |
|---|---|---|
| Declaration | Usage description strings in Info.plist (`NSCameraUsageDescription`, `NSLocationWhenInUseUsageDescription`...); a missing string crashes the app when it requests access | `<uses-permission>` entries in AndroidManifest; Expo modules add theirs; `android.permissions` and `android.blockedPermissions` in the app config adjust the list |
| Runtime prompt | Once per permission; afterwards only the Settings app | "Dangerous" permissions at runtime (Android 6+); two denials (Android 11+) stop the prompt |
| Granularity | Limited photo library access (iOS 14+), precise or approximate location, While Using or Always | Approximate or precise location, foreground and background as separate grants, selected-photos access (Android 14+) |
| Notifications | Prompt required; provisional authorization possible | Runtime permission from Android 13 (API 33) |
| Rationale | Write your own explanation screen before the system prompt | The platform signals when a rationale is recommended; Expo surfaces `canAskAgain` |

Set iOS strings through config plugins (or `ios.infoPlist`) so CNG builds always include them, and make them specific: Apple rejects vague ones.

```json
{
  "expo": {
    "plugins": [
      ["expo-camera", { "cameraPermission": "Allow $(PRODUCT_NAME) to use the camera to scan receipts for shared bills." }],
      ["expo-image-picker", { "photosPermission": "Choose a photo for your profile or attach a receipt to a bill." }],
      ["expo-location", { "locationWhenInUsePermission": "Show houses and events near you." }]
    ],
    "android": { "blockedPermissions": ["android.permission.RECORD_AUDIO"] }
  }
}
```

`blockedPermissions` removes permissions that a library adds but you do not use, which keeps store forms and reviews simpler.

## 4. A permission flow that handles every state

```tsx
import { CameraView, useCameraPermissions } from 'expo-camera';
import { Linking, Pressable, Text, View } from 'react-native';

export function ReceiptScanner({ onScanned }: { onScanned: (data: string) => void }) {
  const [permission, requestPermission] = useCameraPermissions();

  if (!permission) return null; // status still loading

  if (!permission.granted) {
    const canAsk = permission.canAskAgain; // false after "Don't allow" on iOS or repeated denials on Android
    return (
      <View style={{ flex: 1, justifyContent: 'center', padding: 24, gap: 12 }}>
        <Text>Camera access lets you scan a receipt instead of typing the amount.</Text>
        <Pressable accessibilityRole="button" onPress={canAsk ? requestPermission : () => Linking.openSettings()}>
          <Text>{canAsk ? 'Allow camera' : 'Open Settings'}</Text>
        </Pressable>
        {/* Always offer a path that works without the permission */}
      </View>
    );
  }

  return (
    <CameraView
      style={{ flex: 1 }}
      facing="back"
      barcodeScannerSettings={{ barcodeTypes: ['qr'] }}
      onBarcodeScanned={({ data }) => onScanned(data)}
    />
  );
}
```

Rules: request at the moment of need (not at launch), explain the benefit first, never dead-end the user (manual entry as a fallback), and re-check the permission when the app returns from Settings (the hook refreshes; with plain methods, listen to `AppState`).

## 5. Camera, image picker, location, files and sharing

**Image picker and upload to Rails:**

```ts
import * as ImagePicker from 'expo-image-picker';

export async function pickAndUploadReceipt(billId: number, accessToken: string) {
  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ['images'], // array form in recent SDKs; older code used MediaTypeOptions.Images
    quality: 0.7, // compress before upload
  });
  if (result.canceled) return;

  const asset = result.assets[0];
  const form = new FormData();
  // React Native's FormData accepts a { uri, name, type } file descriptor
  form.append('receipt', { uri: asset.uri, name: asset.fileName ?? 'receipt.jpg', type: asset.mimeType ?? 'image/jpeg' } as unknown as Blob);

  const res = await fetch(`${process.env.EXPO_PUBLIC_API_URL}/api/v1/bills/${billId}/receipt`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${accessToken}` }, // no Content-Type: fetch sets the multipart boundary
    body: form,
  });
  if (!res.ok) throw new Error(`Upload failed: ${res.status}`);
}
```

The system photo picker often needs no library permission at all (it returns only what the user chose), and Google Play restricts apps that request broad photo and video access, so prefer the picker. On the Rails side, Active Storage attaches the upload.

**Location:** `Location.requestForegroundPermissionsAsync()` then `Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Balanced })`; `watchPositionAsync` for live updates while a screen is open. Background location needs `requestBackgroundPermissionsAsync()` after foreground access (on Android 11+ the user must pick "Allow all the time" in Settings), a task, and on Google Play a declaration with a justification video. Most apps, including a resident app, should avoid it.

```ts
import * as TaskManager from 'expo-task-manager';
import * as Location from 'expo-location';

const LOCATION_TASK = 'background-location';

// Module scope, not inside a component: the OS can start your JS with no screen mounted
TaskManager.defineTask(LOCATION_TASK, async ({ data, error }) => {
  if (error) return;
  const { locations } = data as { locations: Location.LocationObject[] };
  // batch and send to the API
});

export async function startBackgroundLocation(): Promise<boolean> {
  if ((await Location.requestForegroundPermissionsAsync()).status !== 'granted') return false;
  if ((await Location.requestBackgroundPermissionsAsync()).status !== 'granted') return false;
  await Location.startLocationUpdatesAsync(LOCATION_TASK, { accuracy: Location.Accuracy.Balanced, distanceInterval: 100 });
  return true;
}
```

**Files and sharing:** `expo-file-system` reads, writes and downloads files (SDK 54 made an object-based API with `File`, `Directory` and `Paths` the default and moved the old functions to `expo-file-system/legacy`; check the docs for your SDK). `expo-sharing` opens the native share sheet: `await Sharing.shareAsync(localUri, { mimeType: 'application/pdf' })` after checking `Sharing.isAvailableAsync()`, for example to share a rent receipt PDF downloaded from the API.

## 6. Background tasks

- Mobile operating systems aggressively limit background work to save battery. Your JS does not keep running after the app is backgrounded.
- `expo-task-manager` registers named tasks (defined at module scope) used by location, notifications and background task modules.
- Recent SDKs provide `expo-background-task` (using WorkManager on Android and BGTaskScheduler on iOS), which replaced the deprecated `expo-background-fetch` (check the docs for the exact API). The OS decides when to run tasks based on battery, network and usage; minimum intervals are around 15 minutes, and iOS will not run them after the user force-quits the app.
- Never rely on background tasks for anything time-critical. Rent reminders belong on the server (a scheduled Sidekiq job plus push); background tasks are for opportunistic syncs such as uploading a queued photo.

## 7. App lifecycle

| State | Meaning | Typical reaction |
|---|---|---|
| `active` | Foreground, receiving input | Refetch stale data, re-check permissions, unlock if needed |
| `inactive` (iOS) | Transitioning: app switcher, incoming call, Control Center | Pause animations or media; do not treat it as backgrounded |
| `background` | Not visible; may be suspended or killed at any time | Save drafts, stop polling, record the time for auto-lock |

A **cold start** launches a killed app from scratch (restore session, read the last notification response, handle the initial deep link); a **warm start** resumes it from memory. Test both, because many bugs (deep links, notification taps, stale sessions) appear only on cold starts.

## 8. Crash reporting and monitoring with Sentry

```ts
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';
import * as Updates from 'expo-updates';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN, // a DSN only allows sending events, so it can be public
  environment: process.env.EXPO_PUBLIC_APP_ENV ?? 'development',
  enabled: !__DEV__,
  tracesSampleRate: 0.2, // performance sampling; tune to your quota
});
Sentry.setTag('expo-update-id', Updates.updateId ?? 'embedded'); // which OTA update was running
Sentry.setTag('runtime-version', Updates.runtimeVersion ?? 'unknown');

function RootLayout() {
  /* providers and navigator */
  return null;
}

export default Sentry.wrap(RootLayout);
```

- **Setup:** install with `npx expo install @sentry/react-native`, add its Expo config plugin (`@sentry/react-native/expo`) and its Metro config helper, and give EAS Build a `SENTRY_AUTH_TOKEN` secret so source maps and native debug files upload during builds. The older `sentry-expo` package is deprecated. Plugin options change between versions, so follow the current Sentry docs.
- **Source maps:** production JS is minified, so stack traces are unreadable without source maps. They must be uploaded for **every build and every EAS Update** (Sentry provides a command for uploading update source maps after `eas update`).
- **Native crashes** need iOS dSYM files and Android mapping files (when R8 is on) for symbolication; the plugin handles uploads during EAS Build.
- **Release health:** crash-free sessions and users per release, adoption per version, regressions after a release. Alert on spikes.
- **Breadcrumbs and context:** navigation, network calls and logs leading to the error; attach the user ID only (no emails or tokens) and scrub sensitive fields.
- **Platform consoles** add more: Play Console Android vitals (crash and ANR rates, which affect store visibility) and Xcode Organizer and App Store Connect crash reports.
- **Error boundaries:** catch render errors per screen (Expo Router lets a route export an `ErrorBoundary` component) and show a retry UI instead of a white screen.

## 9. Debugging a crash in production

1. **Scope:** which platform, app version, build number, update ID, OS versions and devices? When did it start, and does it line up with a release or an API deploy?
2. **Read the evidence:** the symbolicated stack trace, breadcrumbs (last screen, last request), device state (memory, network) and how many users are affected.
3. **Classify:** JS exception (red screen in development), native crash (no JS stack, a symbolicated native frame), ANR (main thread blocked too long on Android) or out-of-memory kill (often no report at all; look for patterns such as large images).
4. **Reproduce:** the same app version and update on a similar device, in a release build, with the same account state and network conditions. Unexpected API data (a `null` field, a new enum value) is a frequent cause, so check the Rails logs for the failing request.
5. **Mitigate quickly:** roll back or fix by OTA if it is JS-only; disable the feature with a server-side flag; fix the API response; for native crashes, halt the rollout and ship a fixed build.
6. **Fix and prevent:** a regression test, safer parsing (validate API responses, for example with Zod), an alert on the error rate, and a short blameless postmortem.

## 10. Testing

| Layer | Tool | What it covers |
|---|---|---|
| Unit | Jest | Pure logic: bill splitting, date helpers, reducers, the API client's refresh logic |
| Component | React Native Testing Library | Rendering, user interactions, loading and error states |
| Integration | RNTL with mocked network | A screen with its hooks, query client and navigation |
| End to end | Maestro or Detox | Real builds on simulators or devices: login, create a bill, pay |

```json
{
  "scripts": { "test": "jest" },
  "jest": { "preset": "jest-expo" }
}
```

`jest-expo` configures Babel, platform resolution and mocks for many Expo modules. Install it with `npx expo install jest-expo jest -- --save-dev`, plus `@testing-library/react-native`.

```tsx
// features/auth/LoginForm.test.tsx
import { fireEvent, render, screen, waitFor } from '@testing-library/react-native';
import { LoginForm } from './LoginForm';

test('submits the email and password', async () => {
  const onSubmit = jest.fn().mockResolvedValue(undefined);
  render(<LoginForm onSubmit={onSubmit} />);

  fireEvent.changeText(screen.getByLabelText('Email'), 'resident@example.com');
  fireEvent.changeText(screen.getByLabelText('Password'), 'test-password');
  fireEvent.press(screen.getByRole('button', { name: 'Log in' }));

  await waitFor(() =>
    expect(onSubmit).toHaveBeenCalledWith({ email: 'resident@example.com', password: 'test-password' }),
  );
});

test('shows a validation error for an invalid email', async () => {
  render(<LoginForm onSubmit={jest.fn()} />);
  fireEvent.changeText(screen.getByLabelText('Email'), 'not-an-email');
  fireEvent.press(screen.getByRole('button', { name: 'Log in' }));
  expect(await screen.findByText('Enter a valid email')).toBeTruthy();
});
```

Query the way users perceive the screen: by role and accessible name, label or text, and `testID` only as a last resort. Prefer `findBy` and `waitFor` for async UI, and avoid testing implementation details such as state values.

**Mocking native modules:**

```ts
// jest.setup.ts (referenced from "setupFiles" in the Jest config)
jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn().mockResolvedValue(null),
  setItemAsync: jest.fn().mockResolvedValue(undefined),
  deleteItemAsync: jest.fn().mockResolvedValue(undefined),
}));

jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock'),
);
```

For components that use TanStack Query, render inside a fresh `QueryClientProvider` per test with `retry: false`, and mock the network at the API client or `fetch` level. Expo Router ships testing utilities for rendering a mocked route tree (check `expo-router/testing-library` in the docs).

**End to end:**

```yaml
# .maestro/login.yaml
appId: com.example.coliving
---
- launchApp:
    clearState: true
- tapOn: "Email"
- inputText: "resident@example.com"
- tapOn: "Password"
- inputText: "test-password"
- tapOn: "Log in"
- assertVisible: "Upcoming events"
```

Run it with `maestro test .maestro/login.yaml` against a simulator build pointed at a seeded staging API. **Maestro** uses simple YAML flows, works black-box on any build and tolerates timing well. **Detox** is gray-box: JS tests that synchronize with the app's idle state, more setup with Expo but tight control. Keep end-to-end suites small and focused on the flows that make money or break releases: login, viewing bills, paying, receiving a notification.

## 11. Common production issues

| Symptom | Common cause | Fix |
|---|---|---|
| Crash on launch right after an OTA update | Update needs native code the binary lacks | Correct runtime version policy; roll back the update |
| Crash on permission request (iOS) | Missing usage description string | Add it through the config plugin; new build |
| Works on iOS, broken layout on Android | Font metrics, elevation, edge-to-edge insets, keyboard behavior | Test both platforms; insets and keyboard handling per platform |
| Out-of-memory crashes on cheap phones | Full-size images, huge lists | Thumbnails, `expo-image`, virtualization |
| "Invalid Date" only on devices | Non-ISO date strings parsed with `new Date()` | Send ISO 8601 from Rails, parse with a date library |
| Random logouts | Parallel refresh with rotation, Keychain or backup quirks | Single-flight refresh; handle storage read errors |
| Notifications open the wrong screen | Cold-start taps not handled, stale payloads | Read the last notification response; validate paths |

## Common pitfalls
- Requesting every permission at first launch, burning the one-time iOS prompt.
- No fallback when a permission is denied, leaving users stuck.
- Background location or broad photo access without a real need, inviting store rejection.
- Defining `TaskManager` tasks inside components instead of at module scope.
- Shipping without uploading source maps for builds and OTA updates, so crash reports are unreadable.
- Sending emails, tokens or payment details to the crash reporter.
- Tests that query by `testID` or internal state everywhere, breaking on every refactor.
- Real network calls in unit tests; flaky end-to-end tests against an unstable shared backend.

## How to talk about this in an interview
- Permissions: show the full flow (explain, request in context, handle "denied forever" with Settings, fall back) and the platform differences (iOS asks once, Android 13 notification permission, background location rules).
- Production crashes: tell it as a process, from scoping with crash reporting through mitigation (OTA rollback, feature flag) to the root-cause fix and a regression test.
- Testing: describe the pyramid you would apply to the resident app: unit tests for money math, component tests for forms, a few end-to-end flows for login and payments.

> **Fill in:** which device features the resident app uses (camera, photos, location, files, notifications) and how permission prompts are worded and timed.

> **Fill in:** the crash reporting tool used (if any), and one production bug you investigated: the symptom, cause and fix.

> **Fill in:** what automated tests exist for the app (unit, component, end to end) and what you added.

## Interview questions

**Q1. When is the Expo SDK not enough, and what are your options?**
When you need a capability no Expo module provides, such as a vendor SDK or an unusual platform API. In order: a maintained community library (with a config plugin and New Architecture support), a config plugin for missing native setup, then a local module with the Expo Modules API. All of these need a development build.
*Follow-up:* How do you judge a community library? Recent releases, New Architecture support, Expo compatibility, open issue health and whether it ships a config plugin.

**Q2. What is the Expo Modules API?**
A framework for writing native modules in Swift and Kotlin with a declarative DSL (`Name`, `Function`, `AsyncFunction`, `Events`, `View`, `Prop`). Expo generates the JSI bindings, so modules are fast and consistent across platforms, and TypeScript loads them with `requireNativeModule`. Scaffold with `npx create-expo-module@latest --local`.
*Follow-up:* What stays in TypeScript? Business logic; native code should expose small typed primitives.

**Q3. How do permissions differ between iOS and Android?**
iOS needs a usage string per resource in Info.plist and shows the system prompt only once; later changes happen in Settings. Android declares permissions in the manifest and requests dangerous ones at runtime; after two denials on Android 11+ the prompt stops. Both have granular choices (approximate location, limited photos), and Android 13+ requires a runtime permission for notifications.
*Follow-up:* What happens on iOS without the usage string? The app crashes when it requests that permission.

**Q4. How do you handle "denied forever"?**
Check `canAskAgain`: when it is false, requesting again does nothing, so show an explanation with a button that calls `Linking.openSettings()`. Re-check the permission when the app returns to the foreground and continue if it was granted, and always offer a fallback path such as manual entry.
*Follow-up:* How do you avoid getting there? Ask in context with a clear explanation first, and only for what the feature needs.

**Q5. How do you set iOS permission messages in an Expo app?**
Through config plugins in the app config, for example `["expo-camera", { "cameraPermission": "..." }]`, or the `ios.infoPlist` section. Prebuild writes them into Info.plist, so CNG builds always include them. Make each string specific about why the app needs access, or review may reject it.
*Follow-up:* And on Android? Modules add manifest permissions; adjust with `android.permissions` and remove unused ones with `android.blockedPermissions`.

**Q6. How do you request background location, and what are the implications?**
Request foreground access first, then `requestBackgroundPermissionsAsync()`; on Android 11+ the user grants "Allow all the time" in Settings. Define a task at module scope with `expo-task-manager` and start updates with `startLocationUpdatesAsync`. Google Play requires a declaration and justification, Apple scrutinizes it, and it costs battery, so use it only when the core feature needs it.
*Follow-up:* Does a resident app need it? Almost certainly not; foreground location is enough for "near me" features.

**Q7. Can you run code periodically in the background?**
Only opportunistically. Background task modules (expo-background-task in recent SDKs, on WorkManager and BGTaskScheduler) let the OS run a registered task at intervals it chooses, roughly 15 minutes at best, and iOS stops running them after a force-quit. Anything time-critical belongs on the server with push notifications.
*Follow-up:* Where must the task be defined? At module scope, because the OS may launch the JS without any UI.

**Q8. What are the app lifecycle states, and how do you react to them?**
`active` (foreground), `inactive` (iOS transitions such as the app switcher) and `background` (hidden, may be killed without warning). On `active`: refetch stale data, re-check permissions, enforce auto-lock. On `background`: save drafts and stop polling. Handle cold starts separately: session restore, initial deep link, last notification response.
*Follow-up:* Why not do cleanup on `inactive`? It also fires for brief interruptions, so treat only `background` as leaving.

**Q9. How do you set up crash reporting for an Expo app?**
Install `@sentry/react-native` with its Expo config plugin and Metro helper, initialize it at the app root with a DSN, environment and sampling, wrap the root component, and give EAS a Sentry auth token so source maps and native debug files upload with every build. Upload update source maps after each `eas update`, and tag events with the update ID.
*Follow-up:* What happened to `sentry-expo`? It was deprecated in favor of `@sentry/react-native`.

**Q10. How do you debug a crash in production?**
Scope it in the crash reporter (platform, app version, update ID, devices, start time versus releases), read the symbolicated stack and breadcrumbs, classify it (JS, native, ANR, out of memory), reproduce on a similar device in a release build with the same data, mitigate (OTA rollback or fix, feature flag, API fix), then fix the root cause, add a regression test and an alert.
*Follow-up:* The stack trace is minified. Why? Source maps were not uploaded for that build or update.

**Q11. What are source maps, and why do they matter for OTA updates?**
They map minified production JS back to your original TypeScript files and lines, so crash reports show real function names. Each EAS Update ships a new bundle, so each update needs its own uploaded source maps, matched by release and update identifiers, or crashes in that update stay unreadable.
*Follow-up:* What is the native equivalent? dSYM files on iOS and R8 mapping files on Android.

**Q12. How do you test a React Native app?**
A pyramid: Jest unit tests for pure logic (money math, date helpers, token refresh), React Native Testing Library component tests that render screens and interact like a user, a few integration tests with mocked network, and a small Maestro or Detox suite for critical flows on real builds. Run unit and component tests in CI on every pull request.
*Follow-up:* What do you test first? Logic where bugs cost money or trust: bill splitting, payment states, auth refresh.

**Q13. How do you mock native modules in Jest?**
Use the `jest-expo` preset, which already mocks many Expo modules, and add `jest.mock` calls in a setup file for the rest, such as SecureStore returning promises or the official AsyncStorage mock. Mock at the module boundary, keep mocks minimal and reset them between tests.
*Follow-up:* Why mock at all? Native code does not run in Node, where Jest executes.

**Q14. Maestro or Detox?**
Maestro: YAML flows, black-box, quick to write and tolerant of timing, runs against any build. Detox: JavaScript tests with gray-box synchronization on the app's idle state, more precise but more setup, especially with Expo. For a small team, Maestro usually delivers end-to-end coverage fastest.
*Follow-up:* What do you run end to end? Only critical flows: login, core screens, payment and a notification tap.

**Q15. A bug happens only on some Android devices. How do you approach it?**
Group crash reports by manufacturer, model, OS version and WebView or system component versions to find the pattern; reproduce on an emulator image or a cloud device with that OS; check for vendor quirks (battery optimizers killing tasks, custom keyboards, font scaling) and for memory limits on low-end models. Fix with defensive code or device-specific handling, and add that device class to the test matrix.
*Follow-up:* Example of a vendor quirk? Aggressive battery savers delaying notifications or killing background work.

**Q16. What makes a good test for a component that fetches data?**
Render it inside a fresh `QueryClientProvider` with retries off, mock the API at the network or client boundary, and assert what the user sees: loading, then data, or an error with a retry button. Use `findBy` queries for async states and test interactions such as pull-to-refresh or pagination through the UI.
*Follow-up:* Why a fresh client per test? Cached data leaking between tests causes order-dependent failures.

## Further reading
- [Expo Modules API](https://docs.expo.dev/modules/overview/)
- [Permissions in Expo](https://docs.expo.dev/guides/permissions/)
- [Unit testing with Jest in Expo](https://docs.expo.dev/develop/unit-testing/)
- [Using Sentry with Expo](https://docs.expo.dev/guides/using-sentry/)
- [React Native Testing Library](https://callstack.github.io/react-native-testing-library/)
