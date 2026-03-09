# Deep Linking Guide

Setting up and handling deep links in React Native with Expo Router and React Navigation.

## Table of Contents

- [URL Scheme Setup](#url-scheme-setup)
- [Expo Router Deep Links](#expo-router-deep-links)
- [React Navigation Deep Links](#react-navigation-deep-links)
- [Universal Links (iOS)](#universal-links-ios)
- [App Links (Android)](#app-links-android)
- [Handling Deep Links](#handling-deep-links)
- [Testing Deep Links](#testing-deep-links)

---

## URL Scheme Setup

### Custom URL Scheme

Register a custom scheme in `app.config.ts` so your app responds to URLs like `myapp://path`:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  scheme: 'myapp',
  // For multiple schemes:
  // scheme: ['myapp', 'myapp-dev'],
});
```

### Testing the Scheme

```bash
# iOS Simulator
npx uri-scheme open myapp://profile/123 --ios

# Android Emulator
npx uri-scheme open myapp://profile/123 --android

# Or via adb directly
adb shell am start -a android.intent.action.VIEW -d "myapp://profile/123"
```

---

## Expo Router Deep Links

Expo Router generates linking configuration automatically from your file structure. Every route file maps to a URL path.

### File Structure to URL Mapping

```
app/
├── (tabs)/
│   ├── index.tsx          → myapp://
│   ├── search.tsx         → myapp://search
│   └── profile.tsx        → myapp://profile
├── product/
│   └── [id].tsx           → myapp://product/42
├── order/
│   └── [orderId]/
│       ├── index.tsx      → myapp://order/abc123
│       └── tracking.tsx   → myapp://order/abc123/tracking
└── settings/
    └── notifications.tsx  → myapp://settings/notifications
```

### Typed Route Parameters

```tsx
// app/product/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { View, Text } from 'react-native';

export default function ProductScreen() {
  // Type-safe params from the route segment
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View className="flex-1 p-4">
      <Text className="text-xl font-bold">Product #{id}</Text>
    </View>
  );
}
```

### Nested Dynamic Routes

```tsx
// app/order/[orderId]/tracking.tsx
import { useLocalSearchParams } from 'expo-router';

export default function OrderTrackingScreen() {
  const { orderId } = useLocalSearchParams<{ orderId: string }>();

  return (
    <View className="flex-1 p-4">
      <Text className="text-lg font-semibold">Tracking Order: {orderId}</Text>
      <TrackingMap orderId={orderId} />
    </View>
  );
}
```

### Navigating with Links

```tsx
import { Link, useRouter } from 'expo-router';

function ProductCard({ product }: { product: Product }) {
  return (
    // Declarative navigation
    <Link
      href={`/product/${product.id}`}
      accessibilityRole="link"
      accessibilityLabel={`View ${product.name}`}
      className="rounded-lg bg-white p-4 shadow-sm"
    >
      <Text className="font-medium">{product.name}</Text>
    </Link>
  );
}

function SearchResults() {
  const router = useRouter();

  const handleProductSelect = (productId: string) => {
    // Imperative navigation
    router.push(`/product/${productId}`);
  };

  return <ProductList onSelect={handleProductSelect} />;
}
```

---

## React Navigation Deep Links

For projects using React Navigation instead of Expo Router, configure deep links manually via the `linking` prop.

### Linking Configuration

```typescript
// navigation/linking.ts
import { LinkingOptions } from '@react-navigation/native';
import type { RootStackParamList } from './types';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://myapp.com'],

  config: {
    screens: {
      Home: '',
      Search: 'search',
      Product: 'product/:id',
      Order: {
        path: 'order/:orderId',
        screens: {
          OrderDetails: '',
          OrderTracking: 'tracking',
        },
      },
      Settings: {
        path: 'settings',
        screens: {
          Notifications: 'notifications',
          Privacy: 'privacy',
        },
      },
      NotFound: '*', // Catch-all for unknown routes
    },
  },
};
```

### NavigationContainer Setup

```tsx
import { NavigationContainer } from '@react-navigation/native';
import { linking } from './navigation/linking';

function App() {
  return (
    <NavigationContainer
      linking={linking}
      fallback={<LoadingScreen />} // Shown while resolving deep link
    >
      <RootNavigator />
    </NavigationContainer>
  );
}
```

### Custom Path Parsing

```typescript
import { getStateFromPath, getPathFromState } from '@react-navigation/native';

export const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://'],
  config: { /* ... */ },

  // Custom URL → navigation state parsing
  getStateFromPath(path, options) {
    // Handle legacy URLs
    const legacyPath = path.replace('/items/', '/product/');
    return getStateFromPath(legacyPath, options);
  },

  // Custom navigation state → URL conversion
  getPathFromState(state, options) {
    return getPathFromState(state, options);
  },
};
```

---

## Universal Links (iOS)

Universal links open your app from standard `https://` URLs, providing a seamless web-to-app experience.

### Apple App Site Association

Host this file at `https://yourdomain.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appIDs": ["TEAM_ID.com.mycompany.myapp"],
        "components": [
          { "/": "/product/*", "comment": "Product pages" },
          { "/": "/order/*", "comment": "Order pages" },
          { "/": "/invite/*", "comment": "Invite links" }
        ]
      }
    ]
  }
}
```

Requirements:
- Must be served over HTTPS
- Content-Type: `application/json`
- No redirects allowed
- File must be at the exact path `/.well-known/apple-app-site-association`

### Expo Config Plugin

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  ios: {
    bundleIdentifier: 'com.mycompany.myapp',
    associatedDomains: ['applinks:myapp.com', 'applinks:www.myapp.com'],
  },
});
```

### Testing Universal Links

```bash
# Test with iOS Simulator
xcrun simctl openurl booted "https://myapp.com/product/42"

