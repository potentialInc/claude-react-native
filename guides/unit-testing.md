# Unit Testing Guide

Comprehensive guide for unit testing React Native applications with Jest and React Native Testing Library (RNTL).

## Table of Contents

- [Jest Configuration](#jest-configuration)
- [Test Setup File](#test-setup-file)
- [Testing Components with RNTL](#testing-components-with-rntl)
- [Testing Hooks](#testing-hooks)
- [Testing Redux](#testing-redux)
- [Testing TanStack Query Hooks](#testing-tanstack-query-hooks)
- [Testing Navigation](#testing-navigation)
- [Coverage Configuration](#coverage-configuration)
- [Best Practices](#best-practices)

---

## Jest Configuration

Create `jest.config.ts` at the project root:

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'react-native',
  testEnvironment: 'node',
  transform: {
    '^.+\\.tsx?$': [
      'ts-jest',
      {
        tsconfig: 'tsconfig.spec.json',
      },
    ],
  },
  transformIgnorePatterns: [
    'node_modules/(?!(' +
      'react-native|' +
      '@react-native|' +
      'react-native-reanimated|' +
      'react-native-gesture-handler|' +
      'react-native-screens|' +
      'react-native-safe-area-context|' +
      '@react-navigation|' +
      'expo|' +
      'expo-router|' +
      'expo-modules-core|' +
      'nativewind' +
    ')/)',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@store/(.*)$': '<rootDir>/src/store/$1',
    '\\.(png|jpg|jpeg|gif|svg)$': '<rootDir>/__mocks__/fileMock.ts',
  },
  setupFilesAfterSetup: ['<rootDir>/jest.setup.ts'],
  testMatch: ['**/__tests__/**/*.test.ts?(x)', '**/*.test.ts?(x)'],
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
    '!src/**/*.types.ts',
  ],
};

export default config;
```

For Expo projects, use the Expo preset instead:

```typescript
const config: Config = {
  preset: 'jest-expo',
  // ... rest of config
};
```

---

## Test Setup File

Create `jest.setup.ts` with essential mocks:

```typescript
import '@testing-library/react-native/extend-expect';

// Mock Reanimated
jest.mock('react-native-reanimated', () =>
  require('react-native-reanimated/mock')
);

// Mock Gesture Handler
jest.mock('react-native-gesture-handler', () => {
  const View = require('react-native').View;
  return {
    GestureHandlerRootView: View,
    Swipeable: View,
    DrawerLayout: View,
    State: {},
    PanGestureHandler: View,
    TapGestureHandler: View,
    gestureHandlerRootHOC: (component: React.ComponentType) => component,
  };
});

// Mock Safe Area
jest.mock('react-native-safe-area-context', () => ({
  SafeAreaProvider: ({ children }: { children: React.ReactNode }) => children,
  SafeAreaView: ({ children }: { children: React.ReactNode }) => children,
  useSafeAreaInsets: () => ({ top: 0, bottom: 0, left: 0, right: 0 }),
}));

// Mock MMKV
jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    getString: jest.fn(),
    set: jest.fn(),
    delete: jest.fn(),
    contains: jest.fn(),
    getAllKeys: jest.fn().mockReturnValue([]),
  })),
}));
```

> For a full reference of mock patterns for common React Native libraries, see the [test-mocking-patterns skill](../skills/debugging/test-mocking-patterns.md).

---

## Testing Components with RNTL

### Basic Rendering and Queries

```typescript
import { render, screen } from '@testing-library/react-native';
import { LoginForm } from '@/components/LoginForm';

describe('LoginForm', () => {
  it('should render email and password fields', () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    // GOOD: Query by role and accessible name
    expect(screen.getByRole('text', { name: 'Email' })).toBeOnTheScreen();
    expect(screen.getByLabelText('Password')).toBeOnTheScreen();
    expect(screen.getByRole('button', { name: 'Sign In' })).toBeOnTheScreen();
  });

  it('should not render error message initially', () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    // queryBy returns null instead of throwing
    expect(screen.queryByText('Invalid credentials')).not.toBeOnTheScreen();
  });

  it('should display loading indicator while submitting', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    // findBy waits for element to appear (async)
    const loader = await screen.findByText('Submitting...');
    expect(loader).toBeOnTheScreen();
  });
});
```

### User Events

```typescript
import { render, screen } from '@testing-library/react-native';
import { userEvent } from '@testing-library/react-native';
import { LoginForm } from '@/components/LoginForm';

