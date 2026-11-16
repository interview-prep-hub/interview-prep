# Expo Fundamentals

> **Tier:** T1 (know cold) · **Related:** [React Native fundamentals](01-react-native-fundamentals.md), [Navigation and Expo Router](03-navigation-expo-router.md), [EAS Build, Submit and Update](08-eas-build-submit-update.md), [Native modules, permissions and testing](09-native-modules-permissions-testing.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md)
> **On your resume:** "React Native (Expo)" and "Expo EAS" in your skills; the Coloc resident app at Devntech, built with Expo and TypeScript and shipped through EAS releases.

## Quick revision
- Expo is a framework around React Native: the **Expo SDK** (versioned native modules), **Expo CLI** (`npx expo ...`), **Expo Router**, **Expo Go**, and **EAS** cloud services (Build, Submit, Update).
- Each SDK release pins a React Native version and a tested set of module versions (SDK 53: RN 0.79 and React 19; SDK 54: RN 0.81).
- **Expo Go** is a prebuilt store app with a fixed set of native modules for one SDK version. It cannot load your own native code, config plugins, app icon or permission strings.
- A **development build** is your own debug binary that includes `expo-dev-client`. Switch to one when you add a native library Expo Go lacks, write config plugins, need your own URL scheme or universal links, or test remote push on Android (Expo Go dropped it in SDK 53).
- Rebuild the development client only when native code changes; JS changes still hot reload from Metro.
- **Continuous Native Generation (CNG):** `npx expo prebuild` generates `ios/` and `android/` from the app config, config plugins and installed packages. Treat those folders as build output and gitignore them.
- **Config plugins** are functions that edit the generated native projects (Info.plist, AndroidManifest, Gradle, entitlements) during prebuild. Libraries ship them; you can write your own with mods from `expo/config-plugins`.
- `app.json` is static JSON; `app.config.ts` is code that returns the config, for environment-specific names, bundle IDs and values.
- `EXPO_PUBLIC_*` variables are inlined into the JS bundle at build time and are **public**. Access them statically: `process.env.EXPO_PUBLIC_API_URL`.
- **EAS environment variables** (development, preview, production environments with plain, sensitive or secret visibility) feed builds and updates; manage them with `eas env:*`.
- Use `npx expo install <pkg>` so versions match your SDK; `npx expo install --fix` repairs mismatches; `npx expo-doctor` checks project health.
- Upgrading an SDK: read the changelog, bump `expo`, `--fix`, doctor, update native libraries, regenerate native projects, rebuild dev clients, test, ship a new store build.
- Old words: "managed" (no native folders), "bare" (committed native folders), "eject" (one-way managed to bare). CNG plus config plugins replaced ejecting; `expo build` became EAS Build and `expo publish` became EAS Update.
- Custom native code: Expo SDK first, then a community library with a config plugin, then your own config plugin, then the **Expo Modules API** (Swift and Kotlin).

## 1. What Expo is

Expo is the most widely used React Native framework, and the React Native docs recommend starting new apps with a framework like it. An Expo app *is* a React Native app; Expo adds the pieces you would otherwise assemble yourself.

| Part | What it gives you |
|---|---|
| Expo CLI (`npx expo`) | Metro dev server, `start`, `install`, `prebuild`, `run:ios` / `run:android`, `export` |
| Expo SDK | Native modules such as `expo-notifications`, `expo-camera`, `expo-secure-store`, versioned and tested together per SDK |
| Expo Router | File-based navigation on top of React Navigation |
| Expo Go | A store app that runs projects without a native build |
| Development builds | Your own debug app with `expo-dev-client` (launcher plus dev menu) |
| EAS | Cloud services: Build, Submit, Update (OTA) and newer additions such as Workflows; free tier plus paid plans |

```bash
npx create-expo-app@latest resident-app   # TypeScript + Expo Router template
cd resident-app
npx expo start                             # QR code for Expo Go or a development build
```

Expo is not "less native" than plain React Native. With development builds and config plugins you can use any native library or write your own module, and you still get Expo's tooling.

## 2. Expo Go vs development builds

| | Expo Go | Development build |
|---|---|---|
| What it is | One store app shared by every project | Your own app binary with `expo-dev-client` |
| Native code | Only the modules bundled in Expo Go for its SDK | Any library, plus your own modules |
| Config plugins, icon, name, bundle ID, permission strings | Not applied | Applied |
| Deep links | `exp://` URLs only | Your `scheme`, universal links and app links |
| Push notifications | Limited; no Android remote push from SDK 53 | Full support |
| SDK version | The single SDK the store version supports | Whatever your project uses |
| Build needed | No | Yes, and again whenever native dependencies change |

