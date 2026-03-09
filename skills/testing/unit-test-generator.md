# Unit Test Generator

Guide for generating unit tests for React Native components, hooks, and screens using Jest + React Native Testing Library.

---

## When to Generate Tests

This guide activates when:
- Creating or modifying components, hooks, or screens
- Before releases or after major refactors
- When asked to "write unit tests" or "test this component"
- Adding new business logic that needs regression coverage

---

## Test Infrastructure

### Directory Structure

```
src/
├── components/
│   ├── Button.tsx
│   └── __tests__/
│       └── Button.test.tsx
├── hooks/
│   ├── useAuth.ts
│   └── __tests__/
│       └── useAuth.test.ts
├── screens/
│   ├── LoginScreen.tsx
│   └── __tests__/
│       └── LoginScreen.test.tsx
├── utils/
│   ├── formatters.ts
│   └── __tests__/
│       └── formatters.test.ts
└── test/
    ├── jest.setup.ts          # Global mocks and setup
    ├── test-utils.tsx         # renderWithProviders, wrappers
    └── mocks/                 # Shared mock factories
        ├── handlers.ts
        └── fixtures.ts
```

### Jest Configuration

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  preset: 'jest-expo',
  setupFilesAfterSetup: ['<rootDir>/src/test/jest.setup.ts'],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg|react-native-reanimated|react-native-gesture-handler|react-native-mmkv|nativewind)',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@screens/(.*)$': '<rootDir>/src/screens/$1',
    '^@services/(.*)$': '<rootDir>/src/services/$1',
    '^@store/(.*)$': '<rootDir>/src/store/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
    '!src/test/**/*',
  ],
  testMatch: ['**/__tests__/**/*.test.{ts,tsx}'],
};

export default config;
```

### Test Setup File

```typescript
// src/test/jest.setup.ts
import '@testing-library/react-native/extend-expect';

// See test-mocking-patterns.md for comprehensive mock setup
// Minimal required mocks below:

jest.mock('react-native-reanimated', () =>
  require('react-native-reanimated/mock')
);

jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    getString: jest.fn(),
    set: jest.fn(),
    delete: jest.fn(),
    contains: jest.fn(),
    getAllKeys: jest.fn().mockReturnValue([]),
  })),
}));

// Silence act() warnings in test output
jest.spyOn(console, 'error').mockImplementation((...args: unknown[]) => {
  const message = typeof args[0] === 'string' ? args[0] : '';
  if (message.includes('act(')) return;
  console.warn(...args);
});
```

---

## Component Test Patterns

### Presentational Component Test

```typescript
// src/components/__tests__/UserCard.test.tsx
import { screen, render } from '@testing-library/react-native';
import { UserCard } from '../UserCard';

const mockUser = {
  id: '1',
  name: 'Jane Doe',
  email: 'jane@example.com',
  avatarUrl: 'https://example.com/avatar.png',
};

describe('UserCard', () => {
  it('renders user information', () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText('Jane Doe')).toBeOnTheScreen();
    expect(screen.getByText('jane@example.com')).toBeOnTheScreen();
  });

  it('renders avatar image with correct source', () => {
    render(<UserCard user={mockUser} />);

    const avatar = screen.getByRole('image', { name: 'User avatar' });
    expect(avatar.props.source).toEqual({ uri: 'https://example.com/avatar.png' });
  });

  it('renders fallback when no avatar provided', () => {
    render(<UserCard user={{ ...mockUser, avatarUrl: undefined }} />);

    expect(screen.getByText('JD')).toBeOnTheScreen(); // initials fallback
  });

  it('calls onPress when card is pressed', async () => {
    const onPress = jest.fn();
    const user = require('@testing-library/user-event').userEvent.setup();

    render(<UserCard user={mockUser} onPress={onPress} />);

    await user.press(screen.getByRole('button', { name: 'Jane Doe' }));
    expect(onPress).toHaveBeenCalledWith(mockUser.id);
  });
});
```

### Form Component Test

```typescript
// src/components/__tests__/LoginForm.test.tsx
import { screen, render, waitFor } from '@testing-library/react-native';
import { userEvent } from '@testing-library/user-event';
import { LoginForm } from '../LoginForm';