describe('LoginForm interactions', () => {
  it('should call onSubmit with email and password', async () => {
    const mockSubmit = jest.fn();
    const user = userEvent.setup();

    render(<LoginForm onSubmit={mockSubmit} />);

    const emailInput = screen.getByLabelText('Email');
    const passwordInput = screen.getByLabelText('Password');
    const submitButton = screen.getByRole('button', { name: 'Sign In' });

    await user.type(emailInput, 'user@example.com');
    await user.type(passwordInput, 'securepassword');
    await user.press(submitButton);

    expect(mockSubmit).toHaveBeenCalledWith({
      email: 'user@example.com',
      password: 'securepassword',
    });
  });
});
```

### Scoped Queries with within()

```typescript
import { render, screen, within } from '@testing-library/react-native';
import { OrderList } from '@/components/OrderList';

it('should display item count in each order card', () => {
  render(<OrderList orders={mockOrders} />);

  const firstOrder = screen.getByTestId('order-card-1');
  expect(within(firstOrder).getByText('3 items')).toBeOnTheScreen();

  const secondOrder = screen.getByTestId('order-card-2');
  expect(within(secondOrder).getByText('1 item')).toBeOnTheScreen();
});
```

### Full Example: Login Form Component Test

```typescript
import { render, screen, waitFor } from '@testing-library/react-native';
import { userEvent } from '@testing-library/react-native';
import { LoginForm } from '@/components/LoginForm';

const mockOnSubmit = jest.fn();

function renderLoginForm() {
  return render(<LoginForm onSubmit={mockOnSubmit} />);
}

describe('LoginForm', () => {
  const user = userEvent.setup();

  beforeEach(() => {
    mockOnSubmit.mockClear();
  });

  it('should render all form fields and submit button', () => {
    renderLoginForm();

    expect(screen.getByLabelText('Email')).toBeOnTheScreen();
    expect(screen.getByLabelText('Password')).toBeOnTheScreen();
    expect(screen.getByRole('button', { name: 'Sign In' })).toBeOnTheScreen();
  });

  it('should display validation error for invalid email', async () => {
    renderLoginForm();

    await user.type(screen.getByLabelText('Email'), 'not-an-email');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(screen.getByText('Please enter a valid email')).toBeOnTheScreen();
    });
  });

  it('should display error when password is too short', async () => {
    renderLoginForm();

    await user.type(screen.getByLabelText('Email'), 'user@test.com');
    await user.type(screen.getByLabelText('Password'), '123');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(
        screen.getByText('Password must be at least 8 characters')
      ).toBeOnTheScreen();
    });
  });

  it('should submit form with valid data', async () => {
    renderLoginForm();

    await user.type(screen.getByLabelText('Email'), 'user@test.com');
    await user.type(screen.getByLabelText('Password'), 'password123');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        email: 'user@test.com',
        password: 'password123',
      });
    });
  });

  it('should disable button while submitting', async () => {
    mockOnSubmit.mockImplementation(
      () => new Promise((resolve) => setTimeout(resolve, 1000))
    );
    renderLoginForm();

    await user.type(screen.getByLabelText('Email'), 'user@test.com');
    await user.type(screen.getByLabelText('Password'), 'password123');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(screen.getByRole('button', { name: 'Sign In' })).toBeDisabled();
    });
  });
});
```

---

## Testing Hooks

### Custom Hooks with renderHook

```typescript
import { renderHook, act } from '@testing-library/react-native';
import { useCounter } from '@/hooks/useCounter';

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter(0));
    expect(result.current.count).toBe(0);
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useCounter(0));

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('should reset to initial value', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });

    expect(result.current.count).toBe(5);
  });
});
```

### Hooks with TanStack Query

```typescript
import { renderHook, waitFor } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';
import { useUser } from '@/hooks/useUser';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
    },
  });

  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

