# Error Handling Examples

Complete error handling patterns for React Native applications covering boundaries, network errors, form validation, global handlers, and offline support.

---

## Example 1: Error Boundary Component

```typescript
// src/components/ErrorBoundary.tsx
import { Component, type ReactNode } from 'react';
import { View, Pressable } from 'react-native';
import { Text, Icon } from 'react-native-paper';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode;
  onReset?: () => void;
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    console.error('ErrorBoundary caught:', error, errorInfo.componentStack);
  }

  handleReset = (): void => {
    this.setState({ hasError: false, error: null });
    this.props.onReset?.();
  };

  render(): ReactNode {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <View className="flex-1 items-center justify-center p-6 bg-background">
          <Icon source="alert-circle-outline" size={64} color="#DC2626" />
          <Text variant="headlineSmall" className="mt-4 text-center">
            Something went wrong
          </Text>
          <Text className="mt-2 text-center text-muted-foreground">
            {this.state.error?.message ?? 'An unexpected error occurred'}
          </Text>
          <Pressable
            onPress={this.handleReset}
            className="mt-6 bg-primary px-6 py-3 rounded-lg active:opacity-80"
          >
            <Text className="text-on-primary font-medium">Try Again</Text>
          </Pressable>
        </View>
      );
    }

    return this.props.children;
  }
}
```

### Usage

```typescript
// src/app/(tabs)/index.tsx
import { ErrorBoundary } from '~/components/ErrorBoundary';
import { HomeContent } from '~/components/HomeContent';

export default function HomeScreen() {
  return (
    <ErrorBoundary onReset={() => console.log('User retried from error boundary')}>
      <HomeContent />
    </ErrorBoundary>
  );
}
```

### Reset on Navigation Change

```typescript
// src/components/NavigationErrorBoundary.tsx
import { useNavigation } from '@react-navigation/native';
import { useEffect, useRef, type ReactNode } from 'react';
import { ErrorBoundary } from './ErrorBoundary';

interface NavigationErrorBoundaryProps {
  children: ReactNode;
}

export function NavigationErrorBoundary({ children }: NavigationErrorBoundaryProps) {
  const navigation = useNavigation();
  const boundaryRef = useRef<ErrorBoundary>(null);

  useEffect(() => {
    const unsubscribe = navigation.addListener('focus', () => {
      boundaryRef.current?.handleReset();
    });
    return unsubscribe;
  }, [navigation]);

  return (
    <ErrorBoundary ref={boundaryRef}>
      {children}
    </ErrorBoundary>
  );
}
```

---

## Example 2: Network Error Retry with TanStack Query

### HTTP Error Interceptor

```typescript
// src/services/httpService.ts
import axios, { type AxiosError, type AxiosResponse } from 'axios';

export interface ApiError {
  message: string;
  code: string;
  statusCode: number;
  details?: Record<string, string[]>;
}

export const httpService = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  timeout: 15000,
  headers: { 'Content-Type': 'application/json' },
});

httpService.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error: AxiosError<ApiError>) => {
    if (!error.response) {
      return Promise.reject(new Error('Network connection failed. Please check your internet.'));
    }

    const apiError: ApiError = {
      message: error.response.data?.message ?? 'An unexpected error occurred',
      code: error.response.data?.code ?? 'UNKNOWN_ERROR',
      statusCode: error.response.status,
      details: error.response.data?.details,
    };

    return Promise.reject(apiError);
  }
);
```

### Query with Retry and Error Boundary Integration

