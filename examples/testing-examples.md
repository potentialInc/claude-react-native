# Testing Examples

Complete testing examples for React Native applications using Jest + React Native Testing Library and Maestro E2E flows.

---

## Example 1: Testing a Form Component

### Component Under Test

```typescript
// src/components/LoginForm.tsx
import { View, KeyboardAvoidingView, Platform } from 'react-native';
import { TextInput, Button, HelperText, Text } from 'react-native-paper';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const loginSchema = z.object({
  email: z.string().email('Please enter a valid email'),
  password: z.string().min(1, 'Password is required'),
});

type LoginFormData = z.infer<typeof loginSchema>;

interface LoginFormProps {
  onSubmit: (data: LoginFormData) => Promise<void>;
  serverError?: string | null;
}

export function LoginForm({ onSubmit, serverError }: LoginFormProps) {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      className="flex-1"
    >
      <View className="flex-1 justify-center p-6 gap-4">
        {serverError && (
          <View className="bg-destructive/10 p-4 rounded-lg" testID="server-error">
            <Text className="text-destructive">{serverError}</Text>
          </View>
        )}

        <Controller
          control={control}
          name="email"
          render={({ field: { onChange, onBlur, value } }) => (
            <View>
              <TextInput
                testID="email-input"
                label="Email"
                mode="outlined"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={!!errors.email}
                keyboardType="email-address"
                autoCapitalize="none"
              />
              <HelperText type="error" visible={!!errors.email} testID="email-error">
                {errors.email?.message}
              </HelperText>
            </View>
          )}
        />

        <Controller
          control={control}
          name="password"
          render={({ field: { onChange, onBlur, value } }) => (
            <View>
              <TextInput
                testID="password-input"
                label="Password"
                mode="outlined"
                value={value}
                onChangeText={onChange}
                onBlur={onBlur}
                error={!!errors.password}
                secureTextEntry
              />
              <HelperText type="error" visible={!!errors.password} testID="password-error">
                {errors.password?.message}
              </HelperText>
            </View>
          )}
        />

        <Button
          testID="submit-button"
          mode="contained"
          onPress={handleSubmit(onSubmit)}
          loading={isSubmitting}
          disabled={isSubmitting}
        >
          {isSubmitting ? 'Signing In...' : 'Sign In'}
        </Button>
      </View>
    </KeyboardAvoidingView>
  );
}
```

### Test File

```typescript
// src/components/__tests__/LoginForm.test.tsx
import { render, screen, userEvent, waitFor } from '@testing-library/react-native';
import { PaperProvider } from 'react-native-paper';
import { LoginForm } from '../LoginForm';

function renderLoginForm(props: Partial<React.ComponentProps<typeof LoginForm>> = {}) {
  const defaultProps = {
    onSubmit: jest.fn().mockResolvedValue(undefined),
    serverError: null,
    ...props,
  };

  return render(
    <PaperProvider>
      <LoginForm {...defaultProps} />
    </PaperProvider>
  );
}

describe('LoginForm', () => {
  const user = userEvent.setup();

  it('renders email and password inputs', () => {
    renderLoginForm();

    expect(screen.getByTestId('email-input')).toBeOnTheScreen();
    expect(screen.getByTestId('password-input')).toBeOnTheScreen();
    expect(screen.getByTestId('submit-button')).toBeOnTheScreen();
  });

  it('shows validation errors on empty submit', async () => {
    renderLoginForm();

    await user.press(screen.getByTestId('submit-button'));

    await waitFor(() => {
      expect(screen.getByTestId('email-error')).toHaveTextContent('Please enter a valid email');
      expect(screen.getByTestId('password-error')).toHaveTextContent('Password is required');
    });
  });

  it('calls onSubmit with valid credentials', async () => {
    const onSubmit = jest.fn().mockResolvedValue(undefined);
    renderLoginForm({ onSubmit });

    await user.type(screen.getByTestId('email-input'), 'user@example.com');
    await user.type(screen.getByTestId('password-input'), 'securepassword');
    await user.press(screen.getByTestId('submit-button'));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith(
        { email: 'user@example.com', password: 'securepassword' },
        expect.anything()
      );
    });
  });

  it('shows loading state during submission', async () => {
    const onSubmit = jest.fn(
      () => new Promise<void>((resolve) => setTimeout(resolve, 1000))
    );
    renderLoginForm({ onSubmit });

    await user.type(screen.getByTestId('email-input'), 'user@example.com');
    await user.type(screen.getByTestId('password-input'), 'password123');
    await user.press(screen.getByTestId('submit-button'));

    await waitFor(() => {
      expect(screen.getByTestId('submit-button')).toBeDisabled();
      expect(screen.getByText('Signing In...')).toBeOnTheScreen();
    });
  });

  it('displays server error message', () => {
    renderLoginForm({ serverError: 'Invalid credentials' });

    expect(screen.getByTestId('server-error')).toHaveTextContent('Invalid credentials');
  });
});
```

---

## Example 2: Testing a Custom Hook with TanStack Query