describe('LoginForm', () => {
  const mockOnSubmit = jest.fn();
  const user = userEvent.setup();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders email and password fields', () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);

    expect(screen.getByLabelText('Email')).toBeOnTheScreen();
    expect(screen.getByLabelText('Password')).toBeOnTheScreen();
    expect(screen.getByRole('button', { name: 'Sign In' })).toBeOnTheScreen();
  });

  it('shows validation error for invalid email', async () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'not-an-email');
    await user.type(screen.getByLabelText('Password'), 'password123');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(screen.getByText('Invalid email address')).toBeOnTheScreen();
    });
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('submits form with valid data', async () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'jane@example.com');
    await user.type(screen.getByLabelText('Password'), 'securePass1!');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        email: 'jane@example.com',
        password: 'securePass1!',
      });
    });
  });

  it('disables submit button while submitting', async () => {
    mockOnSubmit.mockImplementation(
      () => new Promise((resolve) => setTimeout(resolve, 1000))
    );
    render(<LoginForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText('Email'), 'jane@example.com');
    await user.type(screen.getByLabelText('Password'), 'securePass1!');
    await user.press(screen.getByRole('button', { name: 'Sign In' }));

    expect(screen.getByRole('button', { name: 'Sign In' })).toBeDisabled();
  });
});
```

### Screen Test with Navigation

```typescript
// src/screens/__tests__/ProfileScreen.test.tsx
import { screen, render } from '@testing-library/react-native';
import { userEvent } from '@testing-library/user-event';
import { ProfileScreen } from '../ProfileScreen';

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

// For Expo Router projects, mock useLocalSearchParams instead:
// jest.mock('expo-router', () => ({
//   useLocalSearchParams: () => ({ userId: '123' }),
//   useRouter: () => ({ push: mockNavigate, back: mockGoBack }),
// }));

describe('ProfileScreen', () => {
  const user = userEvent.setup();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders user profile for given userId', async () => {
    render(<ProfileScreen />);

    expect(await screen.findByText('Jane Doe')).toBeOnTheScreen();
    expect(screen.getByText('jane@example.com')).toBeOnTheScreen();
  });

  it('navigates to edit screen on edit button press', async () => {
    render(<ProfileScreen />);

    await user.press(
      await screen.findByRole('button', { name: 'Edit Profile' })
    );

    expect(mockNavigate).toHaveBeenCalledWith('EditProfile', { userId: '123' });
  });

  it('navigates back on back button press', async () => {
    render(<ProfileScreen />);

    await user.press(await screen.findByRole('button', { name: 'Back' }));
    expect(mockGoBack).toHaveBeenCalled();
  });
});
```

---

## Hook Test Patterns

### Custom Hook with useState/useEffect

```typescript
// src/hooks/__tests__/useDebounce.test.ts
import { renderHook, act } from '@testing-library/react-native';
import { useDebounce } from '../useDebounce';

describe('useDebounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 500));

    expect(result.current).toBe('hello');
  });

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'hello', delay: 500 } }
    );

    rerender({ value: 'world', delay: 500 });
    expect(result.current).toBe('hello'); // not yet updated

    act(() => {
      jest.advanceTimersByTime(500);
    });

    expect(result.current).toBe('world');
  });

  it('resets timer on rapid changes', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 300),
      { initialProps: { value: 'a' } }
    );

    rerender({ value: 'ab' });
    act(() => jest.advanceTimersByTime(200));

    rerender({ value: 'abc' });
    act(() => jest.advanceTimersByTime(200));

    expect(result.current).toBe('a'); // still waiting

    act(() => jest.advanceTimersByTime(100));
    expect(result.current).toBe('abc');
  });
});
```

### TanStack Query Hook Test

```typescript
// src/hooks/__tests__/useUser.test.tsx
import { renderHook, waitFor } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';
import { useUser } from '../useUser';
import { httpService } from '@/services/httpService';

jest.mock('@/services/httpService');
const mockedHttpService = jest.mocked(httpService);

function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: Infinity },
      mutations: { retry: false },
    },
  });
}

function createWrapper() {
  const queryClient = createTestQueryClient();
  return function Wrapper({ children }: { children: ReactNode }) {
    return (
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    );
  };
}

