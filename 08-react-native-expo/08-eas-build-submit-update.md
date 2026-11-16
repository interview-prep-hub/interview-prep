# EAS Build, Submit and Update

> **Tier:** T1 (know cold) · **Related:** [Expo fundamentals](02-expo-fundamentals.md), [Push notifications](05-push-notifications.md), [Native modules, permissions and testing](09-native-modules-permissions-testing.md), [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md), [Rails API development](../05-ruby-on-rails/10-api-development.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md)
> **On your resume:** "EAS releases" of the Coloc resident app, live on the App Store and Google Play for 400+ residents; "Expo EAS" in your skills.

## Quick revision
- **EAS Build** compiles signed iOS and Android binaries in the cloud (or locally with `--local`). **EAS Submit** uploads them to App Store Connect and Google Play. **EAS Update** ships JS and asset changes over the air (OTA) to installed apps.
- `eas.json` holds **build profiles** (typically `development`, `preview`, `production`), submit profiles and CLI settings. Profiles set the distribution, environment, update channel and platform options.
- Signing: iOS needs a **distribution certificate** and a **provisioning profile** (development, ad hoc or App Store); Android needs an **upload keystore**, while Google Play App Signing holds the final app signing key. EAS can generate and store all of it (`eas credentials`).
- Internal distribution: iOS ad hoc builds for registered devices (`eas device:create`), Android APKs, or TestFlight and Play internal testing.
- Versions: `version` is user-facing (1.4.0); iOS `buildNumber` and Android `versionCode` must increase with every store upload. `cli.appVersionSource: "remote"` plus `autoIncrement` lets EAS manage build numbers.
- Play tracks: internal, closed (alpha), open (beta), production with staged rollouts. iOS: TestFlight (internal and external testers), then App Store review, optionally a 7-day phased release.
- Rejections: crashes and broken flows, no reviewer login, vague permission strings, missing account deletion, payments that break store rules, inaccurate privacy labels or Data safety forms, missing privacy manifest entries.
- EAS Update concepts: a **build** has a **channel** (set at build time); a channel points to a **branch**; you publish **updates** to branches. An update reaches a build only if their **runtime versions** match.
- `runtimeVersion` policies: `appVersion`, `nativeVersion`, `fingerprint` (a hash of the native layer), or a manual string. It protects users from JS that expects native code their binary lacks.
- OTA **can** change JS, styles, images and other bundled assets. It **cannot** change native code, native libraries, permissions, the app icon or name, config plugins or the Expo SDK version: those need a new store build.
- Bad release: for JS, republish the previous good update or roll back to the embedded bundle; for native bugs, halt the staged rollout and ship a fixed build (expedited review on iOS).
- Forced updates: the API returns a minimum supported version; the app compares it with its native version and blocks with a store link. Keep the API backward compatible because old versions stay installed for months.

## 1. What EAS gives you

| Service | Job | Main commands |
|---|---|---|
| EAS Build | Signed binaries on managed macOS and Linux machines | `eas build --platform ios --profile production` |
| EAS Submit | Upload to App Store Connect and Google Play | `eas submit --platform android --latest` |
| EAS Update | OTA updates of JS and assets | `eas update --branch production --message "Fix bill rounding"` |
| Credentials | Certificates, profiles, keystores, push keys | `eas credentials` |
| Workflows | YAML pipelines that chain build, submit and update jobs (newer; check the docs) | Files in `.eas/workflows/` |

```bash
npm install -g eas-cli   # or run every command through npx eas-cli
eas login
eas init                 # links the project and writes extra.eas.projectId into the app config
eas build:configure      # creates eas.json
```

### Going deeper: what happens during `eas build`
The CLI uploads your project (respecting `.gitignore` or `.easignore`), a fresh build machine installs dependencies, loads the profile's environment variables, runs `npx expo prebuild` if there are no native folders, runs CocoaPods or Gradle, signs with your credentials and stores the artifact (an `.ipa`, `.aab` or `.apk`). `package.json` hooks such as `eas-build-post-install` let you run extra steps.