### Hook Under Test

```typescript
// src/hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';
import { httpService } from '~/services/httpService';

interface User {
  id: string;
  name: string;
  email: string;
}

export function useUser(id: string) {
  return useQuery({
    queryKey: ['user', id],
    queryFn: async () => {
      const response = await httpService.get<User>(`/users/${id}`);
      return response.data;
    },
    enabled: !!id,
  });
}
```

### Test File

```typescript
// src/hooks/__tests__/useUser.test.ts
import { renderHook, waitFor } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import type { ReactNode } from 'react';
import { httpService } from '~/services/httpService';
import { useUser } from '../useUser';

jest.mock('~/services/httpService');
const mockedHttpService = jest.mocked(httpService);

function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
      },
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
  afterEach(() => {
    jest.clearAllMocks();
  });

  it('returns loading state initially', () => {
    mockedHttpService.get.mockReturnValue(new Promise(() => {}));

    const { result } = renderHook(() => useUser('user-1'), {
      wrapper: createWrapper(),
    });

    expect(result.current.isLoading).toBe(true);
    expect(result.current.data).toBeUndefined();
  });

  it('returns user data on success', async () => {
    const mockUser = { id: 'user-1', name: 'Jane Doe', email: 'jane@example.com' };
    mockedHttpService.get.mockResolvedValueOnce({ data: mockUser });

    const { result } = renderHook(() => useUser('user-1'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual(mockUser);
    expect(mockedHttpService.get).toHaveBeenCalledWith('/users/user-1');
  });

  it('returns error on failure', async () => {
    mockedHttpService.get.mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useUser('user-1'), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error).toEqual(new Error('Network error'));
  });
});
```

---

## Example 3: Testing a Redux-Connected Screen

### Screen Under Test

```typescript
// src/screens/ProductListScreen.tsx
import { View, FlatList, Pressable } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Text, Card } from 'react-native-paper';
import { useNavigation } from '@react-navigation/native';
import { useAppSelector } from '~/redux/store/hooks';
import type { Product } from '~/types/product';

export function ProductListScreen() {
  const navigation = useNavigation();
  const products = useAppSelector((state) => state.product.items);

  if (products.length === 0) {
    return (
      <SafeAreaView className="flex-1 bg-background">
        <View className="flex-1 items-center justify-center p-8" testID="empty-state">
          <Text className="text-muted-foreground text-lg">No products available</Text>
        </View>
      </SafeAreaView>
    );
  }

  return (
    <SafeAreaView className="flex-1 bg-background">
      <FlatList
        testID="product-list"
        data={products}
        keyExtractor={(item) => item.id}
        contentContainerClassName="p-4 gap-3"
        renderItem={({ item }) => (
          <Pressable
            testID={`product-item-${item.id}`}
            onPress={() => navigation.navigate('ProductDetail' as never, { id: item.id } as never)}
          >
            <Card>
              <Card.Title title={item.name} subtitle={`$${item.price.toFixed(2)}`} />
            </Card>
          </Pressable>
        )}
      />
    </SafeAreaView>
  );
}
```

### Test File

```typescript
// src/screens/__tests__/ProductListScreen.test.tsx
import { render, screen, userEvent } from '@testing-library/react-native';
import { ProductListScreen } from '../ProductListScreen';
import { renderWithProviders } from '~/test/test-utils';
import type { Product } from '~/types/product';

const mockNavigate = jest.fn();
jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({ navigate: mockNavigate }),
}));

const mockProducts: Product[] = [
  { id: 'p1', name: 'Wireless Headphones', price: 79.99, category: 'electronics' },
  { id: 'p2', name: 'Running Shoes', price: 129.99, category: 'sports' },
  { id: 'p3', name: 'Coffee Maker', price: 49.99, category: 'kitchen' },
];

describe('ProductListScreen', () => {
  const user = userEvent.setup();

  afterEach(() => {
    jest.clearAllMocks();
  });

  it('renders the product list', () => {
    renderWithProviders(<ProductListScreen />, {
      preloadedState: { product: { items: mockProducts } },
    });

    expect(screen.getByTestId('product-list')).toBeOnTheScreen();
    expect(screen.getByText('Wireless Headphones')).toBeOnTheScreen();
    expect(screen.getByText('$79.99')).toBeOnTheScreen();
    expect(screen.getByText('Running Shoes')).toBeOnTheScreen();
    expect(screen.getByText('Coffee Maker')).toBeOnTheScreen();
  });

  it('navigates to product detail on item press', async () => {
    renderWithProviders(<ProductListScreen />, {
      preloadedState: { product: { items: mockProducts } },
    });

    await user.press(screen.getByTestId('product-item-p1'));

    expect(mockNavigate).toHaveBeenCalledWith('ProductDetail', { id: 'p1' });
  });

  it('shows empty state when no products exist', () => {
    renderWithProviders(<ProductListScreen />, {
      preloadedState: { product: { items: [] } },
    });

    expect(screen.getByTestId('empty-state')).toBeOnTheScreen();
    expect(screen.getByText('No products available')).toBeOnTheScreen();
    expect(screen.queryByTestId('product-list')).not.toBeOnTheScreen();
  });
});
```