```typescript
// src/screens/UserProfileScreen.tsx
import { View } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Text, Button, ActivityIndicator, Card } from 'react-native-paper';
import { useQuery, QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from '~/components/ErrorBoundary';
import { httpService } from '~/services/httpService';

interface UserProfile {
  id: string;
  name: string;
  email: string;
  avatarUrl: string;
}

function UserProfileContent({ userId }: { userId: string }) {
  const { data: user, isLoading, error, refetch } = useQuery({
    queryKey: ['user', userId],
    queryFn: async () => {
      const response = await httpService.get<UserProfile>(`/users/${userId}`);
      return response.data;
    },
    retry: 3,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 10000),
  });

  if (isLoading) {
    return (
      <View className="flex-1 items-center justify-center">
        <ActivityIndicator size="large" />
      </View>
    );
  }

  if (error) {
    return (
      <View className="flex-1 items-center justify-center p-6">
        <Text className="text-destructive text-center mb-4">
          {error instanceof Error ? error.message : 'Failed to load profile'}
        </Text>
        <Button mode="outlined" onPress={() => refetch()}>
          Retry
        </Button>
      </View>
    );
  }

  if (!user) return null;

  return (
    <View className="p-4">
      <Card>
        <Card.Title title={user.name} subtitle={user.email} />
      </Card>
    </View>
  );
}

export function UserProfileScreen({ userId }: { userId: string }) {
  return (
    <SafeAreaView className="flex-1 bg-background">
      <QueryErrorResetBoundary>
        {({ reset }) => (
          <ErrorBoundary onReset={reset}>
            <UserProfileContent userId={userId} />
          </ErrorBoundary>
        )}
      </QueryErrorResetBoundary>
    </SafeAreaView>
  );
}
```

---

## Example 3: Form Validation Error Display

```typescript
// src/screens/CreateAccountScreen.tsx
import { View, ScrollView, KeyboardAvoidingView, Platform } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Text, TextInput, Button, HelperText } from 'react-native-paper';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import { useMutation } from '@tanstack/react-query';
import { httpService, type ApiError } from '~/services/httpService';

const createAccountSchema = z.object({
  name: z.string()
    .min(2, 'Name must be at least 2 characters')
    .max(50, 'Name must be under 50 characters'),
  email: z.string()
    .email('Please enter a valid email address'),
  phone: z.string()
    .regex(/^\+?[1-9]\d{7,14}$/, 'Please enter a valid phone number'),
  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain an uppercase letter')
    .regex(/[0-9]/, 'Password must contain a number'),
});

type CreateAccountData = z.infer<typeof createAccountSchema>;

type FieldName = keyof CreateAccountData;

export default function CreateAccountScreen() {
  const {
    control,
    handleSubmit,
    setError,
    formState: { errors },
  } = useForm<CreateAccountData>({
    resolver: zodResolver(createAccountSchema),
    defaultValues: { name: '', email: '', phone: '', password: '' },
  });

  const mutation = useMutation({
    mutationFn: async (data: CreateAccountData) => {
      const response = await httpService.post('/auth/register', data);
      return response.data;
    },
    onError: (error: ApiError) => {
      if (error.statusCode === 422 && error.details) {
        // Map server validation errors to individual fields
        const fieldNames: FieldName[] = ['name', 'email', 'phone', 'password'];
        fieldNames.forEach((field) => {
          const fieldErrors = error.details?.[field];
          if (fieldErrors && fieldErrors.length > 0) {
            setError(field, { type: 'server', message: fieldErrors[0] });
          }
        });
      }
    },
  });

  const onSubmit = (data: CreateAccountData) => {
    mutation.mutate(data);
  };

  return (
    <SafeAreaView className="flex-1 bg-background">
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
        className="flex-1"
      >
        <ScrollView contentContainerClassName="p-6 gap-4">
          <Text variant="headlineMedium" className="mb-4">Create Account</Text>

          {/* General server error (non-field) */}
          {mutation.isError && !mutation.error.details && (
            <View className="bg-destructive/10 p-4 rounded-lg">
              <Text className="text-destructive">{mutation.error.message}</Text>
            </View>
          )}

          {/* Name */}
          <View>
            <Controller
              control={control}
              name="name"
              render={({ field: { onChange, onBlur, value } }) => (
                <TextInput
                  label="Full Name"
                  mode="outlined"
                  value={value}
                  onChangeText={onChange}
                  onBlur={onBlur}
                  error={!!errors.name}
                  autoCapitalize="words"
                />
              )}
            />
            <HelperText type="error" visible={!!errors.name}>
              {errors.name?.message}
            </HelperText>
          </View>

          {/* Email */}
          <View>
            <Controller
              control={control}
              name="email"
              render={({ field: { onChange, onBlur, value } }) => (
                <TextInput
                  label="Email"
                  mode="outlined"
                  value={value}
                  onChangeText={onChange}
                  onBlur={onBlur}
                  error={!!errors.email}
                  keyboardType="email-address"
                  autoCapitalize="none"
                />
              )}
            />
            <HelperText type="error" visible={!!errors.email}>
              {errors.email?.message}
            </HelperText>
          </View>

          {/* Phone */}
          <View>
            <Controller
              control={control}
              name="phone"
              render={({ field: { onChange, onBlur, value } }) => (
                <TextInput
                  label="Phone Number"
                  mode="outlined"
                  value={value}
                  onChangeText={onChange}
                  onBlur={onBlur}
                  error={!!errors.phone}
                  keyboardType="phone-pad"
                />
              )}
            />
            <HelperText type="error" visible={!!errors.phone}>
              {errors.phone?.message}
            </HelperText>
          </View>

          {/* Password */}
          <View>
            <Controller
              control={control}
              name="password"
              render={({ field: { onChange, onBlur, value } }) => (
                <TextInput
                  label="Password"
                  mode="outlined"
                  value={value}
                  onChangeText={onChange}
                  onBlur={onBlur}
                  error={!!errors.password}
                  secureTextEntry
                />
              )}
            />
            <HelperText type="error" visible={!!errors.password}>
              {errors.password?.message}
            </HelperText>
          </View>

          <Button
            mode="contained"
            onPress={handleSubmit(onSubmit)}
            loading={mutation.isPending}
            disabled={mutation.isPending}
            className="mt-4"
          >
            Create Account
          </Button>
        </ScrollView>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}
```