You must switch to a development build when you:
- add a library with native code that Expo Go does not include (react-native-mmkv, react-native-firebase, many payment or analytics SDKs);
- add or change config plugins, app name, icon, splash, bundle identifier or Info.plist strings;
- test your own URL scheme, universal links or app links;
- test remote push notifications (on Android from SDK 53), background tasks or anything tied to your real app identity;
- need an SDK version that the store's Expo Go no longer supports.

```bash
npx expo install expo-dev-client
eas build --profile development --platform android   # cloud build, installed on test devices
npx expo run:ios                                      # or build locally (runs prebuild, needs Xcode)
npx expo start                                        # the dev client connects to Metro like Expo Go
```

Most teams move to development builds early. Expo Go stays useful for quick experiments and onboarding.

## 3. Continuous Native Generation (prebuild)

A native iOS or Android project is mostly boilerplate derived from a few settings: app name, bundle ID, permissions, icons, linked libraries. Hand-editing those projects makes every upgrade a painful diff of Xcode and Gradle files. With CNG the source of truth is your **app config + `package.json` + config plugins**, and the native folders are regenerated whenever needed.

```bash
npx expo prebuild                  # generate ios/ and android/
npx expo prebuild --clean          # delete and regenerate (after config, plugin or SDK changes)
npx expo prebuild --platform ios   # one platform only
npx expo config --type introspect  # print the config after plugins and mods are applied
```

- EAS Build runs prebuild automatically when `ios/` and `android/` are absent.
- If you commit those folders, EAS builds them as they are, and native settings in the app config (plugins, permissions, icons) no longer reach those folders automatically: you maintain native changes by hand. That is the old "bare" setup, still valid when you need it.
- CNG projects list `/ios` and `/android` in `.gitignore`.
- Anything native you cannot express in the app config goes into a config plugin, so it survives regeneration.

### Going deeper: what prebuild does
1. Copies a native template that matches your SDK version.
2. Applies the app config: name, identifiers, version, icons, splash, permissions, scheme.
3. Runs config plugins in order. Each plugin registers *mods* that edit a specific native file (Info.plist, entitlements, AndroidManifest, Gradle files).
4. Autolinking finds native modules in `node_modules`, and CocoaPods or Gradle wire them into the projects.

## 4. Config plugins

Libraries that need native configuration ship a plugin you list in the app config, often with options:

```json
{
  "expo": {
    "plugins": [
      "expo-router",
      ["expo-camera", { "cameraPermission": "Allow $(PRODUCT_NAME) to scan receipts for shared bills." }],
      ["expo-notifications", { "icon": "./assets/notification-icon.png", "color": "#2f6fed" }],
      ["expo-build-properties", { "ios": { "useFrameworks": "static" } }]
    ]
  }
}
```

Writing your own is a function from config to config that registers mods. This one adds a vendor SDK key (a public key, not a secret) to both native projects:

```ts
// app.config.ts
import type { ConfigContext, ExpoConfig } from 'expo/config';
import { AndroidConfig, withAndroidManifest, withInfoPlist, type ConfigPlugin } from 'expo/config-plugins';

const withVendorKey: ConfigPlugin<{ key: string }> = (config, { key }) => {
  config = withInfoPlist(config, (cfg) => {
    cfg.modResults['VendorSdkKey'] = key; // ends up in ios/<App>/Info.plist
    return cfg;
  });
  return withAndroidManifest(config, (cfg) => {
    const app = AndroidConfig.Manifest.getMainApplicationOrThrow(cfg.modResults);
    AndroidConfig.Manifest.addMetaDataItemToMainApplication(app, 'vendor_sdk_key', key); // <meta-data> entry
    return cfg;
  });
};

export default ({ config }: ConfigContext): ExpoConfig =>
  withVendorKey({ ...config, name: 'Coliving', slug: 'coliving' }, { key: process.env.VENDOR_SDK_KEY ?? '' });
```

Rules of thumb:
- Plugins must be **idempotent**: running prebuild twice must not add duplicate entries.
- Prefer typed mods (`withInfoPlist`, `withAndroidManifest`, `withEntitlementsPlist`, `withAppBuildGradle`); `withDangerousMod` (raw file edits) is a last resort.
- Verify with `npx expo prebuild --clean` and inspect the generated files.

## 5. app.json vs app.config.ts

