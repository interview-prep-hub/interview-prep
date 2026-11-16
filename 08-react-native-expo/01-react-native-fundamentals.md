# React Native Fundamentals

> **Tier:** T1 (know cold) · **Related:** [Expo fundamentals](02-expo-fundamentals.md), [Navigation and Expo Router](03-navigation-expo-router.md), [Performance and animations](07-performance-and-animations.md), [React fundamentals](../06-react/01-fundamentals.md), [Hooks](../06-react/02-hooks.md), [Event loop](../03-languages/javascript/04-async-event-loop-promises.md)
> **On your resume:** React Native (Expo) with TypeScript at Devntech: features across the Coloc resident app for Colochousing (events, tasks, split bills, rent payments, push notifications), live on the App Store and Google Play for 400+ residents.

## Quick revision
- React Native (RN) runs your React code in a JS engine and renders **real native views** (`UIView` on iOS, `android.view.View` on Android). It is not HTML in a WebView and not compiled to Swift or Kotlin.
- Threads: the **JS thread** runs React and your logic; the **UI (main) thread** draws native views and handles touches; background threads run layout (Yoga), image decoding and native work.
- Blocked JS thread: taps and JS-driven animations stall, native scrolling continues. Blocked UI thread: the whole app freezes (an ANR dialog on Android).
- **Old architecture:** an asynchronous, batched, JSON-serialized **bridge**. Costs: serialization, no synchronous calls, every native module initialized at startup, no type contract.
- **New Architecture:** **JSI** (JS calls C++ directly), **Fabric** (C++ renderer, synchronous layout, concurrent React), **TurboModules** (lazy, typed native modules), **Codegen** (typed specs to native glue), **bridgeless** mode.
- Default from RN 0.76 (Oct 2024). Expo SDK 52 enabled it for new projects, SDK 53 for all; SDK 54 was the last SDK with the legacy option; from RN 0.82 it is the only architecture.
- **Hermes** is the default engine (since RN 0.70): JS is compiled to bytecode at build time, so startup is faster and memory use lower.
- Core components: `View`, `Text`, `Image`, `ScrollView`, `FlatList`/`SectionList`, `Pressable`, `TextInput`, `Modal`, `ActivityIndicator`, `Switch`. Every string must be inside `<Text>`.
- `ScrollView` renders all children; `FlatList` virtualizes. Never nest a vertical `FlatList` in a vertical `ScrollView`.
- Styling: JS objects via `StyleSheet.create`, no selectors or cascade; text styles inherit only through nested `Text`; unitless density-independent pixels.
- Flexbox defaults differ from the web: `flexDirection: 'column'`, `alignItems: 'stretch'`, `flexShrink: 0`, and `flex` takes a single number.
- `useWindowDimensions()` updates on rotation; `Dimensions.get()` is a one-off snapshot. `PixelRatio` converts dp to physical pixels.
- Platform code: `Platform.OS`, `Platform.select`, `Platform.Version`, or `File.ios.tsx` / `File.android.tsx`.
- Safe areas: `react-native-safe-area-context` (`SafeAreaProvider`, `SafeAreaView` with `edges`, `useSafeAreaInsets`). The core `SafeAreaView` is iOS-only and deprecated (RN 0.81).
- Keyboard: `KeyboardAvoidingView` (`padding` on iOS), `keyboardShouldPersistTaps="handled"` on scroll containers, `Keyboard.dismiss()`.
- Network images need explicit width and height; `expo-image` adds caching and placeholders.
- Debugging: React Native DevTools (default from RN 0.76, press `j` in the Expo CLI), Dev Menu, LogBox, `adb logcat`, the Xcode console. Flipper is gone.

## 1. What React Native is (and is not)

React Native lets you build mobile apps with React and TypeScript. You compose components exactly as on the web, but the host components are native primitives instead of DOM elements: `<View>` becomes a `UIView` on iOS and a `ViewGroup` on Android, `<Text>` becomes a native text view. The app looks, scrolls and feels native because it is made of native views.

It is **not** a WebView wrapper (that is Cordova, Capacitor or Ionic): there is no HTML, no CSS file and no browser engine drawing your screens. It is also **not** compiled to Swift or Kotlin: your JavaScript runs at runtime inside an embedded engine (Hermes) and drives the native views.

