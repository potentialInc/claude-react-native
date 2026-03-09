# Performance Debugger

Guide for diagnosing and fixing performance issues in React Native/Expo applications.

## When to Use

- App feels slow, janky, or laggy
- Frame drops during scrolling or animations
- High memory usage or memory leaks
- Slow app startup time
- When asked to "optimize", "improve performance", "fix lag", "reduce jank"

## Performance Diagnostics

### React DevTools Profiler

```bash
# Launch standalone React DevTools
npx react-devtools
```

**What to look for:**

- **Excessive re-renders:** Components highlighted frequently when data hasn't changed
- **Slow renders:** Any render taking >16ms blocks a frame (60fps target)
- **Flame graph:** Wide bars = slow components. Tall stacks = deep component trees re-rendering unnecessarily
- **"Why did this render?"** Enable in Profiler settings to see which props/state changed

### Flipper Performance Plugin

```bash
# Install Flipper from https://fbflipper.com/
# Connect to your running app (auto-detected for debug builds)
```

**Useful Flipper plugins:**

| Plugin | Use Case |
|--------|----------|
| Network | Identify slow API calls blocking renders |
| Layout Inspector | Find unnecessary nesting and layout thrashing |
| React DevTools | Component tree + profiler (embedded) |
| Databases | Check MMKV/SQLite query performance |

### Hermes Performance

Hermes is the default JS engine for React Native 0.70+ and Expo SDK 49+. It provides:

- **Bytecode precompilation:** JS is compiled to bytecode at build time, not runtime
- **Faster startup:** 30-50% improvement over JSC
- **Lower memory usage:** Optimized garbage collector

**Profiling with Hermes:**

```bash
# Record a Hermes CPU profile
# 1. In Dev Menu, select "Start/Stop Sampling Profiler"
# 2. Perform the slow action
# 3. Stop profiling — file saved to device

# Pull the profile
adb pull /data/user/0/com.yourapp/cache/sampling-profiler-trace.cpuprofile

# Open in Chrome DevTools
# chrome://tracing → Load the .cpuprofile file
```

## Common Performance Issues

### Unnecessary Re-renders

**Symptoms:** UI jank when underlying data hasn't visually changed. React DevTools shows components re-rendering on every state update.

```typescript
// BAD: Selecting entire Redux state — re-renders on ANY store change
const state = useSelector((s: RootState) => s);

// GOOD: Select only what you need
const userName = useSelector((s: RootState) => s.user.name);
const isLoggedIn = useSelector((s: RootState) => s.auth.isLoggedIn);
```

```typescript
// BAD: Creating new object/array references every render
function ParentComponent() {
  const config = { theme: 'dark', lang: 'en' }; // New object every render
  return <ChildComponent config={config} />;
}

// GOOD: Memoize objects and callbacks
function ParentComponent() {
  const config = useMemo(() => ({ theme: 'dark', lang: 'en' }), []);
  return <ChildComponent config={config} />;
}
```

```typescript
// GOOD: Wrap pure components with React.memo
const UserCard = React.memo(function UserCard({ name, avatar }: UserCardProps) {
  return (
    <View className="flex-row items-center rounded-lg bg-white p-4">
      <Image source={{ uri: avatar }} className="h-12 w-12 rounded-full" />
      <Text className="ml-3 text-base font-medium text-gray-900">{name}</Text>
    </View>
  );
});

// GOOD: Stabilize callback references
const handlePress = useCallback((id: string) => {
  navigation.navigate('Detail', { id });
}, [navigation]);
```

### FlatList Performance

```typescript
// BAD: Inline arrow function in renderItem — creates new function every render
<FlatList
  data={items}
  renderItem={({ item }) => (
    <View className="p-4">
      <Text>{item.title}</Text>
    </View>
  )}
/>

// GOOD: Extract renderItem to a named component defined outside the parent
const ItemRow = React.memo(function ItemRow({ title }: { title: string }) {
  return (
    <View className="p-4">
      <Text className="text-base text-gray-900">{title}</Text>
    </View>
  );
});

function ListScreen() {
  const renderItem = useCallback(
    ({ item }: { item: Item }) => <ItemRow title={item.title} />,
    [],
  );

  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      getItemLayout={(_, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}
      windowSize={5}
      maxToRenderPerBatch={10}
      updateCellsBatchingPeriod={50}
      removeClippedSubviews={true}
    />
  );
}
```

**Key FlatList props:**

| Prop | Default | Recommendation |
|------|---------|----------------|
| `windowSize` | 21 | Reduce to 5-11 to save memory |
| `maxToRenderPerBatch` | 10 | Increase for fast scroll, decrease for responsiveness |
| `updateCellsBatchingPeriod` | 50 | Increase to reduce render frequency |
| `removeClippedSubviews` | false | Set `true` for long lists on Android |
| `getItemLayout` | undefined | Always provide for fixed-height items (skips measurement) |

**Important:** Remove `console.log` from `renderItem` — logging on every render tanks scroll performance.

### Image Optimization

```typescript
// BAD: Loading a 4000x3000 image for a 100x100 thumbnail
import { Image } from 'react-native';
<Image
  source={{ uri: 'https://example.com/photo-full-res.jpg' }}
  style={{ width: 100, height: 100 }}
  resizeMode="cover"
/>

// GOOD: Use expo-image with server-resized URL and placeholder
import { Image } from 'expo-image';

const blurhash = 'L6PZfSi_.AyE_3t7t7R**0o#DgR4';

<Image
  source={{ uri: 'https://example.com/photo-200x200.jpg' }}
  className="h-24 w-24 rounded-lg"
  contentFit="cover"
  placeholder={{ blurhash }}
  transition={200}
  recyclingKey={item.id}
/>
```

