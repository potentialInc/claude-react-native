# Test Mocking Patterns

Comprehensive mocking reference for React Native testing with Jest. Covers native modules, navigation, state management, storage, and test utilities.

---

## When to Use This Guide

This guide activates when:
- Setting up test infrastructure for a React Native project
- Writing tests that need to mock native modules, navigation, or state management
- Debugging test failures caused by unmocked native dependencies
- Creating shared test utilities (renderWithProviders, mock factories)

---

## jest.setup.ts - Complete Setup File

```typescript
// src/test/jest.setup.ts
import '@testing-library/react-native/extend-expect';

// --- React Native Reanimated ---
jest.mock('react-native-reanimated', () =>
  require('react-native-reanimated/mock')
);

// --- React Native Gesture Handler ---
jest.mock('react-native-gesture-handler', () => ({
  GestureHandlerRootView: ({ children }: { children: React.ReactNode }) =>
    children,
  Swipeable: jest.fn(),
  DrawerLayout: jest.fn(),
  State: {},
  PanGestureHandler: jest.fn(),
  TapGestureHandler: jest.fn(),
  FlingGestureHandler: jest.fn(),
  ForceTouchGestureHandler: jest.fn(),
  LongPressGestureHandler: jest.fn(),
  ScrollView: jest.fn(),
  Directions: {},
}));

// --- Async Storage ---
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// --- MMKV ---
jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    getString: jest.fn(),
    getNumber: jest.fn(),
    getBoolean: jest.fn(),
    set: jest.fn(),
    delete: jest.fn(),
    contains: jest.fn().mockReturnValue(false),
    getAllKeys: jest.fn().mockReturnValue([]),
    clearAll: jest.fn(),
    addOnValueChangedListener: jest.fn().mockReturnValue({ remove: jest.fn() }),
  })),
}));

// --- Expo Modules ---
jest.mock('expo-constants', () => ({
  default: {
    expoConfig: {
      extra: {
        apiUrl: 'https://test-api.example.com',
      },
    },
  },
}));

jest.mock('expo-linking', () => ({
  createURL: jest.fn((path: string) => `myapp://${path}`),
  openURL: jest.fn(),
}));

jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn().mockResolvedValue(null),
  setItemAsync: jest.fn().mockResolvedValue(undefined),
  deleteItemAsync: jest.fn().mockResolvedValue(undefined),
}));

jest.mock('expo-image', () => ({
  Image: 'ExpoImage',
}));

// --- Safe Area ---
jest.mock('react-native-safe-area-context', () => ({
  SafeAreaProvider: ({ children }: { children: React.ReactNode }) => children,
  SafeAreaView: ({ children }: { children: React.ReactNode }) => children,
  useSafeAreaInsets: () => ({ top: 0, bottom: 0, left: 0, right: 0 }),
}));
```

---

## Navigation Mocking

### React Navigation

```typescript
// src/test/mocks/navigation.ts
const mockNavigate = jest.fn();
const mockGoBack = jest.fn();
const mockReset = jest.fn();
const mockSetOptions = jest.fn();

export const mockNavigation = {
  navigate: mockNavigate,
  goBack: mockGoBack,
  reset: mockReset,
  setOptions: mockSetOptions,
  addListener: jest.fn().mockReturnValue(jest.fn()),
  canGoBack: jest.fn().mockReturnValue(true),
};

jest.mock('@react-navigation/native', () => {
  const actual = jest.requireActual('@react-navigation/native');
  return {
    ...actual,
    useNavigation: () => mockNavigation,
    useRoute: () => ({
      params: {},
      key: 'test-route-key',
      name: 'TestScreen',
    }),
    useFocusEffect: (callback: () => void) => callback(),
    useIsFocused: () => true,
  };
});

// Override route params per-test:
// jest.mocked(useRoute).mockReturnValue({
//   params: { userId: '42' },
//   key: 'test-key',
//   name: 'Profile',
// });
```

### Expo Router

```typescript
// src/test/mocks/expo-router.ts
const mockPush = jest.fn();
const mockReplace = jest.fn();
const mockBack = jest.fn();

export const mockRouter = {
  push: mockPush,
  replace: mockReplace,
  back: mockBack,
  canGoBack: jest.fn().mockReturnValue(true),
  setParams: jest.fn(),
};

jest.mock('expo-router', () => ({
  useRouter: () => mockRouter,
  useLocalSearchParams: () => ({}),
  useGlobalSearchParams: () => ({}),
  useSegments: () => [],
  usePathname: () => '/',
  Link: ({ children }: { children: React.ReactNode }) => children,
  Redirect: jest.fn(() => null),
  Stack: {
    Screen: jest.fn(() => null),
  },
  Tabs: {
    Screen: jest.fn(() => null),
  },
}));

