# Crash Handler

Guide for diagnosing app crashes, implementing error boundaries, and integrating crash reporting in React Native/Expo apps.

## When to Use

- App crashes on launch or during user interaction
- Need to implement error boundaries for graceful failure recovery
- Setting up crash reporting (Sentry, Firebase Crashlytics)
- Encountering "fatal error", "app crash", "red screen of death"
- App silently disappears (native crash or OOM)

## Error Boundary Implementation

### Basic Error Boundary

React error boundaries require class components because `getDerivedStateFromError` and `componentDidCatch` have no hook equivalents.

```typescript
import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, Pressable } from 'react-native';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
  };

  static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // Send to crash reporting service
    this.props.onError?.(error, errorInfo);
  }

  resetError = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): ReactNode {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <View className="flex-1 items-center justify-center bg-white px-6">
          <Text className="mb-2 text-xl font-bold text-gray-900">
            Something went wrong
          </Text>
          <Text className="mb-6 text-center text-base text-gray-600">
            {this.state.error?.message ?? 'An unexpected error occurred.'}
          </Text>
          <Pressable
            onPress={this.resetError}
            className="rounded-lg bg-blue-600 px-6 py-3 active:bg-blue-700"
          >
            <Text className="text-base font-semibold text-white">
              Try Again
            </Text>
          </Pressable>
        </View>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

### Root-Level Integration

Wrap the entire app so no crash goes unhandled:

```typescript
// App.tsx
import ErrorBoundary from '@/components/ErrorBoundary';
import { NavigationContainer } from '@react-navigation/native';

export default function App() {
  return (
    <ErrorBoundary onError={reportCrashToService}>
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </ErrorBoundary>
  );
}

function reportCrashToService(error: Error, errorInfo: ErrorInfo): void {
  // Forward to Sentry, Crashlytics, etc.
  Sentry.captureException(error, { extra: { componentStack: errorInfo.componentStack } });
}
```

### Screen-Level Error Boundary

Wrap individual screens for granular recovery without crashing the whole app:

```typescript
// GOOD: Each screen has its own boundary — one crash doesn't take down the app
function ScreenErrorBoundary({ children }: { children: ReactNode }) {
  const navigation = useNavigation();

  const handleReset = () => {
    navigation.goBack();
  };

  return (
    <ErrorBoundary
      fallback={
        <View className="flex-1 items-center justify-center bg-white">
          <Text className="mb-4 text-lg text-gray-700">
            This screen encountered an error.
          </Text>
          <Pressable
            onPress={handleReset}
            className="rounded-lg bg-blue-600 px-6 py-3 active:bg-blue-700"
          >
            <Text className="font-semibold text-white">Go Back</Text>
          </Pressable>
        </View>
      }
    >
      {children}
    </ErrorBoundary>
  );
}

// Usage in navigator
<Stack.Screen name="Profile">
  {() => (
    <ScreenErrorBoundary>
      <ProfileScreen />
    </ScreenErrorBoundary>
  )}
</Stack.Screen>

// BAD: Single try/catch in root with console.log — errors silently swallowed
try {
  return <App />;
} catch (e) {
  console.log(e); // User sees nothing, no recovery path
}
```

## Crash Reporting Setup

### Sentry for React Native

**Installation:**

```bash
npx expo install @sentry/react-native
```

**Configuration in app.config.ts:**

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  plugins: [
    [
      '@sentry/react-native/expo',
      {
        organization: 'your-org',
        project: 'your-project',
      },
    ],
  ],
});
```

**Initialization in App.tsx:**

```typescript
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  debug: __DEV__,
  tracesSampleRate: __DEV__ ? 1.0 : 0.2,
  enableAutoSessionTracking: true,
  attachStacktrace: true,
});

// Wrap the root component
export default Sentry.wrap(App);
```

**Custom error context:**