## 2. eas.json build profiles

```json
{
  "cli": { "version": ">= 12.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "environment": "development",
      "channel": "development",
      "env": { "APP_VARIANT": "development" }
    },
    "preview": {
      "distribution": "internal",
      "environment": "preview",
      "channel": "preview",
      "env": { "APP_VARIANT": "preview" },
      "android": { "buildType": "apk" }
    },
    "production": {
      "environment": "production",
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": { "ascAppId": "1234567890" },
      "android": { "track": "internal", "releaseStatus": "draft" }
    }
  }
}
```

| Profile | Contains | Who installs it |
|---|---|---|
| `development` | `expo-dev-client`, connects to Metro, debug features | Developers |
| `preview` | A release build pointed at staging, installed directly (ad hoc or APK) | QA, product owners, the client |
| `production` | Store build (App Store `.ipa`, Play `.aab`), production API | Everyone, through the stores |

`environment` selects the EAS environment variables for the build (a newer field; check your eas-cli version), `env` sets plain values inline, and `channel` decides which EAS Update channel the binary listens to. `extends` lets one profile inherit from another, and `ios.simulator: true` produces a simulator build.

## 3. Cloud vs local builds

| | Cloud (`eas build`) | Local (`eas build --local`, `npx expo run:*`) |
|---|---|---|
| Machine | Expo's managed macOS and Linux workers | Your machine: Xcode for iOS (macOS only), Android SDK |
| iOS without a Mac | Yes | No |
| Reproducibility | Clean environment every time | Depends on your setup and caches |
| Cost and speed | Free tier with queues, paid plans for priority | No queue, uses your CPU |
| Good for | Releases, CI, teammates without Macs | Debugging build failures, quick native iteration |

## 4. Credentials and signing

**iOS** (requires a paid Apple Developer Program membership):
- **Distribution certificate:** proves builds come from your team; a team can hold only a few, and they expire yearly.
- **Provisioning profile:** ties the app ID, certificate and entitlements together, plus the device list for ad hoc builds. Types: development, ad hoc (internal distribution), App Store.
- **APNs key** for push notifications ([Push notifications](05-push-notifications.md)) and an **App Store Connect API key** for EAS Submit.

**Android:**
- You sign uploads with an **upload key** in a keystore. With **Play App Signing** (standard for new apps), Google re-signs the app with the **app signing key** it holds; a lost upload key can be reset through Play support, a lost signing key without Play App Signing means you can never update the app again.
- App links (`assetlinks.json`) need the fingerprint of the **app signing key** from Play Console, not the upload key.

EAS can generate, store and reuse all of these (`eas credentials`), which spares you keychain archaeology when someone leaves the team. Keep a secure backup of what EAS stores, and restrict who on the Expo account can access credentials.

## 5. Internal distribution

- **iOS ad hoc:** register testers' devices with `eas device:create` (it gives a URL to install a profile and collect the device ID), then build with `distribution: internal`; testers install from the build page. Adding devices requires a new build.
- **Android:** build an APK (`buildType: apk`) and share the link; no store needed.
- **TestFlight:** internal testers (members of your App Store Connect team, up to 100) get builds without review; external testers (up to 10,000) need a light beta review for the first build of a version. Builds expire after 90 days.
- **Play internal testing:** up to 100 testers, available within minutes, no full review.
- Preview builds should point at staging (via the build profile environment) and use their own update channel, so QA can test OTA updates before production.

## 6. Versioning

| Field | Platform meaning | Who sees it | Rule |
|---|---|---|---|
| `version` (for example `1.4.0`) | iOS `CFBundleShortVersionString`, Android `versionName` | Users in the store | Bump for releases; often drives `runtimeVersion` |
| `ios.buildNumber` | `CFBundleVersion` | App Store Connect | Unique and increasing per uploaded build of a version |
| `android.versionCode` | Integer version | Google Play | Must strictly increase with every upload, across all tracks |