---

## Example 4: Global Error Handler Setup

```typescript
// src/app/_layout.tsx (or App.tsx entry point)
import { useEffect } from 'react';
import { Platform } from 'react-native';
import * as Sentry from '@sentry/react-native';

// --- Sentry Initialization ---
Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: process.env.EXPO_PUBLIC_ENV ?? 'development',
  tracesSampleRate: 0.2,
  enableAutoSessionTracking: true,
  attachScreenshot: true,
  integrations: [
    Sentry.reactNativeTracingIntegration(),
    Sentry.reactNavigationIntegration(),
  ],
  beforeSend(event) {
    // Scrub sensitive data before sending
    if (event.request?.headers) {
      delete event.request.headers['Authorization'];
    }
    return event;
  },
});

// --- Global JS Error Handler ---
const originalHandler = ErrorUtils.getGlobalHandler();

ErrorUtils.setGlobalHandler((error: Error, isFatal: boolean) => {
  Sentry.captureException(error, {
    tags: { fatal: String(isFatal), platform: Platform.OS },
  });

  if (__DEV__) {
    console.error(`[GlobalErrorHandler] ${isFatal ? 'FATAL' : 'NON-FATAL'}:`, error);
  }

  // Call the original handler so React Native can show the red screen in dev
  originalHandler?.(error, isFatal);
});

// --- Unhandled Promise Rejection Handler ---
function setupPromiseRejectionHandler(): void {
  const tracking = require('promise/setimmediate/rejection-tracking');
  tracking.enable({
    allRejections: true,
    onUnhandled: (id: number, error: Error) => {
      Sentry.captureException(error, {
        tags: { type: 'unhandled_promise_rejection' },
        extra: { rejectionId: id },
      });

      if (__DEV__) {
        console.warn('[UnhandledPromiseRejection]', error);
      }
    },
    onHandled: () => {
      // Rejection was handled after being reported -- no action needed
    },
  });
}

// --- Root Layout ---
function RootLayout() {
  useEffect(() => {
    setupPromiseRejectionHandler();
  }, []);

  return (
    // ... your root layout (Stack, providers, etc.)
    null
  );
}

// Wrap the root component with Sentry
export default Sentry.wrap(RootLayout);
```

---

## Example 5: Offline Detection and Handling

### Offline Banner Component