// Override search params per-test:
// jest.mocked(useLocalSearchParams).mockReturnValue({ id: '123' });
```

---

## State Management Mocking

### Redux Store

```typescript
// src/test/test-utils.tsx
import { render, type RenderOptions } from '@testing-library/react-native';
import { configureStore, type PreloadedState } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';
import type { ReactNode, ReactElement } from 'react';
import { rootReducer, type RootState } from '@/store';

function createTestStore(preloadedState?: PreloadedState<RootState>) {
  return configureStore({
    reducer: rootReducer,
    preloadedState,
  });
}

interface ExtendedRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  preloadedState?: PreloadedState<RootState>;
  store?: ReturnType<typeof createTestStore>;
}

export function renderWithProviders(
  ui: ReactElement,
  {
    preloadedState,
    store = createTestStore(preloadedState),
    ...renderOptions
  }: ExtendedRenderOptions = {}
) {
  function Wrapper({ children }: { children: ReactNode }) {
    return <Provider store={store}>{children}</Provider>;
  }

  return {
    store,
    ...render(ui, { wrapper: Wrapper, ...renderOptions }),
  };
}
```

### TanStack Query

```typescript
// src/test/query-utils.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';

export function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: Infinity,
      },
      mutations: {
        retry: false,
      },
    },
    logger: {
      log: console.log,
      warn: console.warn,
      error: () => {}, // suppress error logs in tests
    },
  });
}

export function createQueryWrapper() {
  const queryClient = createTestQueryClient();
  return function QueryWrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}
```

---

## HTTP/API Mocking

### httpService / Axios Mock

```typescript
// src/test/mocks/httpService.ts
jest.mock('@/services/httpService', () => ({
  httpService: {
    get: jest.fn(),
    post: jest.fn(),
    put: jest.fn(),
    patch: jest.fn(),
    delete: jest.fn(),
    interceptors: {
      request: { use: jest.fn(), eject: jest.fn() },
      response: { use: jest.fn(), eject: jest.fn() },
    },
  },
}));

// Usage in tests:
import { httpService } from '@/services/httpService';
const mockedHttp = jest.mocked(httpService);

// Mock a successful response
mockedHttp.get.mockResolvedValueOnce({
  data: { users: [{ id: '1', name: 'Jane' }] },
});

// Mock an error response
mockedHttp.post.mockRejectedValueOnce({
  response: { status: 422, data: { message: 'Validation failed' } },
});
```

---

## Storage Mocking

### MMKV Mock

```typescript
// Inline mock for per-test control
import { MMKV } from 'react-native-mmkv';

const mockStorage = new Map<string, string>();

jest.mocked(MMKV).mockImplementation(
  () =>
    ({
      getString: jest.fn((key: string) => mockStorage.get(key)),
      set: jest.fn((key: string, value: string) => mockStorage.set(key, value)),
      delete: jest.fn((key: string) => mockStorage.delete(key)),
      contains: jest.fn((key: string) => mockStorage.has(key)),
      getAllKeys: jest.fn(() => [...mockStorage.keys()]),
      clearAll: jest.fn(() => mockStorage.clear()),
    }) as unknown as MMKV
);
```

### SecureStore Mock

```typescript
import * as SecureStore from 'expo-secure-store';

const secureStorage = new Map<string, string>();

jest.mocked(SecureStore.getItemAsync).mockImplementation(
  async (key: string) => secureStorage.get(key) ?? null
);

jest.mocked(SecureStore.setItemAsync).mockImplementation(
  async (key: string, value: string) => {
    secureStorage.set(key, value);
  }
);

jest.mocked(SecureStore.deleteItemAsync).mockImplementation(
  async (key: string) => {
    secureStorage.delete(key);
  }
);
```

---

## Timer and Animation Mocking

### Fake Timers

```typescript
describe('TimerComponent', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('auto-dismisses after 3 seconds', () => {
    const onDismiss = jest.fn();
    render(<Toast message="Saved" onDismiss={onDismiss} />);

    expect(onDismiss).not.toHaveBeenCalled();

    act(() => {
      jest.advanceTimersByTime(3000);
    });

    expect(onDismiss).toHaveBeenCalledTimes(1);
  });
});
```

### Reanimated Animations

```typescript
// For components using useAnimatedStyle or useSharedValue:
// The mock in jest.setup.ts handles most cases. For specific value testing:
jest.mock('react-native-reanimated', () => {
  const Reanimated = require('react-native-reanimated/mock');
  Reanimated.default.call = () => {};
  return Reanimated;
});
```

---

## Platform Mocking

### Platform.OS

```typescript
import { Platform } from 'react-native';