```typescript
// Set user context after login
Sentry.setUser({ id: user.id, email: user.email });

// Add breadcrumbs for tracing user actions
Sentry.addBreadcrumb({
  category: 'navigation',
  message: `Navigated to ${screenName}`,
  level: 'info',
});

// Capture handled errors with extra context
Sentry.captureException(error, {
  tags: { screen: 'Checkout', flow: 'payment' },
  extra: { cartItems: cart.items.length },
});
```

**Source maps for EAS builds:**

```bash
# In eas.json, add the Sentry upload step
eas build --platform ios --profile production
# Sentry plugin auto-uploads source maps when configured in app.config.ts
```

### Firebase Crashlytics

Use Crashlytics when already invested in the Firebase ecosystem (Analytics, Remote Config, FCM):

```bash
npx expo install @react-native-firebase/app @react-native-firebase/crashlytics
```

```typescript
import crashlytics from '@react-native-firebase/crashlytics';

// Log non-fatal errors
crashlytics().recordError(error);

// Set user identifier
crashlytics().setUserId(user.id);

// Add custom keys for filtering in dashboard
crashlytics().setAttribute('screen', 'Profile');
```

**When to choose Crashlytics vs Sentry:**

| Criteria | Sentry | Crashlytics |
|----------|--------|-------------|
| JS + native crash detail | Excellent | Good |
| Source map support | Built-in | Manual setup |
| Performance monitoring | Yes (tracing) | Via Firebase Performance |
| Already using Firebase | Not required | Natural fit |
| Free tier | 5K errors/month | Unlimited |

## Common Crash Patterns

### JS Thread Crashes

```typescript
// ❌ Null/undefined access in render
const ProfileScreen = ({ user }: { user: User | null }) => {
  return <Text>{user.name}</Text>; // Crashes if user is null
};

// ✅ Guard against null
const ProfileScreen = ({ user }: { user: User | null }) => {
  if (!user) return <LoadingSpinner />;
  return <Text>{user.name}</Text>;
};

// ❌ Unhandled promise rejection
async function fetchData() {
  const response = await api.get('/data'); // Crashes if network fails
}

// ✅ Handle promise rejection
async function fetchData() {
  try {
    const response = await api.get('/data');
    return response.data;
  } catch (error) {
    Sentry.captureException(error);
    throw error; // Re-throw so error boundary can catch in render
  }
}

// ❌ Infinite loop in useEffect
useEffect(() => {
  setState(value); // Triggers re-render, re-triggers effect
}, [value, state]); // state changes every render

// ❌ Stack overflow from recursive rendering
const BadComponent = () => <BadComponent />; // Infinite recursion
```

### Native Crashes

| Cause | Symptom | Fix |
|-------|---------|-----|
| Native module not linked | `NativeModule.X is null` | `npx expo install`, `cd ios && pod install` |
| Memory pressure (OOM) | App killed silently | Optimize images, fix FlatList, clean subscriptions |
| Threading violation | Random crash in native code | Ensure UI updates on main thread |
| Corrupted native state | Crash after hot reload | Full app restart, clean build |

### How to Distinguish JS vs Native Crash

| Signal | JS Crash | Native Crash |
|--------|----------|--------------|
| Visual | Red screen (LogBox) | App disappears instantly |
| Stack trace | Readable JS file/line | Memory addresses, native symbols |
| Where to look | Metro console, React DevTools | Xcode console, `adb logcat` |
| Recovery | Error boundary can catch | App must restart |
| Reproducibility | Usually consistent | May be intermittent |

## Reading Crash Logs

### iOS (Xcode Console)

```bash
# View crash logs in Xcode
# Window → Devices and Simulators → View Device Logs

# Symbolicate a .crash file
# Xcode auto-symbolicates if you have the dSYM files

# Key fields in a crash report:
# Exception Type:   EXC_BAD_ACCESS (SIGSEGV)     ← memory access violation
# Exception Codes:  KERN_INVALID_ADDRESS          ← tried to access freed memory
# Crashed Thread:   0                             ← main thread crash
# Thread 0 Backtrace:                             ← the stack trace to read
```