```typescript
// src/components/OfflineBanner.tsx
import { View } from 'react-native';
import { Text, Icon } from 'react-native-paper';
import { useNetInfo } from '@react-native-community/netinfo';

export function OfflineBanner() {
  const netInfo = useNetInfo();

  if (netInfo.isConnected !== false) {
    return null;
  }

  return (
    <View className="bg-amber-600 px-4 py-3 flex-row items-center gap-2">
      <Icon source="wifi-off" size={18} color="#FFFFFF" />
      <Text className="text-white font-medium flex-1">
        You are offline. Some features may be unavailable.
      </Text>
    </View>
  );
}
```

### TanStack Query Online Manager Integration

```typescript
// src/providers/QueryProvider.tsx
import { useEffect, type ReactNode } from 'react';
import { AppState, type AppStateStatus } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import {
  QueryClient,
  QueryClientProvider,
  onlineManager,
  focusManager,
} from '@tanstack/react-query';

// Sync TanStack Query's online state with device connectivity
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(state.isConnected ?? false);
  });
});

// Refetch on app focus (coming back from background)
focusManager.setEventListener((setFocused) => {
  const subscription = AppState.addEventListener('change', (status: AppStateStatus) => {
    setFocused(status === 'active');
  });

  return () => {
    subscription.remove();
  };
});

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000,   // 10 minutes
    },
    mutations: {
      retry: 1,
    },
  },
});

interface QueryProviderProps {
  children: ReactNode;
}

export function QueryProvider({ children }: QueryProviderProps) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

### Queuing Mutations for Offline

```typescript
// src/hooks/useOfflineMutation.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { onlineManager } from '@tanstack/react-query';
import { httpService } from '~/services/httpService';

interface CreateCommentPayload {
  postId: string;
  body: string;
}

interface Comment {
  id: string;
  postId: string;
  body: string;
  createdAt: string;
}

export function useCreateComment() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload: CreateCommentPayload): Promise<Comment> => {
      const response = await httpService.post<Comment>('/comments', payload);
      return response.data;
    },

    // Optimistic update: add comment immediately
    onMutate: async (newComment) => {
      await queryClient.cancelQueries({ queryKey: ['comments', newComment.postId] });

      const previousComments = queryClient.getQueryData<Comment[]>(
        ['comments', newComment.postId]
      );

      const optimisticComment: Comment = {
        id: `temp-${Date.now()}`,
        postId: newComment.postId,
        body: newComment.body,
        createdAt: new Date().toISOString(),
      };

      queryClient.setQueryData<Comment[]>(
        ['comments', newComment.postId],
        (old) => [...(old ?? []), optimisticComment]
      );

      return { previousComments };
    },

    // Rollback on error
    onError: (_error, newComment, context) => {
      if (context?.previousComments) {
        queryClient.setQueryData(
          ['comments', newComment.postId],
          context.previousComments
        );
      }
    },

    // Refetch after success or error to sync with server
    onSettled: (_data, _error, variables) => {
      queryClient.invalidateQueries({ queryKey: ['comments', variables.postId] });
    },

    // Retry when back online
    networkMode: 'offlineFirst',
    retry: (failureCount) => {
      // Only retry if we are now online and haven't exceeded retries
      return onlineManager.isOnline() && failureCount < 3;
    },
  });
}
```

---

## Summary

These error handling examples cover:

1. **Error Boundaries** -- Class component with retry UI and navigation-aware reset
2. **Network Errors** -- HTTP interceptors, TanStack Query retry, and QueryErrorResetBoundary
3. **Form Validation** -- Zod schemas with field-level error display and server 422 mapping
4. **Global Handlers** -- ErrorUtils, promise rejection tracking, and Sentry integration
5. **Offline Support** -- NetInfo banner, onlineManager sync, and optimistic mutation queuing

**See Also:**
- [complete-examples.md](./complete-examples.md) -- Full screen implementations
- [testing-examples.md](./testing-examples.md) -- Testing these patterns
- [../guides/data-fetching.md](../guides/data-fetching.md) -- TanStack Query architecture