describe('PlatformButton', () => {
  const originalOS = Platform.OS;

  afterEach(() => {
    Platform.OS = originalOS;
  });

  it('renders iOS-style button on iOS', () => {
    Platform.OS = 'ios';
    render(<PlatformButton title="Press me" />);

    expect(screen.getByRole('button')).toHaveStyle({
      borderRadius: 12,
    });
  });

  it('renders Android-style button on Android', () => {
    Platform.OS = 'android';
    render(<PlatformButton title="Press me" />);

    expect(screen.getByRole('button')).toHaveStyle({
      borderRadius: 4,
    });
  });
});
```

---

## Test Utilities

### renderWithProviders (All Providers Combined)

```typescript
// src/test/test-utils.tsx
import { render, type RenderOptions } from '@testing-library/react-native';
import { configureStore, type PreloadedState } from '@reduxjs/toolkit';
import { Provider } from 'react-redux';
import { QueryClientProvider } from '@tanstack/react-query';
import { NavigationContainer } from '@react-navigation/native';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import type { ReactNode, ReactElement } from 'react';
import { rootReducer, type RootState } from '@/store';
import { createTestQueryClient } from './query-utils';

interface AllProvidersOptions extends Omit<RenderOptions, 'wrapper'> {
  preloadedState?: PreloadedState<RootState>;
  withNavigation?: boolean;
}

export function renderWithAllProviders(
  ui: ReactElement,
  {
    preloadedState,
    withNavigation = false,
    ...renderOptions
  }: AllProvidersOptions = {}
) {
  const store = configureStore({
    reducer: rootReducer,
    preloadedState,
  });
  const queryClient = createTestQueryClient();

  function Wrapper({ children }: { children: ReactNode }) {
    let content = children;

    if (withNavigation) {
      content = <NavigationContainer>{content}</NavigationContainer>;
    }

    return (
      <SafeAreaProvider>
        <Provider store={store}>
          <QueryClientProvider client={queryClient}>
            {content}
          </QueryClientProvider>
        </Provider>
      </SafeAreaProvider>
    );
  }

  return {
    store,
    queryClient,
    ...render(ui, { wrapper: Wrapper, ...renderOptions }),
  };
}

// Re-export everything from RNTL for convenience
export * from '@testing-library/react-native';
export { renderWithAllProviders as render };
```

### createMockNavigation Utility

```typescript
// src/test/mocks/create-mock-navigation.ts
export function createMockNavigation() {
  return {
    navigate: jest.fn(),
    goBack: jest.fn(),
    reset: jest.fn(),
    setOptions: jest.fn(),
    setParams: jest.fn(),
    addListener: jest.fn().mockReturnValue(jest.fn()),
    removeListener: jest.fn(),
    canGoBack: jest.fn().mockReturnValue(true),
    getParent: jest.fn(),
    getState: jest.fn().mockReturnValue({
      routes: [],
      index: 0,
    }),
  };
}
```

---

## GOOD vs BAD Practices

### Mock Boundaries

- **GOOD:** Mock at module boundary (services, navigation, storage)
- **BAD:** Mock internal functions of the component under test

### Mock Cleanup

- **GOOD:** Reset mocks in `beforeEach` with `jest.clearAllMocks()`
- **BAD:** Shared mutable mock state that leaks between tests

### Type Safety

- **GOOD:** `jest.mocked(httpService)` -- preserves types, catches API changes
- **BAD:** `(httpService as any).get.mockResolvedValue(...)` -- hides type errors

### Mock Specificity

- **GOOD:** `mockResolvedValueOnce` -- explicit per-test, no cross-test leakage
- **BAD:** `mockResolvedValue` at module level -- shared state, ordering issues

### Provider Setup

- **GOOD:** `renderWithAllProviders` utility -- consistent, DRY, easy to maintain
- **BAD:** Manually wrapping in each test file -- duplicated, drift-prone

---

## Related Resources

- [unit-test-generator.md](./unit-test-generator.md) -- Unit test patterns for components, hooks, and screens
- [mobile-testing.md](../../guides/mobile-testing.md) -- Mobile testing strategies guide
- [e2e-test-generator.md](../e2e-testing/e2e-test-generator.md) -- E2E test generation with Detox and Maestro
