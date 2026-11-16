# Performance and Animations

> **Tier:** T1 (know cold) · **Related:** [React Native fundamentals](01-react-native-fundamentals.md), [React performance](../06-react/06-performance.md), [Hooks](../06-react/02-hooks.md), [Navigation performance](03-navigation-expo-router.md), [Native modules and testing](09-native-modules-permissions-testing.md)
> **On your resume:** the Coloc resident app (React Native, Expo, TypeScript): lists of events, tasks, bills and payments that must stay smooth on residents' everyday phones.

## Quick revision
- Frame budget: about 16.7 ms per frame at 60 Hz (8.3 ms at 120 Hz). Miss it and the UI drops frames.
- What makes RN apps slow: long tasks on the **JS thread**, unnecessary **re-renders**, long lists rendered without virtualization, **oversized images**, heavy startup work, and (old architecture) chatty bridge traffic.
- **Measure first** in a release-like build: the Perf Monitor (JS vs UI FPS), the React Profiler in React Native DevTools, Android Studio and Xcode Instruments for native work.
- Re-renders: `memo` for list rows and heavy children, stable callbacks (`useCallback`), state colocation, split contexts, Zustand selectors. React Compiler automates much of the memoization in recent setups.
- FlatList: stable `keyExtractor`, memoized `renderItem` and row component, `getItemLayout` for fixed heights, tuned `initialNumToRender`, `maxToRenderPerBatch`, `windowSize`, optionally `removeClippedSubviews`; paginate with `onEndReached`.
- **FlashList** (Shopify) recycles views instead of mounting and unmounting them; rows must reset local state because instances are reused.
- Images: `expo-image` with memory and disk caching, server-side thumbnails sized for the screen, `recyclingKey` in recycled lists.
- Startup: Hermes bytecode, less work before first render, lazy imports, embedded fonts, cached data instead of network waits.
- **Animated** with `useNativeDriver: true` runs transform and opacity animations on the UI thread, but not layout props, and gestures still route through JS.
- **Reanimated:** shared values plus **worklets** that run on the UI thread (`useSharedValue`, `useAnimatedStyle`, `withTiming`, `withSpring`), so animations and gestures stay smooth even when JS is busy. **Gesture Handler** recognizes gestures natively.
- Deferring work: `InteractionManager.runAfterInteractions` (check its status in recent RN versions), `requestIdleCallback`, `startTransition`, skeletons first.
- Memory leaks: listeners, timers and subscriptions without cleanup; unbounded caches; huge images. Find them with heap snapshots and native profilers.
- Bundle size: analyze with Expo Atlas, avoid heavy libraries and full-library imports, compress assets; Android App Bundles deliver per-device APKs.

## 1. Mental model: where time goes

| Bottleneck | Typical cause | Symptom |
|---|---|---|
| JS thread | Big renders, JSON parsing, sorting or filtering thousands of items, synchronous loops in handlers | Taps respond late, JS animations stutter, the JS FPS counter drops |
| UI thread | Deep view hierarchies, huge images decoding, expensive native layout | Scrolling janks, the UI FPS counter drops; on Android an ANR in the worst case |
| Network | Waterfalls, no caching, oversized responses | Spinners, slow screens even on fast phones |
| Memory | Undisposed listeners, large bitmaps, unbounded arrays | Gradual slowdown, crashes on low-end devices (out of memory) |

Development builds are much slower than release builds (dev checks, no minification, debugging hooks). Judge performance only in a release or preview build, and on a mid or low-end Android phone, which is where real users notice problems first.

## 2. Measuring

- **Perf Monitor** (Dev Menu): separate JS and UI frame rates, so you know which thread to look at.
- **React Native DevTools:** the React Profiler shows which components rendered, how long each took and why; recent versions add performance tracing (check what your version offers).
- **Highlight updates** in the React DevTools component view reveals rows that re-render on every keystroke.
- **Native profilers:** Android Studio's profiler (CPU, memory, frames) and Xcode Instruments (Time Profiler, Allocations, Leaks) for anything below JS.
- **Production signals:** slow screens and app start times in a monitoring tool such as Sentry's performance features, plus Android vitals in Play Console.
- Measure, change one thing, measure again. Random memoization without a profile mostly adds noise.