| | `app.json` | `app.config.ts` (or `.js`) |
|---|---|---|
| Format | Static JSON | Code that returns the config |
| Use for | Fixed settings | Values that depend on environment variables or build profile |
| Both present | Read first | Receives the `app.json` contents as `config` and can override them |
| Gotchas | Tools can edit it for you (version bumps with a local version source) | Keep it deterministic and fast; tools cannot rewrite code for you |

A common use is **app variants**, so development, preview and production builds install side by side on one phone:

```ts
// app.config.ts
import type { ConfigContext, ExpoConfig } from 'expo/config';

const variant = process.env.APP_VARIANT ?? 'production'; // set per build profile in eas.json
const isProduction = variant === 'production';
const idSuffix = isProduction ? '' : `.${variant}`;

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: isProduction ? 'Coliving' : `Coliving (${variant})`,
  slug: 'coliving',
  scheme: isProduction ? 'myapp' : `myapp-${variant}`,
  ios: { ...config.ios, bundleIdentifier: `com.example.coliving${idSuffix}` },
  android: { ...config.android, package: `com.example.coliving${idSuffix}` },
});
```

At runtime, `Constants.expoConfig` (from `expo-constants`) exposes the public config, including `extra`. Everything embedded there is readable by anyone who downloads the app.

## 6. Commonly used SDK modules

| Module | Use it for | Notes |
|---|---|---|
| `expo-router` | File-based navigation | [Navigation](03-navigation-expo-router.md) |
| `expo-notifications` | Push tokens, local notifications, Android channels, badges | Android remote push needs a development build from SDK 53; [Push notifications](05-push-notifications.md) |
| `expo-secure-store` | Small secrets such as refresh tokens | Keychain on iOS, Keystore-encrypted values on Android; [State, storage, auth](04-state-storage-auth-offline.md) |
| `expo-image` | Images with memory and disk caching, blurhash placeholders, transitions | Replaces most uses of the core `Image` |
| `expo-camera` | Camera preview, photos, barcode scanning | `CameraView` + `useCameraPermissions` (the API since SDK 51) |
| `expo-image-picker` | Pick or take a photo | Uses system pickers, fewer permissions |
| `expo-location` | Foreground and background location | Background needs extra permissions and store declarations |
| `expo-file-system` | Read, write and download files | SDK 54 made an object-based API the default; the old functions moved to `expo-file-system/legacy` |
| `expo-updates` | OTA updates from EAS Update | [EAS](08-eas-build-submit-update.md) |
| `expo-local-authentication` | Face ID and fingerprint prompts | A local gate, not server authentication |
| `expo-auth-session`, `expo-web-browser` | OAuth and OpenID Connect flows | PKCE by default |
| `expo-splash-screen`, `expo-font`, `expo-constants`, `expo-device`, `expo-linking`, `expo-haptics` | App plumbing | |
| `expo-audio`, `expo-video` | Audio and video | Replace the older `expo-av` |
| `expo-sqlite` | Local relational data | Offline-heavy features |

## 7. Environment variables

```bash
# .env (read by Expo CLI; put machine-specific values in .env.local and keep it out of git)
EXPO_PUBLIC_API_URL=https://staging-api.example.com
EXPO_PUBLIC_SENTRY_DSN=https://publickey@o0.ingest.sentry.io/0
```

```ts
// Inlined at bundle time, so it must be written exactly like this
const API_URL = process.env.EXPO_PUBLIC_API_URL;

// Not inlined, stays undefined in the app:
// const { EXPO_PUBLIC_API_URL } = process.env;
// const value = process.env[`EXPO_PUBLIC_${name}`];
```

- Only `EXPO_PUBLIC_*` variables reach app code (SDK 49+). Other variables are visible to Node-side code such as `app.config.ts` and build tooling only.
- **Everything in the bundle is public.** Pulling strings out of an app binary takes minutes. Fine to expose: the API base URL, a Sentry DSN, a Stripe *publishable* key, analytics write keys. Never expose: payment secret keys, Expo access tokens, signing credentials, third-party API secrets. Keep those on the Rails server and proxy the calls.
- Values are fixed when the bundle is built: changing one needs a new build or an EAS Update, and locally a Metro restart (`npx expo start --clear`).
- EAS Build uploads your project without gitignored files, so a local `.env` is not there. Define variables in EAS instead.

**EAS environment variables.** Variables live in the `development`, `preview` and `production` environments, each with a visibility (plain text, sensitive, or secret, which cannot be read back after creation). A build profile selects an environment, and `eas update` can use one too. They replaced the older `eas secret:*` commands (check the docs for exact flags).