describe('useUser', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('fetches user data successfully', async () => {
    mockedHttpService.get.mockResolvedValueOnce({
      data: { id: '1', name: 'Jane Doe', email: 'jane@example.com' },
    });

    const { result } = renderHook(() => useUser('1'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual({
      id: '1',
      name: 'Jane Doe',
      email: 'jane@example.com',
    });
    expect(mockedHttpService.get).toHaveBeenCalledWith('/users/1');
  });

  it('handles fetch error', async () => {
    mockedHttpService.get.mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useUser('1'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error).toEqual(new Error('Network error'));
  });
});
```

### Redux Hook Test

```typescript
// src/hooks/__tests__/useAuthStatus.test.tsx
import { renderHook } from '@testing-library/react-native';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import type { ReactNode } from 'react';
import { useAuthStatus } from '../useAuthStatus';
import { authReducer } from '@/store/authSlice';

function renderWithStore(
  hook: () => ReturnType<typeof useAuthStatus>,
  preloadedState: Record<string, unknown> = {}
) {
  const store = configureStore({
    reducer: { auth: authReducer },
    preloadedState,
  });

  function Wrapper({ children }: { children: ReactNode }) {
    return <Provider store={store}>{children}</Provider>;
  }

  return renderHook(hook, { wrapper: Wrapper });
}

describe('useAuthStatus', () => {
  it('returns authenticated when token exists', () => {
    const { result } = renderWithStore(() => useAuthStatus(), {
      auth: { token: 'valid-token', user: { id: '1', name: 'Jane' } },
    });

    expect(result.current.isAuthenticated).toBe(true);
    expect(result.current.user?.name).toBe('Jane');
  });

  it('returns unauthenticated when no token', () => {
    const { result } = renderWithStore(() => useAuthStatus(), {
      auth: { token: null, user: null },
    });

    expect(result.current.isAuthenticated).toBe(false);
    expect(result.current.user).toBeNull();
  });
});
```

---

## Mocking Patterns

For comprehensive mocking setup including native module mocks, navigation mocks, storage mocks, and test utilities, see [test-mocking-patterns.md](./test-mocking-patterns.md).

---

## GOOD vs BAD Practices

### Query Strategy

- **GOOD:** `screen.getByRole('button', { name: 'Submit' })` -- accessible, resilient to layout changes
- **BAD:** `screen.getByTestId('submit-btn')` as primary query -- not accessible, brittle

### User Interaction

- **GOOD:** `await userEvent.press(button)` -- simulates real user behavior
- **BAD:** `fireEvent.press(button)` -- skips intermediate events, less realistic

### Test Focus

- **GOOD:** Test behavior and output (what the user sees and does)
- **BAD:** Test implementation details (internal state, private methods)

### Assertion Scope

- **GOOD:** One logical assertion per test -- clear failure messages
- **BAD:** Multiple unrelated assertions in one test -- unclear what failed

### Mock Scope

- **GOOD:** Mock at module boundaries (services, navigation, storage)
- **BAD:** Mock internal component functions -- couples tests to implementation

### Async Handling

- **GOOD:** `await waitFor(() => expect(...))` -- waits for async updates
- **BAD:** `setTimeout` or manual delays -- flaky, slow tests

---

## Coverage Configuration

```typescript
// In jest.config.ts, add:
const config: Config = {
  // ...existing config
  coverageThresholds: {
    global: {
      branches: 70,
      functions: 75,
      lines: 80,
      statements: 80,
    },
  },
  coverageReporters: ['text', 'text-summary', 'lcov', 'clover'],
};
```

### Running Coverage

```bash
# Generate coverage report
npx jest --coverage

# Run coverage for specific directory
npx jest --coverage --collectCoverageFrom='src/components/**/*.{ts,tsx}'

# CI integration -- fail if below thresholds
npx jest --coverage --ci
```

---

## Related Resources

- [test-mocking-patterns.md](./test-mocking-patterns.md) -- Comprehensive mocking reference
- [mobile-testing.md](../../guides/mobile-testing.md) -- Mobile testing strategies guide
- [e2e-test-generator.md](../e2e-testing/e2e-test-generator.md) -- E2E test generation with Detox and Maestro