# Verify AASA file is valid
curl -s "https://myapp.com/.well-known/apple-app-site-association" | python3 -m json.tool

# Apple's CDN caches AASA files — use their validator
# https://search.developer.apple.com/appsearch-validation-tool/
```

---

## App Links (Android)

Android App Links are the Android equivalent of Universal Links.

### Asset Links File

Host at `https://yourdomain.com/.well-known/assetlinks.json`:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": [
        "AA:BB:CC:DD:EE:FF:..."
      ]
    }
  }
]
```

Get your SHA-256 fingerprint:

```bash
# Debug keystore
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android

# Or from EAS
eas credentials --platform android
```

### Intent Filters in app.config.ts

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  android: {
    package: 'com.mycompany.myapp',
    intentFilters: [
      {
        action: 'VIEW',
        autoVerify: true,
        data: [
          {
            scheme: 'https',
            host: 'myapp.com',
            pathPrefix: '/product',
          },
          {
            scheme: 'https',
            host: 'myapp.com',
            pathPrefix: '/order',
          },
        ],
        category: ['BROWSABLE', 'DEFAULT'],
      },
    ],
  },
});
```

### Testing App Links

```bash
# Open a link in the emulator
adb shell am start -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://myapp.com/product/42"

# Verify App Links status
adb shell pm get-app-links com.mycompany.myapp

# Force re-verification
adb shell pm verify-app-links --re-verify com.mycompany.myapp
```

---

## Handling Deep Links

### Cold Start vs Warm Start

```typescript
import { useEffect } from 'react';
import { Linking } from 'react-native';

function useDeepLinkHandler(handler: (url: string) => void): void {
  useEffect(() => {
    // Cold start: app was not running, opened via link
    Linking.getInitialURL().then((url) => {
      if (url) {
        handler(url);
      }
    });

    // Warm start: app is in background, opened via link
    const subscription = Linking.addEventListener('url', (event) => {
      handler(event.url);
    });

    return () => subscription.remove();
  }, [handler]);
}
```

### Auth-Gated Deep Links

When a deep link targets an authenticated route, save the URL and redirect after login:

```typescript
import { useEffect } from 'react';
import { Linking } from 'react-native';
import { useRouter } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';
import { useMMKVString } from 'react-native-mmkv';

function useAuthGatedDeepLink(): void {
  const { isAuthenticated } = useAuth();
  const router = useRouter();
  const [pendingDeepLink, setPendingDeepLink] = useMMKVString('pendingDeepLink');

  // Save deep link URL when user is not authenticated
  useEffect(() => {
    const handleUrl = (url: string) => {
      const isProtectedRoute = url.includes('/order') || url.includes('/profile');

      if (isProtectedRoute && !isAuthenticated) {
        setPendingDeepLink(url);
        router.replace('/login');
      }
    };

    Linking.getInitialURL().then((url) => {
      if (url) handleUrl(url);
    });

    const subscription = Linking.addEventListener('url', (event) => {
      handleUrl(event.url);
    });

    return () => subscription.remove();
  }, [isAuthenticated]);

  // Redirect to saved URL after login
  useEffect(() => {
    if (isAuthenticated && pendingDeepLink) {
      const url = pendingDeepLink;
      setPendingDeepLink(undefined);

      // Parse path from URL and navigate
      const path = new URL(url).pathname;
      router.replace(path as never);
    }
  }, [isAuthenticated, pendingDeepLink]);
}
```

### Deferred Deep Links

For first-install scenarios (user clicks link, goes to app store, installs, then should land on the right screen):

```typescript
// Deferred deep links typically require a third-party service like:
// - Branch.io
// - Firebase Dynamic Links (deprecated, migrate to alternatives)
// - Adjust
// - AppsFlyer

// Example with a generic deferred deep link service:
import { useDeferredDeepLink } from '@/services/deepLinkService';

function App() {
  useDeferredDeepLink((url: string) => {
    // Handle the deferred link after first install
    const path = extractPath(url);
    router.push(path);
  });

  return <RootNavigator />;
}
```

---

## Testing Deep Links

### Simulator and Emulator Commands

```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://product/42"
xcrun simctl openurl booted "https://myapp.com/order/abc123"

# Android Emulator
adb shell am start -a android.intent.action.VIEW -d "myapp://product/42"
adb shell am start -a android.intent.action.VIEW -d "https://myapp.com/order/abc123"

# Expo CLI (development)
npx uri-scheme open "myapp://product/42" --ios
npx uri-scheme open "myapp://product/42" --android
```

### Expo Go Limitations

Expo Go uses its own URL scheme (`exp://`), so custom schemes do not work directly. For deep link testing:

- Use a **development build** (`npx expo run:ios` or EAS Build with `developmentClient: true`)
- For Expo Go, test with `exp://127.0.0.1:8081/--/product/42` format

### E2E Testing with Maestro

```yaml
# deep-link-test.yaml
appId: com.mycompany.myapp
---
- openLink: "myapp://product/42"
- assertVisible: "Product #42"
- back
- openLink: "https://myapp.com/order/abc123"
- assertVisible: "Order: abc123"
- openLink: "myapp://settings/notifications"
- assertVisible: "Notification Settings"
```

### Unit Testing Deep Link Handling

```typescript
import { Linking } from 'react-native';
import { renderHook } from '@testing-library/react-native';
import { useDeepLinkHandler } from '@/hooks/useDeepLinkHandler';

jest.mock('react-native', () => ({
  ...jest.requireActual('react-native'),
  Linking: {
    getInitialURL: jest.fn(),
    addEventListener: jest.fn(),
  },
}));

describe('useDeepLinkHandler', () => {
  it('should handle cold start deep link', async () => {
    const handler = jest.fn();
    (Linking.getInitialURL as jest.Mock).mockResolvedValueOnce(
      'myapp://product/42'
    );

    renderHook(() => useDeepLinkHandler(handler));

    await waitFor(() => {
      expect(handler).toHaveBeenCalledWith('myapp://product/42');
    });
  });

  it('should handle warm start deep link', () => {
    const handler = jest.fn();
    let urlCallback: (event: { url: string }) => void;

    (Linking.addEventListener as jest.Mock).mockImplementation(
      (_event: string, callback: (event: { url: string }) => void) => {
        urlCallback = callback;
        return { remove: jest.fn() };
      }
    );
    (Linking.getInitialURL as jest.Mock).mockResolvedValueOnce(null);

    renderHook(() => useDeepLinkHandler(handler));

    // Simulate incoming URL while app is running
    urlCallback!({ url: 'myapp://order/abc123' });

    expect(handler).toHaveBeenCalledWith('myapp://order/abc123');
  });
});
```

---

## Related Resources

- [Navigation Guide](./navigation-guide.md) — Expo Router and React Navigation setup
- [E2E Test Generator Skill](../skills/e2e-testing/e2e-test-generator.md) — automated E2E test creation