describe('useUser', () => {
  it('should fetch user data', async () => {
    const { result } = renderHook(() => useUser('123'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data?.name).toBe('John Doe');
  });
});
```

### Hooks with Redux

```typescript
import { renderHook } from '@testing-library/react-native';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import type { ReactNode } from 'react';
import { useAppUser } from '@/hooks/useAppUser';
import { authReducer } from '@/store/authSlice';

function createWrapper(preloadedState?: Partial<RootState>) {
  const store = configureStore({
    reducer: { auth: authReducer },
    preloadedState,
  });

  return function Wrapper({ children }: { children: ReactNode }) {
    return <Provider store={store}>{children}</Provider>;
  };
}

describe('useAppUser', () => {
  it('should return current user from store', () => {
    const { result } = renderHook(() => useAppUser(), {
      wrapper: createWrapper({
        auth: { user: { id: '1', name: 'Jane' }, token: 'abc' },
      }),
    });

    expect(result.current?.name).toBe('Jane');
  });
});
```

---

## Testing Redux

### Testing Slices (Reducers)

```typescript
import { authReducer, login, logout, setToken } from '@/store/authSlice';
import type { AuthState } from '@/store/authSlice';

describe('authSlice', () => {
  const initialState: AuthState = {
    user: null,
    token: null,
    isLoading: false,
  };

  it('should handle login.fulfilled', () => {
    const user = { id: '1', name: 'John', email: 'john@test.com' };
    const action = login.fulfilled({ user, token: 'jwt-token' }, 'requestId', {
      email: 'john@test.com',
      password: 'pass',
    });

    const state = authReducer(initialState, action);

    expect(state.user).toEqual(user);
    expect(state.token).toBe('jwt-token');
    expect(state.isLoading).toBe(false);
  });

  it('should handle logout', () => {
    const loggedInState: AuthState = {
      user: { id: '1', name: 'John', email: 'john@test.com' },
      token: 'jwt-token',
      isLoading: false,
    };

    const state = authReducer(loggedInState, logout());

    expect(state.user).toBeNull();
    expect(state.token).toBeNull();
  });
});
```

### Testing Async Thunks

```typescript
import { configureStore } from '@reduxjs/toolkit';
import { login, authReducer } from '@/store/authSlice';
import { httpService } from '@/services/httpService';

jest.mock('@/services/httpService');
const mockedHttp = jest.mocked(httpService);

describe('login thunk', () => {
  const store = configureStore({ reducer: { auth: authReducer } });

  it('should set user on successful login', async () => {
    mockedHttp.post.mockResolvedValueOnce({
      data: { user: { id: '1', name: 'John' }, token: 'abc' },
    });

    await store.dispatch(login({ email: 'john@test.com', password: 'pass' }));

    const state = store.getState().auth;
    expect(state.user?.name).toBe('John');
    expect(state.token).toBe('abc');
  });

  it('should handle login failure', async () => {
    mockedHttp.post.mockRejectedValueOnce(new Error('Invalid credentials'));

    await store.dispatch(login({ email: 'bad@test.com', password: 'wrong' }));

    const state = store.getState().auth;
    expect(state.user).toBeNull();
    expect(state.isLoading).toBe(false);
  });
});
```

### Testing Selectors

```typescript
import { selectCurrentUser, selectIsAuthenticated } from '@/store/authSlice';
import type { RootState } from '@/store';

describe('auth selectors', () => {
  const authenticatedState: RootState = {
    auth: {
      user: { id: '1', name: 'John', email: 'john@test.com' },
      token: 'jwt-token',
      isLoading: false,
    },
  };

  const unauthenticatedState: RootState = {
    auth: { user: null, token: null, isLoading: false },
  };

  it('should select current user', () => {
    expect(selectCurrentUser(authenticatedState)).toEqual({
      id: '1',
      name: 'John',
      email: 'john@test.com',
    });
  });

  it('should return true when user is authenticated', () => {
    expect(selectIsAuthenticated(authenticatedState)).toBe(true);
    expect(selectIsAuthenticated(unauthenticatedState)).toBe(false);
  });
});
```

---

## Testing TanStack Query Hooks

### Test Query Client

```typescript
// test/utils.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';

export function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
        staleTime: 0,
      },
      mutations: {
        retry: false,
      },
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

### Testing Query Hooks

```typescript
import { renderHook, waitFor } from '@testing-library/react-native';
import { createQueryWrapper } from '@/test/utils';
import { useProducts } from '@/hooks/useProducts';
import { httpService } from '@/services/httpService';

jest.mock('@/services/httpService');
const mockedHttp = jest.mocked(httpService);

describe('useProducts', () => {
  const wrapper = createQueryWrapper();

  it('should return loading state initially', () => {
    mockedHttp.get.mockReturnValue(new Promise(() => {})); // never resolves

    const { result } = renderHook(() => useProducts(), { wrapper });

    expect(result.current.isLoading).toBe(true);
    expect(result.current.data).toBeUndefined();
  });

  it('should return products on success', async () => {
    const products = [
      { id: '1', name: 'Widget', price: 9.99 },
      { id: '2', name: 'Gadget', price: 19.99 },
    ];
    mockedHttp.get.mockResolvedValueOnce({ data: products });

    const { result } = renderHook(() => useProducts(), { wrapper });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual(products);
  });

  it('should return error state on failure', async () => {
    mockedHttp.get.mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useProducts(), { wrapper });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error?.message).toBe('Network error');
  });
});
```

---

## Testing Navigation

### Mocking Navigation Hooks

