# Error Handling Guide

Patterns for handling errors gracefully in React Native applications — from global error boundaries to network failures and form validation.

## Table of Contents

- [Error Boundary Component](#error-boundary-component)
- [Global Error Handler](#global-error-handler)
- [Network Error Handling](#network-error-handling)
- [Form Validation Errors](#form-validation-errors)
- [Navigation Errors](#navigation-errors)
- [Crash Reporting](#crash-reporting)
- [Retry and Fallback Patterns](#retry-and-fallback-patterns)

---

## Error Boundary Component

React error boundaries catch JavaScript errors in the component tree and display fallback UI instead of crashing the app.

### Implementation

```tsx
import React from 'react';
import { View, Text, Pressable } from 'react-native';

interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  onError?: (error: Error, errorInfo: React.ErrorInfo) => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    this.props.onError?.(error, errorInfo);
  }

  handleRetry = (): void => {
    this.setState({ hasError: false, error: null });
  };

  render(): React.ReactNode {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <View className="flex-1 items-center justify-center p-6">
          <Text className="mb-2 text-xl font-bold text-gray-900">
            Something went wrong
          </Text>
          <Text className="mb-6 text-center text-gray-600">
            {this.state.error?.message ?? 'An unexpected error occurred'}
          </Text>
          <Pressable
            onPress={this.handleRetry}
            accessibilityRole="button"
            accessibilityLabel="Try again"
            className="rounded-lg bg-blue-600 px-6 py-3"
          >
            <Text className="font-semibold text-white">Try Again</Text>
          </Pressable>
        </View>
      );
    }

    return this.props.children;
  }
}

export { ErrorBoundary };
```

### Usage: Screen-Level vs Section-Level

```tsx
// Screen-level: wraps entire screen content
function App() {
  return (
    <ErrorBoundary onError={reportCrash}>
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </ErrorBoundary>
  );
}

// Section-level: wraps a specific section that might fail
function DashboardScreen() {
  return (
    <View className="flex-1">
      <ScreenHeader title="Dashboard" />

      {/* Charts section can fail without crashing the whole screen */}
      <ErrorBoundary fallback={<ChartErrorFallback />}>
        <AnalyticsCharts />
      </ErrorBoundary>

      {/* Rest of the screen still renders */}
      <RecentActivity />
    </View>
  );
}
```

### Reset on Navigation

```tsx
import { useNavigationState } from '@react-navigation/native';

function NavigationAwareErrorBoundary({ children }: { children: React.ReactNode }) {
  // Key changes on navigation, forcing boundary to remount and clear errors
  const navigationKey = useNavigationState((state) => state.index);

  return (
    <ErrorBoundary key={navigationKey}>
      {children}
    </ErrorBoundary>
  );
}
```

---

## Global Error Handler

### Unhandled JavaScript Errors

```typescript
// app/error-handler.ts
import { ErrorUtils } from 'react-native';

type ErrorHandler = (error: Error, isFatal: boolean) => void;

export function setupGlobalErrorHandler(reportError: ErrorHandler): void {
  const previousHandler = ErrorUtils.getGlobalHandler();

  ErrorUtils.setGlobalHandler((error: Error, isFatal: boolean) => {
    // Report to crash tracking
    reportError(error, isFatal);

    // Call the previous handler (React Native's default)
    previousHandler(error, isFatal);
  });
}
```

### Unhandled Promise Rejections

```typescript
// app/promise-handler.ts
export function setupPromiseRejectionHandler(
  reportError: (error: Error) => void
): void {
  const tracking = require('promise/setimmediate/rejection-tracking');

  tracking.enable({
    allRejections: true,
    onUnhandled: (_id: number, error: Error) => {
      reportError(error);
    },
  });
}
```

### Integration Point

```typescript
// app/_layout.tsx or App.tsx
import { setupGlobalErrorHandler } from './error-handler';
import { setupPromiseRejectionHandler } from './promise-handler';
import { reportCrash } from '@/services/crashReporting';

setupGlobalErrorHandler((error, isFatal) => {
  reportCrash(error, { isFatal });
});

setupPromiseRejectionHandler((error) => {
  reportCrash(error, { isFatal: false, type: 'unhandled-promise' });
});
```

---

## Network Error Handling

### Axios Interceptors in httpService

```typescript
// services/httpService.ts
import axios, { AxiosError, AxiosResponse, InternalAxiosRequestConfig } from 'axios';
import NetInfo from '@react-native-community/netinfo';

const httpService = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 15000,
});

// Response error interceptor
httpService.interceptors.response.use(
  (response: AxiosResponse) => response,
  async (error: AxiosError<ApiErrorResponse>) => {
    // Check for network connectivity
    const netState = await NetInfo.fetch();
    if (!netState.isConnected) {
      return Promise.reject(new AppError(
        'No internet connection. Please check your network.',
        'NETWORK_OFFLINE'
      ));
    }

    // Handle specific HTTP status codes
    if (error.response) {
      const { status, data } = error.response;

      switch (status) {
        case 401:
          // Token expired — trigger auth refresh or logout
          await handleUnauthorized();
          break;
        case 403:
          return Promise.reject(new AppError(
            'You do not have permission to perform this action.',
            'FORBIDDEN'
          ));
        case 422:
          // Validation errors from server
          return Promise.reject(new ValidationError(
            data.errors ?? [],
            'Please check your input and try again.'
          ));
        case 429:
          return Promise.reject(new AppError(
            'Too many requests. Please try again later.',
            'RATE_LIMIT'
          ));
        case 500:
        case 502:
        case 503:
          return Promise.reject(new AppError(
            'Server error. Please try again later.',
            'SERVER_ERROR'
          ));
      }
    }

    // Timeout
    if (error.code === 'ECONNABORTED') {
      return Promise.reject(new AppError(
        'Request timed out. Please try again.',
        'TIMEOUT'
      ));
    }

    return Promise.reject(error);
  }
);
```

### Custom Error Classes

```typescript
// types/errors.ts
export class AppError extends Error {
  code: string;

  constructor(message: string, code: string) {
    super(message);
    this.name = 'AppError';
    this.code = code;
  }
}

export class ValidationError extends Error {
  errors: FieldError[];

  constructor(errors: FieldError[], message: string) {
    super(message);
    this.name = 'ValidationError';
    this.errors = errors;
  }
}

interface FieldError {
  field: string;
  message: string;
}
```

### Offline Detection

```typescript
import { useEffect, useState } from 'react';
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';

export function useNetworkStatus(): { isConnected: boolean; isInternetReachable: boolean } {
  const [status, setStatus] = useState({
    isConnected: true,
    isInternetReachable: true,
  });

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state: NetInfoState) => {
      setStatus({
        isConnected: state.isConnected ?? false,
        isInternetReachable: state.isInternetReachable ?? false,
      });
    });

    return unsubscribe;
  }, []);

  return status;
}
```

```tsx
function OfflineBanner() {
  const { isConnected } = useNetworkStatus();

  if (isConnected) return null;

  return (
    <View className="bg-red-500 px-4 py-2" accessibilityLiveRegion="assertive">
      <Text className="text-center text-sm font-medium text-white">
        No internet connection
      </Text>
    </View>
  );
}
```

### TanStack Query Error Handling

```tsx
import { QueryErrorResetBoundary } from '@tanstack/react-query';

function ProductListScreen() {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          onReset={reset}
          fallback={
            <View className="flex-1 items-center justify-center p-6">
              <Text className="mb-4 text-gray-600">Failed to load products</Text>
              <Pressable
                onPress={reset}
                accessibilityRole="button"
                accessibilityLabel="Retry loading products"
                className="rounded-lg bg-blue-600 px-6 py-3"
              >
                <Text className="font-semibold text-white">Retry</Text>
              </Pressable>
            </View>
          }
        >
          <ProductList />
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

---

## Form Validation Errors

### Zod Schema with React Hook Form

```typescript
import { z } from 'zod';

export const registerSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Please enter a valid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain an uppercase letter')
    .regex(/[0-9]/, 'Password must contain a number'),
});

export type RegisterFormData = z.infer<typeof registerSchema>;
```

### Field-Level Error Display

```tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

function RegisterForm() {
  const {
    control,
    handleSubmit,
    setError,
    formState: { errors },
  } = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
    defaultValues: { name: '', email: '', password: '' },
  });

  const onSubmit = async (data: RegisterFormData): Promise<void> => {
    try {
      await registerUser(data);
    } catch (error) {
      if (error instanceof ValidationError) {
        // Map server validation errors to form fields
        error.errors.forEach((fieldError) => {
          setError(fieldError.field as keyof RegisterFormData, {
            message: fieldError.message,
          });
        });
      } else {
        // Form-level error (not field-specific)
        setError('root', {
          message: 'Registration failed. Please try again.',
        });
      }
    }
  };

  return (
    <View className="gap-4 p-4">
      {/* Form-level error */}
      {errors.root && (
        <View
          accessibilityRole="alert"
          className="rounded-lg bg-red-50 p-3"
        >
          <Text className="text-red-700">{errors.root.message}</Text>
        </View>
      )}

      {/* Field with error display */}
      <Controller
        control={control}
        name="email"
        render={({ field: { onChange, onBlur, value } }) => (
          <View>
            <Text nativeID="email-label" className="mb-1 font-medium text-gray-700">
              Email
            </Text>
            <TextInput
              accessibilityLabelledBy="email-label"
              accessibilityLabel="Email"
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
              keyboardType="email-address"
              autoCapitalize="none"
              className={`rounded-lg border px-4 py-3 ${
                errors.email ? 'border-red-500' : 'border-gray-300'
              }`}
            />
            {errors.email && (
              <Text
                accessibilityRole="alert"
                className="mt-1 text-sm text-red-500"
              >
                {errors.email.message}
              </Text>
            )}
          </View>
        )}
      />

      <Pressable
        onPress={handleSubmit(onSubmit)}
        accessibilityRole="button"
        accessibilityLabel="Create account"
        className="rounded-lg bg-blue-600 py-3"
      >
        <Text className="text-center font-semibold text-white">
          Create Account
        </Text>
      </Pressable>
    </View>
  );
}
```

---

## Navigation Errors

### Auth Redirect on 401

```typescript
// In your httpService interceptor
import { router } from 'expo-router';
import { store } from '@/store';
import { clearAuth } from '@/store/authSlice';

async function handleUnauthorized(): Promise<void> {
  // Clear stored auth state
  store.dispatch(clearAuth());

  // Redirect to login
  router.replace('/login');
}
```

### Invalid Deep Link Handling

```tsx
// app/[...unmatched].tsx — Expo Router catch-all for missing routes
import { View, Text, Pressable } from 'react-native';
import { useRouter } from 'expo-router';

export default function NotFoundScreen() {
  const router = useRouter();

  return (
    <View className="flex-1 items-center justify-center p-6">
      <Text className="mb-2 text-xl font-bold text-gray-900">
        Page Not Found
      </Text>
      <Text className="mb-6 text-center text-gray-600">
        The link you followed may be broken or the page may have been removed.
      </Text>
      <Pressable
        onPress={() => router.replace('/')}
        accessibilityRole="button"
        accessibilityLabel="Go to home screen"
        className="rounded-lg bg-blue-600 px-6 py-3"
      >
        <Text className="font-semibold text-white">Go Home</Text>
      </Pressable>
    </View>
  );
}
```

---

## Crash Reporting

### Sentry Setup Overview

```typescript
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: __DEV__ ? 'development' : 'production',
  tracesSampleRate: 0.2,
  beforeSend(event) {
    // Scrub sensitive data
    if (event.request?.headers) {
      delete event.request.headers['Authorization'];
    }
    return event;
  },
});

export default Sentry.wrap(RootLayout);
```

### Breadcrumbs and Context

```typescript
// Add breadcrumbs at key user actions
Sentry.addBreadcrumb({
  category: 'navigation',
  message: `Navigated to ${screenName}`,
  level: 'info',
});

Sentry.addBreadcrumb({
  category: 'user-action',
  message: 'Tapped checkout button',
  level: 'info',
});

// Set user context after login
Sentry.setUser({
  id: user.id,
  email: user.email,
});

// Add custom context
Sentry.setContext('cart', {
  itemCount: cart.items.length,
  total: cart.total,
});
```

> For full Sentry setup, source maps, and crash handler patterns, see the [crash-handler skill](../skills/debugging/crash-handler.md).

---

## Retry and Fallback Patterns

### TanStack Query Retry Configuration

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,
      retryDelay: (attemptIndex: number) =>
        Math.min(1000 * 2 ** attemptIndex, 30000), // exponential backoff, max 30s
      staleTime: 5 * 60 * 1000, // 5 minutes
    },
    mutations: {
      retry: 1,
    },
  },
});
```

### Manual Retry with Exponential Backoff

```typescript
interface RetryOptions {
  maxAttempts: number;
  baseDelay: number;
  maxDelay: number;
}

async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = { maxAttempts: 3, baseDelay: 1000, maxDelay: 10000 }
): Promise<T> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error instanceof Error ? error : new Error(String(error));

      if (attempt < options.maxAttempts - 1) {
        const delay = Math.min(
          options.baseDelay * 2 ** attempt,
          options.maxDelay
        );
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}
```

### Fallback UI Patterns

```tsx
function ProductList() {
  const { data, isLoading, isError, error, refetch } = useProducts();
  const { isConnected } = useNetworkStatus();

  // Loading skeleton
  if (isLoading) {
    return <ProductListSkeleton count={6} />;
  }

  // Error with retry
  if (isError) {
    return (
      <View className="flex-1 items-center justify-center p-6">
        <Text className="mb-2 text-lg font-semibold text-gray-900">
          {isConnected ? 'Failed to load products' : 'You are offline'}
        </Text>
        <Text className="mb-6 text-center text-gray-600">
          {error.message}
        </Text>
        <Pressable
          onPress={() => refetch()}
          accessibilityRole="button"
          accessibilityLabel="Retry loading products"
          className="rounded-lg bg-blue-600 px-6 py-3"
        >
          <Text className="font-semibold text-white">Try Again</Text>
        </Pressable>
      </View>
    );
  }

  // Empty state
  if (data?.length === 0) {
    return (
      <View className="flex-1 items-center justify-center p-6">
        <Text className="text-lg text-gray-600">No products found</Text>
      </View>
    );
  }

  // Success
  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <ProductCard product={item} />}
    />
  );
}
```

### Graceful Degradation

```tsx
// Show cached data when fresh data fails to load
function UserProfile({ userId }: { userId: string }) {
  const { data, isError, isLoading } = useUser(userId, {
    // Keep showing stale data while refetching fails
    placeholderData: (previousData) => previousData,
  });

  return (
    <View className="p-4">
      {isError && (
        <View className="mb-4 rounded-lg bg-yellow-50 p-3">
          <Text className="text-sm text-yellow-700">
            Showing cached data — some info may be outdated
          </Text>
        </View>
      )}
      {data && <ProfileContent user={data} />}
    </View>
  );
}
```

---

## Related Resources

- [Crash Handler Skill](../skills/debugging/crash-handler.md) — full Sentry setup and crash handling
- [Loading and Error States Guide](./loading-and-error-states.md) — UI patterns for async states
- [Data Fetching Guide](./data-fetching.md) — httpService and TanStack Query architecture