New apps usually start from a framework. The React Native docs recommend one, and Expo is the standard choice: tooling, a library of native modules, file-based routing and build services ([Expo fundamentals](02-expo-fundamentals.md)). The old `npx react-native init` template command was retired.

## 2. React Native vs Flutter vs native vs web

| | React Native | Flutter | Native (Swift / Kotlin) | Web, PWA, WebView shells |
|---|---|---|---|---|
| Language | TypeScript / JavaScript | Dart | Swift (SwiftUI, UIKit), Kotlin (Compose, Views) | HTML, CSS, JS |
| Rendering | Real platform views | Own engine paints every pixel | Platform views | Browser engine (DOM) |
| Look and feel | Native controls by default | Identical on every platform; native look is imitated | Fully native, new OS features first | Web-like unless heavily styled |
| Code sharing | iOS + Android (+ web via React Native Web) | iOS, Android, web, desktop | One platform per codebase | Anything with a browser |
| Performance | Near-native for typical apps; heavy JS work needs care | Smooth, predictable rendering | Best, full control | Weakest for rich interaction |
| Over-the-air updates | Yes for JS and assets (EAS Update) | Not officially (third-party tools exist) | No, every change is a store release | Instant |
| Team fit | Reuses React/TS skills and npm libraries | Needs Dart skills | Two specialist teams | Web team |

How to choose: React Native when you have React/TypeScript developers, need both platforms, want a native look and fast iteration with OTA updates. Flutter when you want pixel-identical custom UI everywhere and the team is happy with Dart. Fully native when the product lives on platform features (AR, advanced camera or audio, widgets, watch apps) or when the company already has iOS and Android teams. A web app or PWA when you need neither store presence nor deep device APIs.

## 3. How it works: threads and the old bridge

| Thread | What runs there | Symptom when blocked |
|---|---|---|
| JS thread | Your bundle in Hermes: React renders, state updates, event handlers, `fetch` callbacks | Taps respond late, JS-driven animations stutter; native scrolling still works |
| UI (main) thread | Creating and updating native views, touch handling, native animations, scrolling | The app freezes; on Android a long freeze triggers "App Not Responding" (ANR) |
| Background threads | Layout with Yoga (a C++ flexbox engine), image decoding, native module work, networking | Usually invisible; slow layout delays updates |

**The legacy bridge.** Before the New Architecture, JS and native talked through the *bridge*: an asynchronous, batched queue of JSON messages. React computed changes on the JS thread and sent messages such as "create view 12 with these props"; a shadow thread computed layout with Yoga; the main thread applied the result. Native modules (storage, camera) were called the same way and answered through callbacks or promises.

Problems with the bridge:
- Every call and event was serialized to JSON and back, which hurt big lists and high-frequency events such as scrolling and gestures.
- Everything was asynchronous: JS could not measure a view or call native code synchronously, so popovers and tooltips flickered while waiting for layout.
- All native modules were initialized at startup, including ones the user never touched.
- There was no type contract between JS and native: a wrong argument type crashed at runtime.

## 4. The New Architecture

| Piece | What it is | What it fixes |
|---|---|---|
| **JSI** (JavaScript Interface) | A C++ API that lets JS hold references to C++ "host objects" and call their methods directly, synchronously or asynchronously | No JSON serialization, no single message queue; engine-agnostic |
| **Fabric** | The renderer rewritten in C++ and shared by both platforms, with an immutable shadow tree | Synchronous layout reads, React 18+ concurrent features (transitions, Suspense, automatic batching), consistent cross-platform behavior |
| **TurboModules** | Native modules exposed through JSI and loaded lazily on first use | Faster startup, typed calls, synchronous methods where they make sense |
| **Codegen** | A build-time generator that reads typed TypeScript or Flow specs and writes the C++, Objective-C and Java glue | A compile-time contract between JS and native |
| **Bridgeless mode** | Running without the legacy bridge at all (the default with the New Architecture since RN 0.74); interop layers keep most older libraries working | Removes the old startup and messaging overhead |

Timeline to state carefully: default in **RN 0.76** (Oct 2024); Expo **SDK 52** turned it on for new projects and **SDK 53** for all projects; **SDK 54** was the last SDK that allowed opting out; from **RN 0.82** it is the only architecture. Libraries written for the old APIs run through interop layers, and a few unmaintained ones break, so check compatibility (React Native Directory lists it) before upgrading.