```bash
eas env:create                            # interactive: environment, name, value, visibility
eas env:pull --environment development    # write the development values to .env.local
```

## 8. Upgrading Expo SDKs

1. Read the SDK release notes and the breaking changes of your other native libraries. Upgrade **one SDK at a time**.
2. Bump the SDK, then align every Expo-managed package.
3. Run `npx expo-doctor` to catch duplicate native modules, incompatible versions and config problems.
4. Update non-Expo native libraries to versions that support the new React Native version and the New Architecture.
5. Regenerate native projects (`npx expo prebuild --clean`) or, with committed native folders, apply the native template changes by hand (Expo provides an upgrade helper for this).
6. Rebuild development clients, run the tests, smoke-test both platforms in release builds.
7. The native layer changed, so the runtime version changes: ship new store builds. Old binaries keep getting only compatible OTA updates.

```bash
npx expo install expo@^54.0.0   # example target SDK
npx expo install --fix          # align Expo-managed dependencies with that SDK
npx expo-doctor                 # health check
npx expo prebuild --clean       # CNG projects that prebuild locally
```

Typical breakages: renamed APIs (`expo-camera` in SDK 51, `expo-file-system` in SDK 54), notification handler fields, Android edge-to-edge (SDK 53 and 54), libraries without New Architecture support, React 19 type changes, and stale CocoaPods or Gradle caches. Google Play raises the required target Android API level every year, so staying on an old SDK eventually blocks store updates.

## 9. Managed, bare and eject: the old vocabulary

| Term | What it meant | Today |
|---|---|---|
| Managed workflow | No native folders; builds through Expo's classic `expo build` service | CNG: native folders are generated; EAS Build runs prebuild |
| Bare workflow | Native folders committed and edited by hand | Still possible: commit `ios/` and `android/`; Expo modules, EAS Build and EAS Update still work |
| Eject (`expo eject`) | One-way conversion from managed to bare | Gone. Prebuild generates the folders and you can delete and regenerate them at will |
| `expo build` | Classic cloud builds | Replaced by EAS Build |
| `expo publish` | Classic OTA updates | Replaced by EAS Update |
| Global `expo-cli` | Globally installed CLI | The local CLI in the `expo` package, run with `npx expo` (SDK 46+) |

The point of the change: you no longer pick between "easy but limited" and "flexible but manual". You keep Expo's tooling and still customize native code, through config plugins and your own modules.

## 10. When you need custom native code

1. **An Expo SDK module exists?** Use it.
2. **A maintained community library** with Expo and New Architecture support (React Native Directory shows both)? Install it with `npx expo install` and use a development build.
3. **The library needs native configuration** it does not automate? Write a config plugin.
4. **Nothing exists** (a vendor SDK, a home-screen widget, a specific Bluetooth device): write a module with the **Expo Modules API**, a Swift and Kotlin DSL backed by JSI, scaffolded with `npx create-expo-module@latest --local`.
5. **A library has a bug:** `patch-package` as a stopgap while you send the fix upstream.

Details and code: [Native modules, permissions and testing](09-native-modules-permissions-testing.md).

## 11. A typical Expo project layout

```text
resident-app/
  app/                  # Expo Router routes: _layout.tsx, (tabs)/, events/[id].tsx
  components/           # shared UI (AppText, Button, EmptyState)
  features/             # bills/, events/, tasks/: hooks, API calls, screens' building blocks
  lib/                  # api client, query client, session, push registration
  assets/               # icon, splash, images, fonts
  modules/              # local Expo modules (only if you write native code)
  app.config.ts         # or app.json
  eas.json              # build, submit and update profiles
  package.json
  tsconfig.json
  .env                  # EXPO_PUBLIC_* defaults for local development
```

Nothing native is committed in a CNG project: `ios/` and `android/` appear only after `npx expo prebuild` or `npx expo run:*`. Keeping API calls and query hooks in `features/` or `lib/` (not inside route files) keeps screens thin and makes them easy to test.

## Common pitfalls
- Adding a native library and launching in Expo Go, which fails with a "native module not found" style error: you need a development build.
- `npm install expo-camera` pulling a version built for a different SDK; use `npx expo install`.
- Hand-editing `ios/` or `android/` in a CNG project and losing the change on the next `prebuild --clean`.
- Committing the native folders by accident, after which config and plugin changes silently stop applying.
- Secrets in `EXPO_PUBLIC_*` variables or in `extra`.
- Reading `process.env` dynamically or by destructuring, which is not inlined and comes back `undefined`.
- Forgetting to rebuild the development client after adding a native dependency.
- Jumping several SDKs at once, which mixes many breaking changes into one undiagnosable failure.
- Config plugins that are not idempotent and duplicate manifest entries on every prebuild.