## 3. Avoiding unnecessary re-renders

React re-renders a component when its state changes, its parent re-renders or a context it reads changes. That is usually cheap; it matters for list rows, big trees and anything during animations.

```tsx
import { memo, useCallback, useState } from 'react';
import { FlatList, Pressable, Text, type ListRenderItemInfo } from 'react-native';

type Task = { id: number; title: string; done: boolean };

const TaskRow = memo(function TaskRow({ task, onToggle }: { task: Task; onToggle: (id: number) => void }) {
  return (
    <Pressable onPress={() => onToggle(task.id)} style={{ height: 64, justifyContent: 'center', paddingHorizontal: 16 }}>
      <Text>{task.done ? 'Done: ' : ''}{task.title}</Text>
    </Pressable>
  );
});

export function TaskList({ tasks, toggleTask }: { tasks: Task[]; toggleTask: (id: number) => void }) {
  const [query, setQuery] = useState(''); // typing here re-renders TaskList, not every row
  const onToggle = useCallback((id: number) => toggleTask(id), [toggleTask]); // stable identity keeps memo working
  const renderItem = useCallback(
    ({ item }: ListRenderItemInfo<Task>) => <TaskRow task={item} onToggle={onToggle} />,
    [onToggle],
  );
  const visible = query ? tasks.filter((t) => t.title.toLowerCase().includes(query.toLowerCase())) : tasks;
  return <FlatList data={visible} keyExtractor={(t) => String(t.id)} renderItem={renderItem} />;
}
```

Techniques:
- **`memo` rows** and pass them primitive or stable props; an inline object or arrow function created per render defeats `memo`.
- **Colocate state:** keep the search input's state in the smallest component that needs it.
- **Split contexts** (session vs theme vs per-feature state) so one change does not re-render every consumer; or use **Zustand selectors** (`useStore((s) => s.billsFilter)`).
- **Derive, do not copy:** compute filtered lists during render (with `useMemo` when the computation is heavy) instead of syncing them into state with effects.
- **React Compiler** memoizes components and values automatically, which makes "wrap everything in `useMemo` and `useCallback`" advice outdated where it is enabled. Recent Expo SDKs can enable it through the app config, and newer templates may turn it on; check your SDK's docs before relying on it.

More background in [React performance](../06-react/06-performance.md).

## 4. Long lists: FlatList tuning and FlashList

```tsx
const ROW_HEIGHT = 64;

<FlatList
  data={tasks}
  keyExtractor={(t) => String(t.id)} // stable, unique; never the array index for mutable lists
  renderItem={renderItem} // memoized, renders a memo() row
  getItemLayout={(_, index) => ({ length: ROW_HEIGHT, offset: ROW_HEIGHT * index, index })} // fixed-height rows only
  initialNumToRender={12} // enough to fill the first screen
  maxToRenderPerBatch={10} // items rendered per batch while scrolling
  windowSize={7} // viewports kept rendered (default 21): lower saves memory, higher reduces blank areas
  removeClippedSubviews // detach off-screen views; measure, it has caused missing-content bugs
  onEndReached={() => hasNextPage && !isFetchingNextPage && fetchNextPage()}
  onEndReachedThreshold={0.5}
  ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null}
/>
```

| Prop | What it does | Default |
|---|---|---|
| `keyExtractor` | Identity for reconciliation and recycling | Uses `item.key`, then the index |
| `getItemLayout` | Skips measuring; enables instant `scrollToIndex` | Not set |
| `initialNumToRender` | Items in the first render | 10 |
| `maxToRenderPerBatch` | Items per incremental batch | 10 |
| `windowSize` | Rendered area in viewport heights | 21 |
| `updateCellsBatchingPeriod` | Delay between batches | 50 ms |
| `removeClippedSubviews` | Detaches off-screen native views | Platform-dependent |