With `"appVersionSource": "remote"` in `eas.json`, EAS stores the build numbers on its servers and `autoIncrement: true` bumps them per production build, so two developers never upload the same number; the `buildNumber` and `versionCode` in the app config are then ignored. With the local source, `autoIncrement` edits `app.json` (it cannot edit `app.config.ts`). You still bump `version` yourself.

## 7. Submitting to the stores

```bash
eas build --platform all --profile production --auto-submit   # build, then submit with the matching submit profile
eas submit --platform ios --latest                              # submit the latest finished build
```

**App Store path:** the build goes to App Store Connect and appears in TestFlight after processing; testers try it; you fill in the version's metadata (release notes, screenshots, privacy details, reviewer notes and demo account) and submit for review. Reviews often finish within a day or two. After approval, release manually, automatically, or as a **phased release** over 7 days (users can still update manually; you can pause it).

**Google Play path:** create the app in Play Console and upload the **first build manually** (the Play API cannot create apps). Then EAS Submit uploads to a track:

| Track | Audience | Notes |
|---|---|---|
| Internal testing | Up to 100 testers | Minutes, no full review |
| Closed testing (alpha) | Invited testers or groups | New personal developer accounts must run a closed test with a minimum number of testers for a period before production access (check Google's current numbers) |
| Open testing (beta) | Anyone who opts in | Listed on the store |
| Production | Everyone | Staged rollout by percentage; you can halt it |

Play reviews vary from hours to days. Google raises the required target Android API level every year, so staying on an old Expo SDK eventually blocks updates.

## 8. Store review: common rejection reasons

| Area | What goes wrong | Prevention |
|---|---|---|
| App completeness | Crashes, placeholder content, broken links, a backend that is down during review | Test the exact build on real devices; keep staging and production healthy |
| Reviewer access | Login required but no demo account | Provide credentials for a demo account with realistic data in the review notes |
| Permissions | Vague purpose strings ("We need the camera"), asking at launch | Specific strings ("Scan receipts to split a bill"), ask in context |
| Payments | Digital features sold outside in-app purchase, or confusion around real-world payments | Explain that rent and bills are real-world services paid outside in-app purchase |
| Privacy | Privacy labels or the Play Data safety form not matching what the app and SDKs collect; no privacy policy | Audit SDKs, keep forms current, link a privacy policy |
| Account deletion | Users can create accounts but cannot delete them in the app (Apple requirement; Google requires a deletion path too) | Add an in-app deletion flow backed by an API endpoint |
| Privacy manifests | iOS apps using certain "required reason" APIs must declare them | Expo supports `ios.privacyManifests` in the app config; check SDK warnings |
| Login options | Third-party logins without Sign in with Apple or an equivalent option | Follow Apple's login guideline (check the current wording) |

Answer rejections in App Store Connect's Resolution Center calmly and specifically; many are resolved by clarification. For urgent bug fixes, Apple accepts requests for an expedited review.

## 9. EAS Update (over-the-air)

**Concepts:**
- An **update** is a JS bundle plus assets for one platform and one runtime version.
- A **branch** is an ordered list of updates (like a git branch of releases).
- A **channel** is baked into each build at build time (from the `channel` field of its profile). A channel points to a branch, by default the one with the same name, and you can repoint it.
- The app downloads only updates whose **runtime version** matches its own.

```bash
eas update:configure                                         # adds expo-updates config and the updates URL
eas update --branch preview --message "Fix event time zone"   # QA verifies on preview builds
eas update --branch production --message "Fix event time zone"
eas channel:edit production --branch production-hotfix       # repoint a channel to another branch
```

```json
{
  "expo": {
    "runtimeVersion": { "policy": "appVersion" },
    "updates": { "url": "https://u.expo.dev/your-project-id", "checkAutomatically": "ON_LOAD", "fallbackToCacheTimeout": 0 }
  }
}
```

**Runtime version policies:**

| Policy | Runtime version equals | Trade-off |
|---|---|---|
| `appVersion` | The `version` field | Simple; you must bump `version` whenever native code changes, or incompatible updates can reach old builds |
| `nativeVersion` | `version` plus build number | Stricter: each build gets its own runtime |
| `fingerprint` | A hash of the native layer (native dependencies, config plugins, native config) | Changes exactly when native code changes; less human error |
| Manual string | Whatever you set | Full control, full responsibility |

**What an OTA update can and cannot change:**

| Can change | Cannot change (needs a store build) |
|---|---|
| JS logic, screens, bug fixes in TypeScript | Native code and libraries with native code |
| Styles, copy, translations | Permissions and Info.plist or AndroidManifest entries |
| Images and other assets bundled through JS | App icon, name, splash configuration, bundle ID |
| Feature flags and API calls (within existing native capabilities) | Config plugins, Expo SDK or React Native upgrades |

Store policies allow OTA updates of interpreted code as long as you do not change the app's primary purpose or sneak in significant new features that would need review; check the current guidelines.

**How runtimeVersion protects users:** suppose 1.4.0 adds a native camera library. JS that calls it would crash on 1.3.0 binaries, which do not contain that native code. Because the native layer changed, the runtime version changes (by bumping `version` with `appVersion`, automatically with `fingerprint`), so the new JS is published for the new runtime only; 1.3.0 users keep receiving fixes built for their runtime until they update from the store.

**When updates apply:** with `checkAutomatically: "ON_LOAD"` and `fallbackToCacheTimeout: 0`, the app launches immediately with the cached update, downloads a newer one in the background and runs it on the next cold start. For critical fixes you can check and apply in-app:

```ts
import * as Updates from 'expo-updates';

export async function applyCriticalUpdate() {
  if (__DEV__ || !Updates.isEnabled) return;
  try {
    const result = await Updates.checkForUpdateAsync();
    if (!result.isAvailable) return;
    await Updates.fetchUpdateAsync();
    await Updates.reloadAsync(); // restarts the JS app: only at a safe moment, never mid-payment
  } catch {
    // offline or update server unreachable: keep running the current version
  }
}
```

The `useUpdates()` hook exposes the same state (available, downloaded, currently running) for an in-app "Restart to update" banner. Tag crash reports with `Updates.updateId` and `Updates.runtimeVersion`, so you can tell which update crashed.

**Rollouts and rollbacks:**
- **Gradual rollout:** EAS supports percentage-based rollouts, either per update or by splitting a channel between two branches; check `eas update --help` and the channel commands for your CLI version.
- **Rollback:** republish the last good update so it becomes the newest on the branch (`eas update:republish`), or roll the channel back to the bundle embedded in the binary (`eas update:roll-back-to-embedded`). Command names come from recent eas-cli versions; run `eas update --help` to confirm. Clients pick up the rollback on their next check.
- expo-updates has built-in error recovery that falls back when a new update crashes at startup, but treat that as a safety net, not a plan.

## 10. Forced updates and minimum supported versions

```ts
// lib/version-gate.ts
import * as Application from 'expo-application';
import { Linking, Platform } from 'react-native';

export function compareVersions(a: string, b: string): number {
  const pa = a.split('.').map(Number);
  const pb = b.split('.').map(Number);
  for (let i = 0; i < Math.max(pa.length, pb.length); i++) {
    const diff = (pa[i] ?? 0) - (pb[i] ?? 0);
    if (diff !== 0) return Math.sign(diff);
  }
  return 0;
}

export async function mustUpdate(): Promise<boolean> {
  const res = await fetch(`${process.env.EXPO_PUBLIC_API_URL}/api/v1/app_config`);
  if (!res.ok) return false; // never lock users out because the config call failed
  const config = (await res.json()) as { min_version: { ios: string; android: string } };
  const minimum = Platform.OS === 'ios' ? config.min_version.ios : config.min_version.android;
  const current = Application.nativeApplicationVersion ?? '0.0.0';
  return compareVersions(current, minimum) < 0;
}

export const openStore = () =>
  Linking.openURL(
    Platform.OS === 'ios'
      ? 'https://apps.apple.com/app/id1234567890'
      : 'https://play.google.com/store/apps/details?id=com.example.coliving',
  );
```

Show a blocking screen with an "Update" button when `mustUpdate()` is true, and a dismissible banner for "recommended" updates. A second line of defense lives in Rails: the app sends its version in a header (for example `X-App-Version`), and the API can answer `426 Upgrade Required` for versions it no longer supports. Mobile clients cannot be updated instantly, so evolve the API additively and version endpoints when you must break them ([Rails API development](../05-ruby-on-rails/10-api-development.md)).

## 11. A release checklist

1. Decide: OTA update (JS-only change, same runtime) or store build (native change, new permission, SDK upgrade)?
2. Bump `version` if needed; confirm the runtime version policy does the right thing.
3. Make sure the Rails API changes are deployed first and are backward compatible with older app versions.
4. Check environment variables and keys per environment (production API URL, live payment keys, Sentry DSN).
5. Test a preview build on real iOS and Android devices: login, core flows (events, tasks, bills, payments), push notifications, deep links, offline behavior.
6. Update store metadata: release notes, screenshots if the UI changed, privacy answers if data collection changed, reviewer notes and demo account.
7. Build and submit; roll out gradually (phased release on iOS, staged rollout on Play).
8. Watch crash-free sessions, error rates and support messages for the first hours and days; keep a rollback plan ready (republish an update, halt the rollout).
9. Tag the release in git and note which update IDs and builds belong to it.

## 12. CI with EAS

```yaml
# .github/workflows/preview-update.yml
name: Preview update
on:
  push:
    branches: [main]
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: npm ci
      - run: npx tsc --noEmit && npx jest --ci
      - run: eas update --branch preview --message "${{ github.sha }}" --non-interactive
```

A common split: every merge to `main` publishes an update to the `preview` branch for QA; a git tag triggers `eas build --platform all --profile production --non-interactive --no-wait` with auto-submit; promoting a tested update to production stays a deliberate, manual step. EAS Workflows can express the same pipeline on Expo's infrastructure with YAML files in `.eas/workflows/`; it is relatively new, so check the docs for syntax. More on pipelines in [CI/CD and deployment](../14-devops-and-tools/04-ci-cd-and-deployment.md).

## Common pitfalls
- Publishing an update that depends on a new native module to a runtime whose binaries do not have it, crashing on launch (the classic `appVersion` mistake: native change without a `version` bump).
- Publishing to the wrong branch, or a production build listening to the preview channel.
- Uploading a `versionCode` or `buildNumber` that was already used; let EAS manage them remotely.
- Losing the Android signing key without Play App Signing.
- Using the upload key fingerprint in `assetlinks.json`.
- No reviewer account, or a staging backend that is down during review.
- Breaking API changes that crash old app versions still installed by many users.
- Calling `Updates.reloadAsync()` in the middle of a user flow.
- Forgetting that ad hoc builds only install on registered devices.

## How to talk about this in an interview
- Walk through a release you know: "Preview builds for QA, production builds with auto-incremented build numbers, submission through EAS to TestFlight and Play internal testing, then a gradual rollout. JS-only fixes go out as EAS Updates on the matching runtime version."
- Make the OTA boundary crisp: JS and assets yes, native code and permissions no, and runtimeVersion enforces it.
- The resume lists EAS releases for the Coloc resident app; describe your actual part in them honestly (running builds, preparing store metadata, publishing updates, fixing build failures).

> **Fill in:** which EAS profiles and channels the resident app uses, and which runtime version policy.

> **Fill in:** your role in releases (who builds, who submits, how often), and a store rejection or build failure you handled.

> **Fill in:** whether the app has a forced-update or minimum-version mechanism, and how a bad release was (or would be) rolled back.

## Interview questions

**Q1. What are EAS Build, EAS Submit and EAS Update?**
EAS Build produces signed iOS and Android binaries on cloud machines (or locally). EAS Submit uploads them to App Store Connect and Google Play. EAS Update publishes JS and asset updates that installed apps download over the air, targeted by channel and runtime version. Together they cover the release pipeline without maintaining Mac build machines or signing setups by hand.
*Follow-up:* What replaced `expo build` and `expo publish`? EAS Build and EAS Update.

**Q2. Explain eas.json build profiles.**
Profiles are named build configurations. `development` includes the dev client for engineers, `preview` is a release build pointed at staging and installed directly by QA, and `production` targets the stores. Each sets the distribution, environment variables, update channel and platform options, and profiles can extend each other.
*Follow-up:* What does `channel` do in a profile? It bakes the EAS Update channel into the binary, deciding which updates it receives.

**Q3. Cloud or local builds?**
Cloud builds run on clean, managed machines (including macOS for iOS), integrate with credentials and CI, and work for teammates without Macs, at the cost of queue times on the free tier. Local builds (`eas build --local` or `npx expo run:*`) are faster to iterate and easier to debug, but depend on each machine's setup.
*Follow-up:* When do you build locally? To debug a native build failure or iterate on a config plugin.

**Q4. How does code signing work on iOS and Android?**
iOS signs with a distribution certificate and embeds a provisioning profile that binds app ID, certificate, entitlements and (for ad hoc) device IDs. Android signs uploads with an upload key; with Play App Signing, Google signs the distributed app with the app signing key it keeps. EAS can create and store these credentials.
*Follow-up:* What happens if you lose the Android key? With Play App Signing you can reset the upload key; without it, you can never update the app again.

**Q5. How do you get test builds to QA and the client?**
Preview profile builds with internal distribution: ad hoc iOS builds for registered devices (`eas device:create`) and APKs for Android, installed from a link. Alternatively TestFlight and Play internal testing, which add store processing but mirror the real install path.
*Follow-up:* A new tester's iPhone cannot install the ad hoc build. Why? The device is not in the provisioning profile; register it and rebuild.

**Q6. What is the difference between version, buildNumber and versionCode?**
`version` is the user-facing release (1.4.0). iOS `buildNumber` and Android `versionCode` identify individual uploaded builds and must increase; Play rejects a `versionCode` it has seen before. With a remote app version source and `autoIncrement`, EAS manages build numbers so collisions cannot happen.
*Follow-up:* Which one usually drives the runtime version? `version`, under the `appVersion` policy.

**Q7. How do you release to the App Store and Play Store?**
Build with the production profile, submit with EAS Submit (or `--auto-submit`), test the build in TestFlight and a Play testing track, complete metadata, privacy forms and reviewer notes, submit for review, then roll out gradually: phased release on iOS, staged percentage rollout on Play. Monitor crashes and be ready to halt or roll back.
*Follow-up:* Anything special about the first Android upload? It must be done manually in Play Console before EAS Submit can upload.

**Q8. What are common App Store rejection reasons, and how do you avoid them?**
Crashes or broken flows during review, no demo account for a login-gated app, vague permission strings, missing in-app account deletion, payments that violate in-app purchase rules, and privacy labels or manifests that do not match the app's behavior. Test the exact build, provide reviewer credentials and notes, write specific purpose strings and audit SDK data collection.
*Follow-up:* What do you do after a rejection? Read the cited guideline, fix or clarify in Resolution Center, resubmit; request expedited review for urgent fixes.

**Q9. What can an OTA update change, and what can it not?**
It can change anything in the JS bundle and bundled assets: logic, screens, styles, copy, images. It cannot change native code or native libraries, permissions and other native config, the app icon, name or splash, config plugins, or the Expo SDK and React Native versions; those need a new store build.
*Follow-up:* Can you add a new screen over the air? Yes, if it only uses native capabilities already in the binary.

**Q10. What is runtimeVersion, and how does it protect users?**
A label for the native layer of a build; an update is delivered only to builds with the same runtime version. When native code changes, the runtime version changes (manually with `appVersion`, automatically with `fingerprint`), so JS that needs new native code never reaches binaries that lack it, which would crash.
*Follow-up:* Which policy would you choose? `fingerprint` reduces human error; `appVersion` is simpler but relies on bumping `version` for every native change.

**Q11. Channels vs branches?**
A channel is set on a build at build time; a branch is where you publish updates. A channel points to a branch (by default the same name), and you can repoint it, for example `production` to a hotfix branch, without rebuilding any binary.
*Follow-up:* How do you promote a tested update? Republish it from the preview branch to production, or repoint the channel.

**Q12. A release is broken. What do you do?**
First scope it with crash reporting: which platforms, versions and update IDs. For a JS bug shipped by OTA, republish the last good update (or roll back to the embedded bundle) immediately, then fix forward. For a native bug in a store build, halt the staged rollout or pause the phased release, ship a fixed build with a higher build number and request expedited review; mitigate meanwhile with server-side flags or an OTA patch if the fix is JS-only. Write a short postmortem.
*Follow-up:* Can you roll back a store release? Not to an older binary; you can only stop the rollout and ship a newer version.

**Q13. How do you roll out changes gradually?**
For store builds, phased release on iOS (7 days, pausable) and staged percentage rollouts on Play (haltable). For OTA, EAS percentage rollouts per update or by splitting a channel across branches. Watch crash-free sessions before widening the rollout.
*Follow-up:* What do you monitor? Crash rate per release, error spikes, key funnel metrics such as successful payments, and support tickets.

**Q14. How do you force users to update?**
The API exposes a minimum supported version per platform; at launch the app compares it with `Application.nativeApplicationVersion` and shows a blocking screen with a store link if it is older. As a backstop, the API rejects unsupported versions (for example with `426`). Never block if the check itself fails, and keep the API backward compatible for supported versions.
*Follow-up:* Forced update or OTA? OTA for JS fixes on the same runtime; forced update when users must move to a new binary.

**Q15. When does the app actually apply an EAS Update?**
By default it launches with the cached bundle, checks for a newer update and downloads it in the background, then runs it on the next cold start. You can check, fetch and reload in-app with `expo-updates` for critical fixes, at a safe moment, or set a fallback timeout to wait briefly for an update at launch.
*Follow-up:* Why not reload immediately? Restarting mid-flow loses user input or interrupts a payment.

**Q16. How would you set up CI/CD for an Expo app?**
GitHub Actions (or EAS Workflows): on pull requests, type-check, lint and run Jest; on merges to `main`, publish an update to the preview branch; on release tags, run production builds with auto-submit. Store `EXPO_TOKEN` as a CI secret and keep production promotion manual.
*Follow-up:* Why not OTA every merge to production? Untested changes would reach every user instantly; promote deliberately after QA.

**Q17. Are OTA updates allowed by Apple and Google?**
Yes, within limits: you may update interpreted code such as a JS bundle, provided you do not change the app's primary purpose, add major features that bypass review or download native code. Use OTA for fixes and incremental improvements, and ship big features through review.
*Follow-up:* What is clearly not allowed? Turning the app into something different or enabling features reviewers never saw.

## Further reading
- [EAS Build](https://docs.expo.dev/build/introduction/)
- [EAS Submit](https://docs.expo.dev/submit/introduction/)
- [EAS Update](https://docs.expo.dev/eas-update/introduction/)
- [Runtime versions and updates](https://docs.expo.dev/eas-update/runtime-versions/)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