## How to talk about this in an interview
- Frame Expo as a deliberate choice: "Expo gives us the SDK modules, development builds, config plugins and EAS, so a small team can ship to both stores without maintaining native projects by hand."
- Connect it to the Coloc resident app: an Expo and TypeScript app, live on the App Store and Google Play for 400+ residents, released with EAS.
- Show you know the modern vocabulary: CNG and config plugins instead of "ejecting", EAS Build and EAS Update instead of `expo build` and `expo publish`.

> **Fill in:** whether the team develops with Expo Go or development builds, and whether `ios/` and `android/` are generated (CNG) or committed.

> **Fill in:** which Expo SDK modules the resident app relies on (notifications, secure storage, image picker, updates, others) and one problem you hit with one of them.

> **Fill in:** an SDK upgrade you took part in, what broke and how you verified it.

## Interview questions

**Q1. What is Expo, and how does it relate to React Native?**
Expo is a framework on top of React Native: the Expo SDK of native modules, the Expo CLI, Expo Router, Expo Go and the EAS cloud services. An Expo app is a React Native app, so anything RN can do, Expo can do through development builds, config plugins and custom modules. The React Native docs recommend starting with a framework like Expo.
*Follow-up:* Is Expo only for simple apps? No. Since development builds and config plugins exist, you can add any native code while keeping Expo's tooling.

**Q2. Expo Go vs a development build: what is the difference, and when must you switch?**
Expo Go is one prebuilt store app with a fixed set of native modules for a single SDK; you load your JS into it. A development build is your own binary with `expo-dev-client` and exactly your native dependencies and config. Switch when you add native code Expo Go lacks, change config plugins, icon or permissions, need your own scheme or universal links, or test remote push on Android (gone from Expo Go in SDK 53).
*Follow-up:* When do you rebuild the dev client? Only when native code or native config changes; JS changes reload from Metro.

**Q3. What is Continuous Native Generation, and what does `npx expo prebuild` do?**
CNG treats the `ios/` and `android/` projects as generated output. Prebuild copies a native template for your SDK, applies the app config, runs config plugins and autolinks native modules. Because the projects can be regenerated at any time, upgrades become "bump versions and regenerate" instead of merging Xcode and Gradle diffs.
*Follow-up:* What if you need a native change? Express it in the app config or a config plugin so it survives regeneration.

**Q4. Should you commit the ios and android folders?**
With CNG, no: gitignore them and let EAS Build run prebuild, so the config stays the single source of truth. Commit them only if you must maintain native changes that config plugins cannot express; then app config changes no longer apply automatically and you own native upgrades.
*Follow-up:* What happens on EAS if the folders exist? EAS builds them as they are and skips prebuild.

**Q5. What is a config plugin? Give an example.**
A function that receives the Expo config and registers mods that edit native files during prebuild: Info.plist keys, AndroidManifest entries, entitlements, Gradle settings. Libraries ship them (`expo-camera` adds the camera usage string from its options), and you can write your own with `withInfoPlist` or `withAndroidManifest`, for example to add a vendor SDK key on both platforms.
*Follow-up:* What makes a good plugin? Idempotent, typed mods instead of raw file edits, verified with `prebuild --clean`.

**Q6. app.json vs app.config.ts?**
`app.json` is static; `app.config.ts` is code that returns the config, so it can read environment variables and change the name, bundle ID, scheme or values per environment. If both exist, `app.config.ts` receives the JSON contents and can override them. Keep it deterministic because builds, updates and tools all evaluate it.
*Follow-up:* Typical use? App variants with different bundle IDs so dev, preview and production install side by side.

**Q7. How do environment variables work in an Expo app, and what is safe to expose?**
Variables prefixed with `EXPO_PUBLIC_` are inlined into the JS bundle at build time when accessed statically (`process.env.EXPO_PUBLIC_API_URL`); others stay on the Node side for config and tooling. Anything inlined is public, so only expose public values: API base URL, Sentry DSN, publishable payment keys. EAS environment variables provide per-environment values for builds and updates.
*Follow-up:* Why is `process.env[key]` undefined? Inlining replaces only exact static references.