Paginating with TanStack Query keeps the list small and loads more as the user scrolls:

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

type Page = { items: Bill[]; next_cursor: string | null };

export function useBillsInfinite(houseId: number) {
  return useInfiniteQuery({
    queryKey: ['bills', 'list', { houseId }],
    queryFn: async ({ pageParam }): Promise<Page> => {
      const cursor = pageParam ? `&cursor=${encodeURIComponent(pageParam)}` : '';
      const res = await api(`/api/v1/houses/${houseId}/bills?limit=30${cursor}`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.next_cursor, // null means no more pages
  });
}

// In the screen: data={query.data?.pages.flatMap((p) => p.items) ?? []}
// onEndReached={() => query.hasNextPage && !query.isFetchingNextPage && query.fetchNextPage()}
```

Other wins: paginate the API (cursor-based) instead of loading 2,000 bills; keep rows simple (fewer nested views, no heavy computations during render); use `SectionList` for grouped data; avoid `extraData` that changes on every render.

**FlashList** (`@shopify/flash-list`) is a drop-in style replacement that **recycles** row views instead of unmounting and mounting them, which is much faster for long, complex lists. Because a row instance is reused for different items, local state (an expanded flag) can leak between items: key it by item ID or reset it. `getItemType` helps heterogeneous lists recycle the right views. FlashList v1 required an `estimatedItemSize` prop; v2 (2025) targets the New Architecture and drops that requirement (check the version you install).

## 5. Images

- Use **`expo-image`**: memory and disk caching (`cachePolicy`), blurhash or thumbhash placeholders, smooth `transition`, `contentFit`, and `Image.prefetch(urls)` for the next screen.
- **Serve the right size.** A 4000 x 3000 photo decodes to roughly 48 MB of bitmap memory, whatever its file size. Request thumbnails sized with `PixelRatio.getPixelSizeForLayoutSize(width)`; Rails Active Storage variants or an image CDN can produce them.
- In recycled lists, pass `recyclingKey={item.id}` so a reused row does not briefly show the previous item's image.
- Prefer vector icons (`@expo/vector-icons`) or SVG for icons, and compress bundled assets.

## 6. Startup time

- **Hermes** precompiles JS to bytecode, so launch skips parsing.
- **Do less before the first frame:** keep the splash screen until fonts and the session are ready, but avoid network waits there; render cached data (a persisted query cache) and refresh in the background.
- **Lazy-load** heavy, rarely used modules (a PDF viewer, a chart library) with dynamic `import()` or by loading them on the screen that needs them; Expo Router screens load only when visited.
- **Embed fonts** at build time with the `expo-font` config plugin instead of loading them at runtime.
- Avoid importing giant JSON files or initializing every SDK (analytics, chat, maps) at launch; initialize them lazily.
- Measure cold start in release builds on a low-end Android device; that number is what users feel.

## 7. Animations: Animated, Reanimated and Gesture Handler

### Animated (built in)

```tsx
import { useEffect, useRef } from 'react';
import { Animated } from 'react-native';

export function FadeIn({ children }: { children: React.ReactNode }) {
  const opacity = useRef(new Animated.Value(0)).current;
  useEffect(() => {
    Animated.timing(opacity, { toValue: 1, duration: 250, useNativeDriver: true }).start();
  }, [opacity]);
  return <Animated.View style={{ opacity }}>{children}</Animated.View>;
}
```

`useNativeDriver: true` sends the whole animation description to native once, so it runs on the UI thread even if JS is busy. Limits: only non-layout properties (`transform`, `opacity`), not `width`, `height`, margins or flex; and anything driven by JS each frame (gesture tracking through PanResponder, chained logic) still depends on the JS thread. `Animated.event` with the native driver links scroll position to an animation without JS per frame:

```tsx
export function CollapsingHeaderList({ events }: { events: EventItem[] }) {
  const scrollY = useRef(new Animated.Value(0)).current;
  const headerOpacity = scrollY.interpolate({ inputRange: [0, 80], outputRange: [1, 0], extrapolate: 'clamp' });

  return (
    <>
      <Animated.View style={{ opacity: headerOpacity }}>
        <Text style={{ fontSize: 28, padding: 16 }}>Upcoming events</Text>
      </Animated.View>
      <Animated.FlatList
        data={events}
        keyExtractor={(e) => String(e.id)}
        renderItem={({ item }) => <EventCard event={item} />}
        onScroll={Animated.event([{ nativeEvent: { contentOffset: { y: scrollY } } }], { useNativeDriver: true })}
        scrollEventThrottle={16}
      />
    </>
  );
}
```

### Reanimated

```tsx
import Animated, { runOnJS, useAnimatedStyle, useSharedValue, withSpring } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

export function SwipeToComplete({ onComplete, children }: { onComplete: () => void; children: React.ReactNode }) {
  const translateX = useSharedValue(0); // lives on the UI thread, readable from JS

  const pan = Gesture.Pan()
    .activeOffsetX([-10, 10]) // let vertical scrolling win until the finger moves sideways
    .onUpdate((e) => {
      translateX.value = e.translationX; // a worklet: runs on the UI thread every frame
    })
    .onEnd(() => {
      if (translateX.value > 120) runOnJS(onComplete)(); // hop back to JS for app logic
      translateX.value = withSpring(0);
    });

  const style = useAnimatedStyle(() => ({ transform: [{ translateX: translateX.value }] }));

  return (
    <GestureDetector gesture={pan}>
      <Animated.View style={style}>{children}</Animated.View>
    </GestureDetector>
  );
}
```

- **Shared values** hold animation state on the UI thread; changing `.value` does not re-render React.
- **Worklets** are small JS functions that Reanimated runs on the UI thread (gesture callbacks, `useAnimatedStyle`), so the animation follows the finger at full frame rate regardless of JS load.
- Animation helpers: `withTiming`, `withSpring`, `withDecay`, `withSequence`, `withRepeat`, `withDelay`, and `interpolate` for mapping ranges. Layout animations (`entering={FadeIn}`, `exiting`, `layout={LinearTransition}`) animate mount, unmount and reordering.
- **Gesture Handler** runs gesture recognition natively and composes gestures (`Gesture.Simultaneous`, `Gesture.Exclusive`). Wrap the app root in `GestureHandlerRootView` (check whether your navigation setup already does).
- Versions: Reanimated 4 (2025) requires the New Architecture, moved worklets into the separate `react-native-worklets` package and added CSS-style animations and transitions; there `runOnJS` gives way to new scheduling helpers. The Babel plugin is configured by `babel-preset-expo` in recent SDKs. Check the docs for your version.

| | Animated | Reanimated |
|---|---|---|
| Where it runs | JS-driven, or UI thread with `useNativeDriver` | UI thread via worklets |
| Animatable props | Native driver: transform and opacity only | Any style prop, including layout |
| Gestures | PanResponder on the JS thread | Gesture Handler with UI-thread callbacks |
| Layout animations | Basic (`LayoutAnimation`) | Rich `entering`, `exiting`, `layout` animations |
| Setup | Built in | Library plus Babel plugin (automatic with Expo) |
| Use for | Simple fades and slides | Gesture-driven UI, complex or interruptible animations |

### Going deeper: what the New Architecture changes for performance
- JSI removes bridge serialization, so libraries such as Reanimated, MMKV and FlashList talk to native synchronously and cheaply.
- Fabric lets `useLayoutEffect` measure and adjust layout before paint, which removes the "render, measure, re-render" flicker for tooltips and popovers.
- Concurrent React features (`startTransition`, Suspense) work in React Native as on the web, so urgent updates such as typing can interrupt expensive re-renders.
- The work you schedule still matters: a 200 ms synchronous loop blocks the JS thread on any architecture.

## 8. Deferring work

- **Skeletons first:** render the screen shell immediately and fill data as it arrives.
- `InteractionManager.runAfterInteractions(fn)` runs work after current animations and touches finish; its behavior and status have changed in recent React Native versions, so check the docs and consider `requestIdleCallback` instead.
- `startTransition` marks updates (filtering a long list as the user types) as low priority so input stays responsive; `useDeferredValue` does the same for derived values.
- Break huge synchronous jobs into chunks, or move them to the server: a Rails endpoint that returns pre-computed balances beats summing 5,000 bills on a phone.

## 9. Memory leaks

```ts
useEffect(() => {
  const appStateSub = AppState.addEventListener('change', onAppStateChange);
  const netInfoUnsubscribe = NetInfo.addEventListener(onNetworkChange);
  const pollId = setInterval(refreshBadge, 60_000);
  return () => {
    appStateSub.remove();
    netInfoUnsubscribe();
    clearInterval(pollId);
  };
}, []);
```

Common leaks: listeners (AppState, Keyboard, Dimensions, NetInfo, notification listeners) and timers without cleanup; WebSocket or store subscriptions created outside React and never closed; module-level caches that grow forever; pushing the same screen onto a stack again and again; huge bitmaps kept alive by long lists. Detect them with heap snapshots in React Native DevTools (compare before and after repeating a flow), Android Studio's memory profiler and Xcode's Allocations and Leaks instruments. A memory chart that keeps climbing as you open and close the same screen is the classic sign.

## 10. Bundle and app size

- Analyze the JS bundle with **Expo Atlas** (available in recent SDKs) to see which packages dominate.
- Replace heavy libraries (moment with all locales, full lodash) with lighter ones (`date-fns`, per-function imports) and remove unused dependencies.
- Compress images and fonts; load large media from a CDN instead of bundling it.
- Android App Bundles (the default for Play Store builds) let Play deliver only the code and resources each device needs; R8 shrinking can be enabled through `expo-build-properties`.
- Hermes bytecode is compact, and EAS Update ships only JS and assets, so smaller bundles also mean faster OTA downloads.

## Common pitfalls
- Profiling in development mode and "optimizing" problems that do not exist in release builds.
- Inline arrow functions and style objects passed to memoized rows, so `memo` never helps.
- Array index as the key in lists that reorder or delete items, causing wrong rows to update.
- `getItemLayout` with a wrong height (forgetting separators), causing jumps and misplaced scrolling.
- A vertical `FlatList` nested in a `ScrollView`, which renders everything.
- Full-resolution photos as thumbnails.
- `useNativeDriver: true` on a `height` animation, which throws, or animating layout with `setState` every frame.
- Forgetting cleanup for listeners and timers; local state leaking between recycled FlashList rows.
- Doing heavy work in `useEffect` on mount while the navigation transition is still running.

## How to talk about this in an interview
- Show a method, not a list of tricks: "I check whether the JS or UI thread drops frames in a release build, profile which components render, fix the biggest cause, then measure again."
- Have one concrete list optimization story and one animation story ready, ideally from the resident app's lists of events, tasks, bills or payments.
- Explain Reanimated in one sentence: "Shared values and worklets keep animation and gesture logic on the UI thread, so they stay smooth even when JS is busy."

> **Fill in:** a performance problem you saw in the resident app (slow list, janky screen, slow startup), how you measured it and what fixed it.

> **Fill in:** which animation or gesture libraries the app uses (Animated, Reanimated, Gesture Handler) and for what.

> **Fill in:** the lowest-end device you test on and anything that only broke there.

## Interview questions

**Q1. What makes React Native apps slow?**
Long tasks on the JS thread (big renders, parsing, sorting), unnecessary re-renders, long lists without virtualization or with heavy rows, oversized images, heavy startup work, and on the old architecture chatty bridge traffic. UI thread problems come from deep view trees and image decoding. Identify which thread drops frames before fixing anything.
*Follow-up:* How do you tell which thread? The Perf Monitor shows JS and UI frame rates separately.

**Q2. How do you find a performance problem?**
Reproduce in a release or preview build on a low-end Android device, check the Perf Monitor, record with the React Profiler to find components that render too often or too slowly, and use Android Studio or Instruments for native costs. Fix the largest cause and measure again.
*Follow-up:* Why not measure in development? Dev mode adds checks and skips optimizations, so numbers are misleading.

**Q3. How do you optimize a long FlatList?**
Stable `keyExtractor`, a memoized `renderItem` rendering a `memo` row with stable props, `getItemLayout` when rows have a fixed height, tuned `initialNumToRender`, `maxToRenderPerBatch` and `windowSize`, lightweight rows, properly sized images, and API pagination with `onEndReached`. If it is still slow, switch to FlashList.
*Follow-up:* What does lowering `windowSize` trade? Less memory and work, but more blank areas during fast scrolling.

**Q4. What does getItemLayout do?**
It tells the list each item's length and offset up front, so it does not have to measure rendered items. That makes initial rendering and `scrollToIndex` fast and precise. It only works when item heights are known, including separators.
*Follow-up:* Variable heights? Skip it, or use FlashList, which handles measurement differently.

**Q5. FlatList vs FlashList?**
FlatList mounts and unmounts rows as they enter and leave the render window. FlashList recycles row views, rebinding them to new items, which is faster and smoother for long, complex lists. The catch: recycled rows keep local state, so reset or key state by item, and use `getItemType` for mixed row types.
*Follow-up:* When is FlatList enough? Short or medium lists with simple rows, which covers many screens.

**Q6. How do you avoid unnecessary re-renders?**
Memoize list rows and heavy children, keep callback and object props stable, colocate state near where it is used, split contexts or use store selectors, and derive data during render rather than syncing it into state. Where React Compiler is enabled, much of the manual memoization is done for you.
*Follow-up:* When is `useMemo` worth it? For expensive computations or to keep a prop stable for a memoized child, not for trivial values.

**Q7. How do you handle images efficiently?**
`expo-image` for caching, placeholders and transitions; thumbnails sized for the display (dp times pixel ratio) instead of originals; `recyclingKey` in recycled lists; prefetching images for the next screen; vector icons instead of bitmaps.
*Follow-up:* Why does image size matter beyond download time? Decoded bitmaps take width x height x 4 bytes of memory, which crashes low-end devices.

**Q8. How do you improve startup time?**
Rely on Hermes bytecode, do less before the first frame, show cached data instead of waiting on the network, lazy-load heavy modules and SDKs, embed fonts at build time and keep the splash screen short. Measure cold start in a release build on a slow device.
*Follow-up:* What usually dominates? Network calls and SDK initialization placed before the first render.

**Q9. Animated vs Reanimated?**
Animated is built in; with `useNativeDriver` it runs transform and opacity animations on the UI thread, but layout properties and gesture tracking still involve JS. Reanimated runs animation logic as worklets on the UI thread with shared values, animates any style, pairs with Gesture Handler for 60 fps gestures, and offers layout animations. Use Animated for simple fades, Reanimated for gesture-driven or complex animations.
*Follow-up:* What does Reanimated 4 change? It requires the New Architecture, splits worklets into `react-native-worklets` and adds CSS-style animations.

**Q10. What does useNativeDriver do, and what is its limitation?**
It serializes the animation to native once, so frames are computed on the UI thread without JS involvement, staying smooth even when JS is busy. It supports only non-layout props such as `transform` and `opacity`; animating `height` or `width` with it throws.
*Follow-up:* How do you animate height smoothly then? Reanimated, a `scaleY` transform, or layout animations.

**Q11. What are shared values and worklets?**
A shared value is a mutable container that lives on the UI thread and can be read and written from both threads; updating it does not re-render React. A worklet is a function marked to run on the UI thread, such as gesture callbacks and `useAnimatedStyle`. Together they keep per-frame logic off the JS thread.
*Follow-up:* How do you call app logic from a worklet? Schedule a JS-thread function (`runOnJS` in Reanimated 3; newer helpers in v4).

**Q12. How would you build a swipe-to-complete gesture on a task row?**
A `Gesture.Pan()` from Gesture Handler with `activeOffsetX` so vertical scrolling still works, updating a shared value in `onUpdate`, and in `onEnd` either springing back with `withSpring(0)` or completing when past a threshold (calling JS for the API mutation). `useAnimatedStyle` maps the shared value to `translateX`.
*Follow-up:* Why not PanResponder? It runs on the JS thread, so the row lags behind the finger when JS is busy.

**Q13. How do you defer non-urgent work so interactions stay smooth?**
Render a skeleton first, mark low-priority updates with `startTransition` or `useDeferredValue`, run work after interactions (`InteractionManager.runAfterInteractions` or `requestIdleCallback`, depending on your RN version), split big jobs into chunks, or move heavy computation to the server.
*Follow-up:* Example? Filtering a 1,000-item list as the user types inside `startTransition`.

**Q14. How do memory leaks happen in React Native, and how do you find them?**
Listeners, timers and subscriptions without cleanup, external subscriptions never closed, unbounded module-level caches, growing navigation stacks and large retained bitmaps. Find them by repeating a flow and comparing heap snapshots in React Native DevTools, and with Android Studio's memory profiler or Xcode's Allocations and Leaks instruments.
*Follow-up:* What does a leak look like in production? Out-of-memory crashes after long sessions, mostly on low-memory devices.

**Q15. How do you reduce bundle and app size?**
Analyze with Expo Atlas, remove or replace heavy dependencies, import functions instead of whole libraries, compress images and fonts, serve media from a CDN, enable R8 shrinking on Android through `expo-build-properties`, and let Play deliver per-device APKs from the App Bundle.
*Follow-up:* Why does JS size matter for OTA? Every EAS Update download includes the JS bundle, so smaller bundles update faster.

**Q16. The app feels slow in development but fine in production. Should you worry?**
Not necessarily: development builds run extra checks, are unminified and talk to Metro, so they are slower by design. Measure in a release or preview build. If production is also slow on low-end devices, profile there.
*Follow-up:* Is anything slower in production? Rarely, but missing source maps or console logs left in hot paths can hurt; strip logs in release builds.

**Q17. Spot the problem: `renderItem={({ item }) => <Row item={item} onPress={() => select(item.id)} style={{ padding: 8 }} />}` with `Row` wrapped in `memo`.**
Every parent render creates a new `onPress` function and a new `style` object for each row, so `memo`'s shallow comparison always fails and every visible row re-renders. Pass a stable callback (`onSelect={select}` from `useCallback`) and let the row call it with its ID, move the style into `StyleSheet.create`, and memoize `renderItem`.
*Follow-up:* How would you confirm the fix? Highlight updates or the React Profiler should show rows no longer rendering on unrelated changes.

**Q18. Users see blank areas when they scroll a long list quickly. What do you check?**
The list cannot render rows as fast as the user scrolls: rows are too heavy, `windowSize` or `maxToRenderPerBatch` are too low, images decode on the fly at full size, or the JS thread is busy with other work. Simplify rows, memoize them, raise the render window slightly, use correctly sized cached images, and consider FlashList, whose recycling handles fast scrolling better.
*Follow-up:* Is some blanking acceptable? Brief blanking during extreme flings is normal for virtualized lists; blanking during normal scrolling is not.

## Further reading
- [React Native performance overview](https://reactnative.dev/docs/performance)
- [Optimizing FlatList configuration](https://reactnative.dev/docs/optimizing-flatlist-configuration)
- [Reanimated documentation](https://docs.swmansion.com/react-native-reanimated/)
- [expo-image](https://docs.expo.dev/versions/latest/sdk/image/)