### Going deeper: the Fabric render pipeline
1. **Render:** React reconciles your components and Fabric builds a C++ shadow tree mirroring the host components (on the JS thread).
2. **Commit:** Yoga computes layout for the new tree, which is then promoted to the next tree to mount. Trees are immutable, so React can prepare a low-priority update while a high-priority one commits.
3. **Mount:** the difference between the old and new trees is applied to native views on the UI thread.

Practical effects: `useLayoutEffect` can measure a view and position a tooltip before the user sees a flicker, `startTransition` works as on the web, and libraries such as Reanimated and react-native-mmkv use JSI for fast synchronous access.

## 5. Hermes

Hermes is the JavaScript engine Meta built for React Native, the default since RN 0.70 and in Expo. Instead of shipping JS source and parsing it on every launch, the build compiles your bundle to **Hermes bytecode**, so the app starts faster, uses less memory and downloads smaller. Hermes also implements the debugging protocol behind React Native DevTools. JavaScriptCore was moved out of core into a community package in recent versions, and a next-generation Hermes has been offered as an opt-in in recent releases (check the release notes before claiming details).

```ts
// Quick runtime check, for example on a hidden debug screen
const usingHermes = 'HermesInternal' in globalThis;
```

## 6. Core components and their native counterparts

| Component | iOS | Android | Notes |
|---|---|---|---|
| `View` | `UIView` | `ViewGroup` | Layout box: flexbox, background, borders |
| `Text` | `UITextView` | `TextView` | The only place strings may appear; nested `Text` inherits styles |
| `Image` | `UIImageView` | `ImageView` | Network images need explicit width and height |
| `ScrollView` | `UIScrollView` | `ScrollView` | Renders all children up front |
| `TextInput` | `UITextField` | `EditText` | Controlled with `value` + `onChangeText` |
| `Switch` | `UISwitch` | switch widget | `value` + `onValueChange` |
| `ActivityIndicator` | `UIActivityIndicatorView` | `ProgressBar` | Spinner: `size="large"`, `color` |
| `Modal` | presented view controller | dialog window | Implement `onRequestClose` for the Android back button |
| `Pressable` | a view with touch handling | same | `onPress`, `onLongPress`, `hitSlop`, pressed styles, `android_ripple` |
| `FlatList`, `SectionList` | built on scroll views | same | Virtualized lists for long data |

```tsx
import { ActivityIndicator, Pressable, StyleSheet, Switch, Text } from 'react-native';

type TaskRowProps = {
  title: string;
  done: boolean;
  saving: boolean;
  onToggle: (done: boolean) => void;
  onOpen: () => void;
};

export function TaskRow({ title, done, saving, onToggle, onOpen }: TaskRowProps) {
  return (
    <Pressable
      onPress={onOpen}
      hitSlop={8} // bigger touch target without changing layout
      android_ripple={{ color: '#e5e7eb' }}
      accessibilityRole="button"
      style={({ pressed }) => [styles.row, pressed && styles.pressed]}
    >
      <Text style={[styles.title, done && styles.done]} numberOfLines={1}>
        {title}
      </Text>
      {saving ? <ActivityIndicator /> : <Switch value={done} onValueChange={onToggle} />}
    </Pressable>
  );
}

const styles = StyleSheet.create({
  row: { flexDirection: 'row', alignItems: 'center', gap: 12, padding: 16 },
  pressed: { opacity: 0.7 },
  title: { flex: 1, fontSize: 16 },
  done: { textDecorationLine: 'line-through', color: '#6b7280' },
});
```

`Pressable` is the recommended touch primitive. `TouchableOpacity` still works and is common in older code; `Button` renders the platform button with almost no styling options.

## 7. Lists: ScrollView vs FlatList vs SectionList

| | `ScrollView` | `FlatList` | `SectionList` |
|---|---|---|---|
| Rendering | Every child, immediately | Only items near the viewport (virtualized) | Virtualized, grouped under section headers |
| Use for | Short, bounded content: forms, settings, a detail screen | Long or unbounded data: feeds, tasks, transactions | Grouped data: bills by month, events by day |
| Extras | `keyboardShouldPersistTaps`, `refreshControl` | `keyExtractor`, `onEndReached`, pull-to-refresh, `ListEmptyComponent` | `renderSectionHeader`, sticky headers |