### Android (Logcat)

```bash
# Error-level logs only
adb logcat *:E

# Filter for React Native
adb logcat | grep -E "ReactNative|ReactNativeJS|FATAL"

# Common fatal patterns
# FATAL EXCEPTION: main
# java.lang.RuntimeException: Unable to start activity

# For native crashes, check tombstone files
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_00
```

## OOM (Out of Memory) Diagnosis

**Symptoms:** App is killed silently with no crash log. On iOS, look for `Jetsam` events in device logs. On Android, check for `Low Memory Killer` in logcat.

**Common causes:**

```typescript
// ❌ Loading full-resolution images
<Image source={{ uri: 'https://example.com/photo-4000x3000.jpg' }} />

// ✅ Use expo-image with appropriately sized URLs
import { Image } from 'expo-image';
<Image
  source={{ uri: 'https://example.com/photo-400x300.jpg' }}
  className="h-24 w-24"
  contentFit="cover"
  placeholder={blurhash}
/>

// ❌ FlatList rendering all items at once
<FlatList data={items} renderItem={renderItem} windowSize={100} />

// ✅ Limit render window
<FlatList
  data={items}
  renderItem={renderItem}
  windowSize={5}
  maxToRenderPerBatch={10}
  removeClippedSubviews={true}
/>

// ❌ Event subscriptions never cleaned up
useEffect(() => {
  const sub = EventEmitter.addListener('update', handler);
  // No cleanup!
}, []);

// ✅ Always clean up subscriptions
useEffect(() => {
  const sub = EventEmitter.addListener('update', handler);
  return () => sub.remove();
}, []);
```

**Diagnostic tools:**

- **iOS:** Xcode Instruments > Allocations / Leaks / Memory Graph Debugger
- **Android:** Android Studio Profiler > Memory tab

## Recovery Strategies

### Retry with Error Boundary

```typescript
// GOOD: Error boundary with contextual recovery per screen section
<ErrorBoundary
  fallback={
    <View className="rounded-lg bg-red-50 p-4">
      <Text className="text-red-800">Failed to load feed.</Text>
      <Pressable
        onPress={() => queryClient.invalidateQueries({ queryKey: ['feed'] })}
        className="mt-2 rounded bg-red-600 px-4 py-2 active:bg-red-700"
      >
        <Text className="text-white">Retry</Text>
      </Pressable>
    </View>
  }
>
  <FeedSection />
</ErrorBoundary>
```

### Clear State and Redirect

```typescript
// When cached data causes a crash loop
async function emergencyReset(navigation: NavigationProp): Promise<void> {
  // Clear async storage / MMKV
  storage.clearAll();
  // Clear query cache
  queryClient.clear();
  // Navigate to a safe screen
  navigation.reset({
    index: 0,
    routes: [{ name: 'Home' }],
  });
}
```

### Graceful Degradation

```typescript
// GOOD: Show partial UI when a section fails
function DashboardScreen() {
  return (
    <ScrollView className="flex-1 bg-gray-50">
      <ErrorBoundary fallback={<PlaceholderCard message="Stats unavailable" />}>
        <StatsSection />
      </ErrorBoundary>
      <ErrorBoundary fallback={<PlaceholderCard message="Feed unavailable" />}>
        <FeedSection />
      </ErrorBoundary>
    </ScrollView>
  );
}

// BAD: Single try/catch in root with console.log — user sees blank screen
```

## Related Resources

- [fix-bug.md](./fix-bug.md) - General bug debugging workflow
- [performance-debugger.md](./performance-debugger.md) - Performance diagnosis and optimization
- [data-fetching.md](../../guides/data-fetching.md) - HttpService and error handling patterns
- [mobile-testing.md](../../guides/mobile-testing.md) - Testing guide