```typescript
import { render, screen } from '@testing-library/react-native';
import { userEvent } from '@testing-library/react-native';
import { ProfileScreen } from '@/screens/ProfileScreen';

const mockNavigate = jest.fn();
const mockGoBack = jest.fn();

jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({
    navigate: mockNavigate,
    goBack: mockGoBack,
  }),
  useRoute: () => ({
    params: { userId: '123' },
  }),
}));

describe('ProfileScreen', () => {
  const user = userEvent.setup();

  beforeEach(() => {
    mockNavigate.mockClear();
    mockGoBack.mockClear();
  });

  it('should render user profile from route params', () => {
    render(<ProfileScreen />);
    expect(screen.getByText('User #123')).toBeOnTheScreen();
  });

  it('should navigate to settings on press', async () => {
    render(<ProfileScreen />);

    await user.press(screen.getByRole('button', { name: 'Settings' }));

    expect(mockNavigate).toHaveBeenCalledWith('Settings');
  });

  it('should go back on back button press', async () => {
    render(<ProfileScreen />);

    await user.press(screen.getByRole('button', { name: 'Back' }));

    expect(mockGoBack).toHaveBeenCalled();
  });
});
```

### Mocking Expo Router

```typescript
jest.mock('expo-router', () => ({
  useRouter: () => ({
    push: mockPush,
    back: mockBack,
    replace: mockReplace,
  }),
  useLocalSearchParams: () => ({ id: '456' }),
  Link: ({ children }: { children: React.ReactNode }) => children,
}));
```

---

## Coverage Configuration

Add coverage thresholds to `jest.config.ts`:

```typescript
const config: Config = {
  // ... other config
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  coveragePathIgnorePatterns: [
    '/node_modules/',
    '/__tests__/',
    '/test/',
    '\\.d\\.ts$',
    'index\\.ts$',       // barrel exports
    '\\.types\\.ts$',    // type-only files
    '\\.styles\\.ts$',   // style-only files
  ],
  coverageReporters: ['text', 'text-summary', 'lcov', 'json-summary'],
};
```

### CI Integration

```yaml
# In your CI pipeline
- name: Run tests with coverage
  run: npx jest --coverage --ci

- name: Check coverage thresholds
  run: npx jest --coverage --coverageThreshold='{"global":{"lines":80}}'
```

---

## Best Practices

### Query Preferences

```typescript
// GOOD: Query by role — tests what the user sees
screen.getByRole('button', { name: 'Submit' });

// GOOD: Query by label — tests accessibility
screen.getByLabelText('Email address');

// GOOD: Query by text — tests visible content
screen.getByText('Welcome back');

// BAD: Query by testID — doesn't test accessibility, brittle
screen.getByTestId('submit-btn');
```

### Event Preferences

```typescript
// GOOD: userEvent simulates real user interaction
const user = userEvent.setup();
await user.press(button);
await user.type(input, 'hello');

// BAD: fireEvent is synchronous and skips intermediate events
fireEvent.press(button);
fireEvent.changeText(input, 'hello');
```

### Test Behavior, Not Implementation

```typescript
// GOOD: Tests what the user sees
it('should display error when form is invalid', async () => {
  renderLoginForm();
  await user.press(screen.getByRole('button', { name: 'Sign In' }));
  expect(screen.getByText('Email is required')).toBeOnTheScreen();
});

// BAD: Tests implementation details
it('should call setError with email required', async () => {
  const setError = jest.fn();
  // ... testing internal state setter
  expect(setError).toHaveBeenCalledWith('email', 'required');
});
```

### Test Naming Convention

```typescript
// GOOD: Describes behavior from user perspective
it('should display error when form is invalid');
it('should navigate to home after successful login');
it('should disable submit button while loading');

// BAD: Vague or implementation-focused
it('works correctly');
it('calls the handler');
it('updates state');
```

### AAA Pattern (Arrange, Act, Assert)

```typescript
it('should add item to cart', async () => {
  // Arrange
  const user = userEvent.setup();
  render(<ProductCard product={mockProduct} />);

  // Act
  await user.press(screen.getByRole('button', { name: 'Add to Cart' }));

  // Assert
  expect(screen.getByText('Added to cart')).toBeOnTheScreen();
});
```

### Test Isolation

```typescript
// GOOD: Each test creates its own state
describe('CartScreen', () => {
  let store: ReturnType<typeof createTestStore>;

  beforeEach(() => {
    store = createTestStore(); // fresh store per test
  });

  it('should display empty cart message', () => {
    render(<CartScreen />, { wrapper: createWrapper(store) });
    expect(screen.getByText('Your cart is empty')).toBeOnTheScreen();
  });
});

// BAD: Shared mutable state between tests
const store = createTestStore(); // shared — tests can leak state
```

---

## Related Resources

- [Test Mocking Patterns](../skills/debugging/test-mocking-patterns.md) — comprehensive mock recipes
- [Mobile Testing Guide](./mobile-testing.md) — integration and device testing
- [E2E Test Generator](../skills/e2e-testing/e2e-test-generator.md) — automated E2E test creation