---

## Example 4: E2E Test with Maestro

```yaml
# e2e/login-and-browse.yaml
# Complete Maestro flow: login -> navigate -> interact -> verify
appId: com.example.app
---
- launchApp:
    clearState: true

# --- Login Flow ---
- assertVisible: "Sign In"

- tapOn:
    id: "email-input"
- inputText: "user@example.com"

- tapOn:
    id: "password-input"
- inputText: "securepassword"

- tapOn:
    id: "submit-button"

# --- Verify Home Screen ---
- assertVisible: "Home"
- assertVisible:
    id: "product-list"

# --- Navigate to Product Detail ---
- tapOn: "Wireless Headphones"
- assertVisible: "Product Details"
- assertVisible: "$79.99"

# --- Add to Cart ---
- tapOn:
    id: "add-to-cart-button"
- assertVisible: "Added to cart"

# --- Navigate to Cart ---
- tapOn:
    id: "cart-tab"
- assertVisible: "Your Cart"
- assertVisible: "Wireless Headphones"
- assertVisible: "1 item"

# --- Verify Total ---
- assertVisible: "$79.99"

# --- Pull to Refresh ---
- scroll:
    direction: DOWN
    duration: 1000

# --- Go Back to Home ---
- tapOn:
    id: "home-tab"
- assertVisible: "Home"
```

---

## Example 5: Test Utilities Setup

```typescript
// src/test/test-utils.tsx
import type { ReactElement, ReactNode } from 'react';
import { render, type RenderOptions } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { PaperProvider } from 'react-native-paper';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Provider as ReduxProvider } from 'react-redux';
import { configureStore, type PreloadedState } from '@reduxjs/toolkit';
import type { RootState, AppStore } from '~/redux/store';
import { rootReducer } from '~/redux/store';

/**
 * Create a QueryClient configured for testing.
 * Disables retries and garbage collection to make tests deterministic.
 */
export function createTestQueryClient(): QueryClient {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: 0,
      },
      mutations: {
        retry: false,
      },
    },
  });
}

/**
 * Create mock navigation object for testing components that use useNavigation.
 */
export function createMockNavigation() {
  return {
    navigate: jest.fn(),
    goBack: jest.fn(),
    reset: jest.fn(),
    setOptions: jest.fn(),
    setParams: jest.fn(),
    dispatch: jest.fn(),
    canGoBack: jest.fn().mockReturnValue(true),
    addListener: jest.fn().mockReturnValue(jest.fn()),
    removeListener: jest.fn(),
    isFocused: jest.fn().mockReturnValue(true),
    getParent: jest.fn(),
    getState: jest.fn(),
    getId: jest.fn(),
  };
}

interface ExtendedRenderOptions extends Omit<RenderOptions, 'wrapper'> {
  preloadedState?: PreloadedState<RootState>;
  store?: AppStore;
  queryClient?: QueryClient;
}

/**
 * Render a component with all required providers for testing:
 * - Redux store (with optional preloaded state)
 * - TanStack Query client
 * - React Native Paper theme
 * - Safe Area provider
 * - Navigation container
 */
export function renderWithProviders(
  ui: ReactElement,
  {
    preloadedState = {} as PreloadedState<RootState>,
    store = configureStore({
      reducer: rootReducer,
      preloadedState,
    }),
    queryClient = createTestQueryClient(),
    ...renderOptions
  }: ExtendedRenderOptions = {}
) {
  function AllProviders({ children }: { children: ReactNode }) {
    return (
      <ReduxProvider store={store}>
        <QueryClientProvider client={queryClient}>
          <PaperProvider>
            <SafeAreaProvider
              initialMetrics={{
                frame: { x: 0, y: 0, width: 375, height: 812 },
                insets: { top: 47, left: 0, right: 0, bottom: 34 },
              }}
            >
              <NavigationContainer>
                {children}
              </NavigationContainer>
            </SafeAreaProvider>
          </PaperProvider>
        </QueryClientProvider>
      </ReduxProvider>
    );
  }

  return {
    store,
    queryClient,
    ...render(ui, { wrapper: AllProviders, ...renderOptions }),
  };
}
```

---

## Summary

These testing examples cover:

1. **Component Testing** -- Form validation, user interaction, loading/error states
2. **Hook Testing** -- TanStack Query hooks with mocked HTTP layer
3. **Integration Testing** -- Redux-connected screens with navigation
4. **E2E Testing** -- Maestro YAML flows for full user journeys
5. **Test Utilities** -- Reusable `renderWithProviders` and helper functions

**See Also:**
- [complete-examples.md](./complete-examples.md) -- Full screen implementations
- [../guides/component-patterns.md](../guides/component-patterns.md) -- Component architecture
- [../guides/data-fetching.md](../guides/data-fetching.md) -- TanStack Query patterns