**Q8. How do you keep a secret API key out of the app?**
You do not ship it at all. The mobile app calls your Rails API with the user's token, and the server calls the third-party service with the secret stored in server credentials or environment variables. Obfuscation in the bundle only slows attackers down.
*Follow-up:* What about keys that must be in the app, like a maps SDK key? Use keys designed to be public and restrict them in the vendor console (by bundle ID, package name or signing certificate).

**Q9. How do you upgrade an Expo SDK safely?**
Read the release notes, upgrade one SDK at a time, bump `expo`, run `npx expo install --fix` and `npx expo-doctor`, update other native libraries, regenerate the native projects, rebuild development clients, then test both platforms in release builds. Because native code changed, ship new store builds; the runtime version keeps old binaries on compatible OTA updates.
*Follow-up:* Why not skip SDKs? Breaking changes pile up and become hard to isolate; Play's yearly target API requirement also forces you to keep moving.

**Q10. Why `npx expo install` instead of `npm install`?**
It installs the version of each Expo-managed package that is compatible with your SDK and its React Native version. A plain `npm install` grabs the newest release, which may need a newer SDK and fail at build time or at runtime with native mismatches.
*Follow-up:* How do you repair existing mismatches? `npx expo install --fix`, or `--check` to list them.

**Q11. What did managed, bare and eject mean, and what replaced them?**
Managed meant no native folders and builds through Expo's classic service; bare meant committed, hand-edited native folders; ejecting converted managed to bare permanently. Today CNG generates native folders on demand, config plugins cover native customization, `expo build` became EAS Build and `expo publish` became EAS Update. Committing native folders is still possible when needed.
*Follow-up:* Why does it matter in interviews? Using the old words suggests outdated knowledge; explaining the change shows you follow the ecosystem.

**Q12. When would you write a custom native module, and how?**
Only after checking the Expo SDK, maintained community libraries and config plugins. If nothing fits (a vendor SDK or a platform API without a wrapper), write a local module with the Expo Modules API: `npx create-expo-module@latest --local`, implement it in Swift and Kotlin with the DSL (`Name`, `Function`, `AsyncFunction`, `Events`), and call it from TypeScript.
*Follow-up:* Can you test it in Expo Go? No, custom native code always needs a development build.

**Q13. How do you run development, preview and production versions on the same phone?**
Use `app.config.ts` with a variant variable (for example `APP_VARIANT`, set per EAS build profile) to change the bundle identifier, package name, app name and scheme. Different identifiers mean separate apps, each pointing at its own API environment.
*Follow-up:* What else should differ per variant? The API URL, push credentials or channels, the EAS Update channel and the app icon, so testers can tell builds apart.

**Q14. A teammate adds a library, and the app crashes on launch in Expo Go with "Cannot find native module". What happened?**
The library contains native code that Expo Go does not bundle, so the JS asks for a native module that does not exist. Create a development build that includes the library (`eas build --profile development` or `npx expo run:android`) and use it from then on.
*Follow-up:* How do you check beforehand? The library's docs or React Native Directory say whether it works in Expo Go and with the New Architecture.

**Q15. What does EAS give you over building on your own machine?**
Reproducible cloud builds on managed macOS and Linux machines (no local Xcode needed for iOS), credential management for certificates, profiles and keystores, automated store submission, OTA updates with channels, branches and runtime versions, and CI-friendly commands. Local builds (`eas build --local` or `npx expo run:*`) remain useful for debugging.
*Follow-up:* What is the trade-off? Build queue times on the free tier and a dependency on Expo's services.

**Q16. Your app shows a blank or white screen for a moment at launch. How do you handle startup in Expo?**
Keep the native splash screen visible until the app is ready: call `SplashScreen.preventAutoHideAsync()` at module scope, load fonts and restore the session, then call `SplashScreen.hideAsync()`. Keep that work short (embed fonts at build time with the `expo-font` config plugin where possible) and render cached data instead of waiting on the network.
*Follow-up:* What if the session check needs the network? Show the app shell from cached state and refresh in the background, with a timeout so the splash never hangs.

## Further reading
- [Expo documentation](https://docs.expo.dev/)
- [Development builds](https://docs.expo.dev/develop/development-builds/introduction/)
- [Continuous Native Generation](https://docs.expo.dev/workflow/continuous-native-generation/)
- [Config plugins](https://docs.expo.dev/config-plugins/introduction/)
- [Environment variables in Expo](https://docs.expo.dev/guides/environment-variables/)