**Why expo-image over Image:**

- Built-in disk and memory caching
- Blurhash/thumbhash placeholder support
- `contentFit` replaces `resizeMode` with better behavior
- `recyclingKey` for efficient list recycling
- Animated image support (GIF, WebP, APNG)

### Slow App Startup

```typescript
// GOOD: Lazy load heavy screens
const SettingsScreen = React.lazy(() => import('./screens/SettingsScreen'));
const AnalyticsScreen = React.lazy(() => import('./screens/AnalyticsScreen'));

// Wrap in Suspense
<Suspense fallback={<LoadingSpinner />}>
  <SettingsScreen />
</Suspense>

// GOOD: Defer non-critical initialization
import { InteractionManager } from 'react-native';

useEffect(() => {
  InteractionManager.runAfterInteractions(() => {
    // Heavy init that can wait until animations/transitions complete
    initializeAnalytics();
    prefetchNonCriticalData();
  });
}, []);
```

**Startup optimization checklist:**

- Use Hermes (enabled by default in Expo SDK 49+)
- Minimize top-level imports — lazy load where possible
- Defer analytics, crash reporting init with `InteractionManager`
- Avoid synchronous storage reads on startup (use async MMKV)
- Profile with `performance.now()` to measure each init step

### Memory Leaks

```typescript
// BAD: addEventListener without removeEventListener
useEffect(() => {
  const subscription = AppState.addEventListener('change', handleAppState);
  // Missing cleanup — subscription lives forever
}, []);

// GOOD: Always return cleanup from useEffect
useEffect(() => {
  const subscription = AppState.addEventListener('change', handleAppState);
  return () => subscription.remove();
}, []);

// BAD: Network request continues after unmount
useEffect(() => {
  fetchUserProfile(userId).then(setProfile);
}, [userId]);

// GOOD: Cancel with AbortController
useEffect(() => {
  const controller = new AbortController();

  async function loadProfile() {
    try {
      const profile = await fetchUserProfile(userId, controller.signal);
      setProfile(profile);
    } catch (error) {
      if (!controller.signal.aborted) {
        handleError(error);
      }
    }
  }

  loadProfile();
  return () => controller.abort();
}, [userId]);
```

**Detection tools:**

- **iOS:** Xcode > Debug Memory Graph (shows retain cycles)
- **Android:** Android Studio Profiler > Memory tab (track allocations)
- **JS heap:** Hermes sampling profiler or Chrome DevTools memory tab

### Animation Performance

```typescript
// BAD: Animated.Value with JS-driven callbacks — runs on JS thread, causes jank
import { Animated } from 'react-native';

const opacity = useRef(new Animated.Value(0)).current;
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: false, // JS thread — bad
}).start();

// BETTER: Always use useNativeDriver when possible
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // Native thread — smooth
}).start();

// BEST: Use Reanimated — runs entirely on UI thread via worklets
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
} from 'react-native-reanimated';

function FadeInCard({ children }: { children: ReactNode }) {
  const opacity = useSharedValue(0);

  useEffect(() => {
    opacity.value = withTiming(1, { duration: 300 });
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
  }));

  return (
    <Animated.View style={animatedStyle} className="rounded-xl bg-white p-4 shadow">
      {children}
    </Animated.View>
  );
}
```

**Animation rules:**

- Never animate `width`, `height`, `top`, `left` — these trigger layout recalculation
- Animate `transform` (translateX/Y, scale, rotate) and `opacity` instead
- `useNativeDriver: true` only works with non-layout properties
- Reanimated worklets run on the UI thread — zero JS bridge overhead

## Bundle Size Analysis

```bash
# Visualize bundle composition
npx react-native-bundle-visualizer

# For Expo projects
npx expo export --dump-sourcemap
# Then analyze the source map with source-map-explorer
npx source-map-explorer dist/bundles/ios-*.js
```

**Common bundle bloat sources:**

| Library | Size Impact | Lighter Alternative |
|---------|-------------|---------------------|
| `moment` | ~300KB | `date-fns` or `dayjs` (~2KB) |
| `lodash` (full) | ~70KB | `lodash-es` with tree-shaking or individual imports |
| Unused icon sets | ~50KB+ each | Import only needed icon families |
| Large animation libs | Varies | Use Reanimated (already needed) |

```typescript
// BAD: Import entire lodash
import _ from 'lodash';
_.debounce(fn, 300);

// GOOD: Import only what you need
import debounce from 'lodash/debounce';
debounce(fn, 300);
```

## Performance Checklist

Pre-release performance validation:

- [ ] **FlatList:** All long lists use `keyExtractor`, `getItemLayout` (if fixed height), extracted `renderItem`
- [ ] **Images:** Using `expo-image` with appropriately sized sources and placeholders
- [ ] **Memoization:** Heavy computations wrapped in `useMemo`, callbacks in `useCallback`
- [ ] **Re-renders:** Redux selectors are granular (no full-state selects)
- [ ] **Animations:** Using `useNativeDriver: true` or Reanimated
- [ ] **Console:** All `console.log` removed from render paths and hot loops
- [ ] **Bundle:** No unused large dependencies, tree-shaking enabled
- [ ] **Hermes:** Enabled (default in Expo SDK 49+)
- [ ] **Memory:** No leaked subscriptions, all `useEffect` have cleanup where needed
- [ ] **Startup:** Non-critical init deferred with `InteractionManager.runAfterInteractions`

## Related Resources

- [crash-handler.md](./crash-handler.md) - Crash diagnosis and error boundaries
- [fix-bug.md](./fix-bug.md) - General bug debugging workflow
- [common-patterns.md](../../guides/common-patterns.md) - Reusable implementation patterns
- [mobile-testing.md](../../guides/mobile-testing.md) - Testing guide