```tsx
<FlatList
  data={events}
  keyExtractor={(event) => String(event.id)}
  renderItem={({ item }) => <EventCard event={item} />}
  ListEmptyComponent={<Text>No upcoming events</Text>}
  onEndReached={() => hasNextPage && fetchNextPage()}
  refreshing={isRefetching}
  onRefresh={refetch}
/>
```

Virtualization keeps memory flat: items far from the viewport are unmounted and replaced with blank space of the right size. A `FlatList` inside a `ScrollView` with the same orientation loses that (RN warns about it); use `ListHeaderComponent` and `ListFooterComponent` instead. Tuning (`getItemLayout`, `windowSize`, FlashList) lives in [Performance and animations](07-performance-and-animations.md).

## 8. Styling with StyleSheet

Styles are plain objects with camelCase keys. `StyleSheet.create` validates them in development and gives them names; you combine styles with arrays, falsy entries are ignored and later entries win.

```tsx
<View style={[styles.card, isOverdue && styles.overdue, { marginTop: index === 0 ? 0 : 8 }]} />
```

| | Web CSS | React Native |
|---|---|---|
| Where styles live | Stylesheets, selectors, the cascade | Objects on each element's `style` prop; no selectors, no cascade |
| Inheritance | Many properties inherit | Only text styles, and only from a parent `Text` |
| Units | px, rem, %, vh | Unitless density-independent pixels; percentages for some size props |
| Layout model | Block layout by default | Every view is a flex container |
| `flexDirection` default | `row` | `column` |
| `alignItems` default | `normal` (behaves like stretch) | `stretch`: children fill the width of a column |
| `flexShrink` default | 1 | 0 |
| `flex` | `grow shrink basis` shorthand | One number (`flex: 1` fills the available space) |
| Shadows | `box-shadow` | iOS `shadow*` props, Android `elevation`; `boxShadow` from RN 0.76 (New Architecture) |
| Hover, media queries | Yes | No: use pressed state, `useWindowDimensions`, `Platform` |

`gap`, `rowGap` and `columnGap` work (RN 0.71+). For utility classes, NativeWind brings Tailwind to RN (NativeWind v4 targets Tailwind v3); see [Tailwind CSS](../04-web-fundamentals/03-tailwind-css.md).

## 9. Screen size and density

```tsx
import { Dimensions, PixelRatio, StyleSheet, useWindowDimensions } from 'react-native';

export function useGridColumns() {
  const { width } = useWindowDimensions(); // re-renders on rotation, split screen and font scale changes
  return width >= 768 ? 3 : 2;
}

const snapshot = Dimensions.get('window'); // read once: stale after rotation unless you subscribe
const thumbPx = PixelRatio.getPixelSizeForLayoutSize(96); // 96 dp in physical pixels (288 on a 3x screen)
const hairline = StyleSheet.hairlineWidth; // thinnest visible line on this screen
```

- `window` is the area your app can use; `screen` is the whole display.
- `PixelRatio.get()` is 2 or 3 on most phones. Use it to request correctly sized images from the server, not to scale layouts.
- `fontScale` (also returned by `useWindowDimensions`) reflects the user's accessibility text size. Let layouts grow; cap with `maxFontSizeMultiplier` only where text truly cannot wrap.

## 10. Platform-specific code

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  card: {
    borderRadius: 12,
    backgroundColor: '#fff',
    ...Platform.select({
      ios: { shadowColor: '#000', shadowOpacity: 0.1, shadowRadius: 8, shadowOffset: { width: 0, height: 2 } },
      android: { elevation: 3 },
      default: {},
    }),
  },
});

if (Platform.OS === 'android' && Platform.Version >= 33) {
  // Android 13+: notifications need a runtime permission
}
```

- `Platform.OS` is `'ios'`, `'android'`, `'web'` and so on. `Platform.Version` is the Android API level (a number) or the iOS version (a string).
- For bigger differences split files: `DatePicker.ios.tsx` and `DatePicker.android.tsx`, both imported as `./DatePicker`. Metro picks the file per platform; `.native.tsx` vs `.web.tsx` works the same way. For TypeScript, add a shared `DatePicker.d.ts` (or configure `moduleSuffixes`, TS 4.7+) so `tsc` can resolve the import.
- Keep the props identical across platform files, otherwise the versions drift apart.

## 11. Safe areas

Phones have notches, the Dynamic Island, rounded corners, a home indicator and Android system bars. Android apps are drawn **edge-to-edge** as well: Expo SDK 53 turned it on for new projects and recent Android versions enforce it, so insets matter on both platforms.

```tsx
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';

export function BillsScreen() {
  return (
    // The tab bar already pads the bottom inset, so only pad the other edges here
    <SafeAreaView style={{ flex: 1 }} edges={['top', 'left', 'right']}>
      <BillList />
    </SafeAreaView>
  );
}

export function FloatingPayButton() {
  const insets = useSafeAreaInsets(); // { top, right, bottom, left } in dp
  return <PayButton style={{ position: 'absolute', right: 16, bottom: insets.bottom + 16 }} />;
}
```

- Use `react-native-safe-area-context` under a `SafeAreaProvider` at the root. Expo Router sets the provider up for you; with plain React Navigation you add it yourself (check your version's docs).
- The core `SafeAreaView` works only on iOS and is deprecated (RN 0.81).
- Navigator headers and tab bars already apply insets. Adding your own padding too gives double spacing.

## 12. Keyboard handling

```tsx
import { KeyboardAvoidingView, Platform, ScrollView, TextInput } from 'react-native';
import { useHeaderHeight } from '@react-navigation/elements';

export function NewBillForm() {
  const headerHeight = useHeaderHeight();
  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      keyboardVerticalOffset={headerHeight} // the navigation header sits above this view
    >
      <ScrollView keyboardShouldPersistTaps="handled" keyboardDismissMode="on-drag" contentContainerStyle={{ padding: 16 }}>
        <TextInput placeholder="What was it for?" returnKeyType="next" />
        <TextInput placeholder="Amount" keyboardType="decimal-pad" />
        {/* The Save button here gets the first tap even while the keyboard is open */}
      </ScrollView>
    </KeyboardAvoidingView>
  );
}
```

- `KeyboardAvoidingView` with `padding` on iOS. On Android the window usually resizes itself (`softwareKeyboardLayoutMode` in the app config), so `behavior` is often left unset; edge-to-edge changes this, which is why many Expo apps adopt `react-native-keyboard-controller` (Expo's keyboard handling guide covers it).
- `keyboardShouldPersistTaps`: the default `'never'` dismisses the keyboard on a tap outside the input and swallows that tap, so Submit needs two taps. `'handled'` lets child touchables receive it. `FlatList` accepts the same prop.
- Dismiss with `Keyboard.dismiss()` or `keyboardDismissMode="on-drag"`.
- Chain inputs with `returnKeyType="next"` and `onSubmitEditing={() => amountRef.current?.focus()}`. Help autofill with `autoComplete` and `textContentType`, and pick the right `keyboardType`.

## 13. Images and assets

- **Static assets:** `require('./assets/logo.png')`. Metro bundles the file and knows its size; add `logo@2x.png` and `logo@3x.png` and the right density is picked automatically.
- **Network images:** `{ uri: 'https://...' }` has no intrinsic size, so give width and height (or `aspectRatio`). Core `Image` caching is limited and platform-dependent; `expo-image` adds memory and disk caching, blurhash placeholders, transitions and `contentFit`.
- Memory follows decoded pixels (width x height x 4 bytes), not file size. Request thumbnails sized with `PixelRatio`, not 4000 px originals.
- SVG needs `react-native-svg`; icons usually come from `@expo/vector-icons`; fonts load with `expo-font`; the app icon and splash screen are set in the app config.

## 14. What transfers from React web, and what differs

| Transfers directly | Differs |
|---|---|
| Components, props, state, hooks, context, composition | No DOM: no `div`, `document`, `localStorage`, CSS files or class names |
| TanStack Query, Zustand, React Hook Form, Zod, date libraries | Primitives: `View`, `Text`, `Pressable`; `onPress` instead of `onClick`; all text inside `Text` |
| TypeScript, Testing Library ideas | Navigation is stacks and tabs with native transitions, plus the Android back button |
| Calling a REST API with `fetch` | Gestures, keyboard, safe areas and permissions are your job |
| Thinking about re-renders | Two threads and tight frame budgets; long lists need virtualization |
| | App lifecycle (background, killed, cold start), flaky networks, store releases, OTA updates |

## 15. Debugging

- **React Native DevTools** is the default debugger from RN 0.76: press `j` in the terminal running `npx expo start`, or open it from the Dev Menu. You get a console, breakpoints in your TypeScript (via source maps), memory snapshots, and the React Components and Profiler tabs; recent versions add network and performance panels (check what your version ships).
- **Dev Menu:** shake the device, `Cmd+D` in the iOS Simulator, `Cmd+M` or `Ctrl+M` in the Android emulator, or press `m` in the terminal.
- **LogBox** shows warnings and errors in development. Fix them instead of hiding them with `LogBox.ignoreLogs`.
- **Native logs:** `adb logcat "*:S" ReactNative:V ReactNativeJS:V` for Android; the Xcode console or the macOS Console app for iOS.
- **Gone:** Flipper (removed from the template in RN 0.74) and the old "Debug JS remotely" mode that ran your JS inside Chrome.
- **Release-only bugs:** development builds are slower and more forgiving. Reproduce in a release build (`npx expo run:ios --configuration Release`, `npx expo run:android --variant release`, or an EAS preview build). Clear Metro's cache with `npx expo start --clear` when the bundle behaves strangely.
- Crash reporting in production: [Native modules, permissions and testing](09-native-modules-permissions-testing.md).

## Common pitfalls
- `{items.length && <List />}` renders `0` as a raw string when the list is empty, which crashes with "Text strings must be rendered within a <Text> component". Use `items.length > 0 && ...` or a ternary.
- Putting a vertical `FlatList` inside a vertical `ScrollView`: virtualization is lost and RN warns.
- Reading `Dimensions.get('window')` once at module load and never updating it after rotation or split screen.
- Network images without width and height render at 0 x 0.
- Expecting CSS inheritance: `fontFamily` on a `View` does nothing. Create a shared `AppText` component instead.
- Using the core `SafeAreaView` and expecting Android insets, or adding insets twice under a navigator header.
- Forgetting `keyboardShouldPersistTaps="handled"`, so the first tap on Save only closes the keyboard.
- Heavy synchronous work (parsing big JSON, sorting thousands of rows) in a press handler or during an animation.
- Judging performance in development mode, or testing only on a recent iPhone instead of a low-end Android phone.

## How to talk about this in an interview
- Lead with the mental model: "React Native renders real native views. My components run on the JS thread in Hermes, and on the New Architecture they reach native code through JSI, Fabric and TurboModules instead of the old JSON bridge." Then give one practical consequence: keep the JS thread free during animations, virtualize long lists.
- Tie it to the Coloc resident app: React Native, Expo and TypeScript, live on the App Store and Google Play for 400+ residents, covering events, tasks, split bills, rent payments and push notifications against a Rails 7 REST API.
- Be honest about depth: at 1-3 years, interviewers want a clear explanation of threads and architecture plus evidence that you have debugged real platform differences, not C++ work inside Fabric.

> **Fill in:** which Expo SDK and React Native version the resident app runs, and whether it is on the New Architecture.

> **Fill in:** one platform-specific bug you fixed (keyboard covering an input, safe areas on a notched or edge-to-edge device, Android-only styling) and how you found it.

> **Fill in:** which devices you test on (simulators, your own phone, a low-end Android) and your debugging routine (DevTools, logs, release builds).

## Interview questions

**Q1. What is React Native, and how is it different from a WebView-based app?**
React Native runs your React/TypeScript code in a JS engine on the device and renders real native views through host components such as `View` and `Text`. A WebView app (Cordova, Capacitor, Ionic) renders HTML and CSS inside an embedded browser. RN therefore gets native scrolling, gestures, accessibility and look, while still sharing one codebase between iOS and Android.
*Follow-up:* Is RN compiled to Swift or Kotlin? No. Hermes runs your JS (compiled to bytecode at build time), and the JS drives native views.

**Q2. How does React Native render native UI? Walk me through the threads.**
Your bundle runs on the JS thread, where React renders components into a tree of host components. On the New Architecture, Fabric turns that into a C++ shadow tree, Yoga computes layout, and the changes are mounted as native views on the UI thread, which also handles touches and native animations. Native features are called through TurboModules over JSI. Blocking the JS thread delays taps and JS animations; blocking the UI thread freezes the app.
*Follow-up:* Where does layout happen? In Yoga, the C++ flexbox engine, off the main thread; only mounting happens on the main thread.

**Q3. What was the bridge, and why was it replaced?**
An asynchronous, batched channel that serialized every message between JS and native to JSON. It added serialization cost, forced everything to be async (no synchronous measurement or calls), initialized every native module at startup and had no type contract. Gestures, animations and large lists suffered most.
*Follow-up:* Do old libraries still work? Mostly, through interop layers; unmaintained ones can break, so check compatibility before upgrading.

**Q4. Explain JSI, Fabric, TurboModules and Codegen.**
JSI is a C++ interface that lets JS call native objects directly, sync or async, without JSON. Fabric is the C++ renderer with an immutable shadow tree, enabling synchronous layout reads and React's concurrent features. TurboModules are native modules exposed through JSI and loaded lazily. Codegen generates the native glue from typed specs at build time, so mismatches fail the build instead of crashing users. Bridgeless mode removes the old bridge entirely.
*Follow-up:* Since when is it the default? RN 0.76 (Oct 2024), Expo SDK 53 for all projects, and the only option from RN 0.82.

**Q5. What is Hermes and why does it matter?**
The JS engine built for React Native and the default since 0.70. It compiles JS to bytecode at build time, so the app skips parsing on launch: faster startup, lower memory, smaller downloads. It also powers debugging in React Native DevTools.
*Follow-up:* Trade-offs? Hermes historically had no JIT, so CPU-heavy JS can run slower than on a JIT engine; mobile apps are usually limited by startup time and memory instead.

**Q6. React Native vs Flutter vs native: how would you choose?**
React Native: a React/TS team, both platforms, native look, OTA updates, the npm ecosystem. Flutter: Dart and its own rendering engine for pixel-identical custom UI and consistent performance, but non-native widgets and no official OTA. Native: the best performance and day-one platform APIs, at the cost of two codebases and teams. For a community app such as a resident app backed by a Rails API, React Native with Expo is a strong fit.
*Follow-up:* When would you avoid RN? Heavy 3D or AR, demanding real-time media, or a company that already runs strong native teams.

**Q7. FlatList vs ScrollView: when do you use each?**
`ScrollView` renders all children at once, which is fine for short, bounded content such as a form. `FlatList` virtualizes: it renders only items near the viewport and unmounts distant ones, so memory and render time stay flat for long lists, and it adds `keyExtractor`, pagination with `onEndReached` and pull-to-refresh. `SectionList` is the grouped variant.
*Follow-up:* How do you make a long list fast? Stable keys, memoized rows, `getItemLayout` for fixed heights, tuned `windowSize` and `initialNumToRender`, or FlashList.

**Q8. How is styling in React Native different from CSS?**
Styles are JS objects with no selectors or cascade, and only text styles inherit, through nested `Text`. Everything is flexbox with `flexDirection: 'column'`, `flexShrink: 0` and `alignItems: 'stretch'` by default, and `flex` takes a single number. Units are unitless dp. There is no hover or media query; you use `useWindowDimensions`, `Platform` and pressed states.
*Follow-up:* Why does my text's background span the whole width? The parent column stretches its children; set `alignSelf: 'flex-start'` on the child.

**Q9. How do you write platform-specific code?**
`Platform.OS` checks for small differences, `Platform.select({ ios, android, default })` for values and styles, `Platform.Version` for OS-version checks, and `.ios.tsx` / `.android.tsx` files for whole components, which Metro resolves automatically. Keep the props identical across platform files.
*Follow-up:* How does TypeScript resolve `./DatePicker` when only platform files exist? Add a shared `.d.ts` or configure `moduleSuffixes`.

**Q10. How do you handle notches, home indicators and Android edge-to-edge?**
With `react-native-safe-area-context`: a `SafeAreaProvider` at the root, `SafeAreaView` with explicit `edges`, or `useSafeAreaInsets()` for custom positioning such as floating buttons. Navigators pad their own headers and tab bars, so avoid double padding. Android draws edge-to-edge in recent versions (Expo SDK 53+ for new projects), so test insets on Android too.
*Follow-up:* Why not the core `SafeAreaView`? It is iOS-only and deprecated in recent RN versions.

**Q11. The keyboard covers the input at the bottom of a form. How do you fix it?**
Wrap the form in `KeyboardAvoidingView` (`behavior="padding"` on iOS, `keyboardVerticalOffset` equal to the header height) with a `ScrollView` inside so the field can scroll into view. Set `keyboardShouldPersistTaps="handled"` so buttons respond to the first tap. Test Android separately, because window resizing and edge-to-edge change the behavior; `react-native-keyboard-controller` makes it consistent.
*Follow-up:* Why does Submit need two taps? The default `keyboardShouldPersistTaps="never"` spends the first tap dismissing the keyboard.

**Q12. Dimensions vs useWindowDimensions, and what is PixelRatio for?**
`Dimensions.get('window')` is a snapshot; `useWindowDimensions()` re-renders on rotation, split screen and font scale changes, so prefer it inside components. `PixelRatio` converts dp to physical pixels on 2x and 3x screens, which helps you request correctly sized images; `StyleSheet.hairlineWidth` gives the thinnest visible line.
*Follow-up:* What is `fontScale`? The user's accessibility text size multiplier; layouts must survive large values.

**Q13. What happens when the JS thread is busy?**
React cannot handle events or re-render, so taps feel ignored and JS-driven animations drop frames. Native scrolling, native-driver animations and Reanimated animations on the UI thread keep running. Fix it by removing work from the hot path: paginate, memoize, defer non-urgent work, or move heavy computation to the server.
*Follow-up:* How do you see it? The Perf Monitor in the Dev Menu shows JS and UI frame rates separately.

**Q14. How do you debug a React Native app during development?**
React Native DevTools (press `j` in the Expo CLI) for the console, breakpoints, component tree and profiler; LogBox for warnings; `adb logcat` and the Xcode console for native logs; a release or preview build for release-only bugs; `npx expo start --clear` for Metro cache problems. Flipper is no longer part of the toolchain.
*Follow-up:* How do you debug a failing API call? Log the request and response in the API client (or use the DevTools network panel where available) and compare with the Rails log for the same request.

**Q15. Coming from React on the web, what transfers and what is new?**
Components, hooks, context, state libraries, TanStack Query, forms and testing ideas transfer. New: native primitives instead of the DOM, no CSS cascade, navigation stacks and the Android back button, gestures, keyboard and safe areas, permissions, the app lifecycle, flaky networks, store releases and OTA updates.
*Follow-up:* What surprised you most? Answer with a real example from your own work rather than a generic one.

**Q16. Spot the bug: `return <View>{tasks.length && <TaskList tasks={tasks} />}</View>;`**
When `tasks` is empty, `tasks.length` is `0`, React renders `0` as a text node directly inside a `View`, and the app crashes with "Text strings must be rendered within a <Text> component". On the web it would just print 0. Use `tasks.length > 0 && ...` or a ternary.
*Follow-up:* Other raw-text crashes? A stray space or string between JSX tags inside a `View`, such as `<View> {child}</View>` written on one line.

**Q17. Pressable, TouchableOpacity or Button: which do you use?**
`Pressable` is the modern primitive: pressed-state styling, `hitSlop`, long press and Android ripple, all customizable. `TouchableOpacity` is older but still common. `Button` renders a platform button with minimal styling, fine for prototypes. Give icon-only buttons an `accessibilityLabel` and an `accessibilityRole`.
*Follow-up:* What is `hitSlop`? Extra touch area around a small target that does not change layout.

## Further reading
- [Core components and native components](https://reactnative.dev/docs/intro-react-native-components)
- [React Native architecture overview](https://reactnative.dev/architecture/landing-page)
- [Platform-specific code](https://reactnative.dev/docs/platform-specific-code)
- [React Native DevTools](https://reactnative.dev/docs/react-native-devtools)
- [Expo: safe areas](https://docs.expo.dev/develop/user-interface/safe-areas/)
