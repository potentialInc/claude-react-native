# Routing Guide

Navigation implementation for React Native using Expo Router (file-based) and React Navigation (declarative).

---

## Navigation Options

| Feature | Expo Router | React Navigation |
|---------|------------|------------------|
| Routing Style | File-based (like Next.js) | Declarative configuration |
| Best For | Expo projects, rapid development | Fine-grained control, complex navigation |
| Deep Linking | Automatic | Manual configuration |
| Type Safety | Built-in via file system | Requires manual typing |

---

## Expo Router (Recommended for Expo Projects)

Expo Router provides file-based routing similar to Next.js for React Native.

### Installation

```bash
npx expo install expo-router expo-linking expo-constants
```

### Project Structure

```
app/
├── _layout.tsx          # Root layout (providers, navigation container)
├── index.tsx            # Home screen (/)
├── (auth)/              # Auth group (route grouping)
│   ├── _layout.tsx      # Auth layout (optional)
│   ├── login.tsx        # /login
│   └── register.tsx     # /register
├── (tabs)/              # Tab navigation group
│   ├── _layout.tsx      # Tab navigator layout
│   ├── home.tsx         # /home tab
│   ├── profile.tsx      # /profile tab
│   └── settings.tsx     # /settings tab
├── user/
│   └── [id].tsx         # /user/:id (dynamic route)
├── posts/
│   └── [...slug].tsx    # /posts/* (catch-all route)
└── +not-found.tsx       # 404 screen
```

### Route Naming Conventions

| Pattern | Example File | URL |
|---------|-------------|-----|
| Static | `about.tsx` | `/about` |
| Dynamic | `user/[id].tsx` | `/user/123` |
| Catch-all | `[...slug].tsx` | `/any/nested/path` |
| Groups | `(auth)/login.tsx` | `/login` |
| Index | `index.tsx` | `/` |

---

### Root Layout

Location: `app/_layout.tsx`

```typescript
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { Provider } from 'react-redux';
import { store } from '../src/redux/store';
import { queryClient } from '../src/lib/queryClient';

export default function RootLayout() {
  return (
    <Provider store={store}>
      <QueryClientProvider client={queryClient}>
        <Stack>
          <Stack.Screen name="index" options={{ headerShown: false }} />
          <Stack.Screen name="(auth)" options={{ headerShown: false }} />
          <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        </Stack>
      </QueryClientProvider>
    </Provider>
  );
}
```

---

### Tab Navigation Layout

Location: `app/(tabs)/_layout.tsx`

```typescript
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: '#007AFF',
        tabBarInactiveTintColor: '#8E8E93',
        headerShown: false,
      }}
    >
      <Tabs.Screen
        name="home"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

---

### Navigation Methods

#### Link Component

```typescript
import { Link } from 'expo-router';
import { Pressable, Text } from 'react-native';

// Basic link
<Link href="/about">
  <Text>About</Text>
</Link>

// Link with params
<Link href={`/user/${userId}`}>
  <Text>View Profile</Text>
</Link>

// Link as button (asChild)
<Link href="/settings" asChild>
  <Pressable className="bg-primary p-4 rounded-lg">
    <Text className="text-white">Settings</Text>
  </Pressable>
</Link>

// Replace (no back navigation)
<Link href="/login" replace>
  <Text>Login</Text>
</Link>
```

#### Programmatic Navigation

```typescript
import { useRouter } from 'expo-router';

export default function MyScreen() {
  const router = useRouter();

  const handleNavigation = () => {
    // Navigate forward
    router.push('/dashboard');

    // Navigate with params
    router.push(`/user/${userId}`);

    // Replace current screen
    router.replace('/login');

    // Go back
    router.back();

    // Navigate to specific tab
    router.push('/(tabs)/profile');
  };

  return <Button onPress={handleNavigation} title="Navigate" />;
}
```

---

### Route Parameters

#### Dynamic Route

File: `app/user/[id].tsx`

```typescript
import { useLocalSearchParams } from 'expo-router';
import { View, Text } from 'react-native';

export default function UserScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View className="flex-1 p-4">
      <Text className="text-lg">User ID: {id}</Text>
    </View>
  );
}
```

#### Query Parameters

```typescript
import { useLocalSearchParams, useRouter } from 'expo-router';

export default function SearchScreen() {
  const { q, page } = useLocalSearchParams<{ q?: string; page?: string }>();
  const router = useRouter();

  const updateSearch = (query: string) => {
    router.setParams({ q: query, page: '1' });
  };

  return (
    <TextInput
      value={q || ''}
      onChangeText={updateSearch}
      placeholder="Search..."
    />
  );
}
```

---

### Protected Routes

```typescript
// app/(auth)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAppSelector } from '../../src/redux/hooks';

export default function AuthLayout() {
  const isAuthenticated = useAppSelector((state) => state.auth.isAuthenticated);

  // Redirect authenticated users away from auth screens
  if (isAuthenticated) {
    return <Redirect href="/(tabs)/home" />;
  }

  return <Stack screenOptions={{ headerShown: false }} />;
}

// app/(tabs)/_layout.tsx - Protected tabs
import { Redirect, Tabs } from 'expo-router';
import { useAppSelector } from '../../src/redux/hooks';

export default function TabLayout() {
  const isAuthenticated = useAppSelector((state) => state.auth.isAuthenticated);

  if (!isAuthenticated) {
    return <Redirect href="/login" />;
  }

  return (
    <Tabs>
      {/* Tab screens */}
    </Tabs>
  );
}
```

---

### Deep Linking

Expo Router automatically generates deep links based on file structure.

```typescript
// app.json
{
  "expo": {
    "scheme": "myapp",
    "web": {
      "bundler": "metro"
    }
  }
}
```

Deep link examples:
- `myapp://` → `/`
- `myapp://user/123` → `/user/123`
- `myapp://settings?tab=notifications` → `/settings?tab=notifications`

---

## React Navigation (Alternative)

For projects not using Expo Router or needing more control.

### Installation

```bash
npm install @react-navigation/native @react-navigation/native-stack @react-navigation/bottom-tabs
npx expo install react-native-screens react-native-safe-area-context
```

### Type Definitions

```typescript
// src/types/navigation.ts
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import type { CompositeScreenProps } from '@react-navigation/native';

export type RootStackParamList = {
  Home: undefined;
  Login: undefined;
  Register: undefined;
  UserDetail: { userId: string };
  MainTabs: undefined;
};

export type TabParamList = {
  HomeTab: undefined;
  ProfileTab: undefined;
  SettingsTab: undefined;
};

export type RootStackScreenProps<T extends keyof RootStackParamList> =
  NativeStackScreenProps<RootStackParamList, T>;

export type TabScreenProps<T extends keyof TabParamList> = CompositeScreenProps<
  BottomTabScreenProps<TabParamList, T>,
  NativeStackScreenProps<RootStackParamList>
>;

// Enable type checking in useNavigation
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

### Navigator Setup

```typescript
// src/navigation/RootNavigator.tsx
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import type { RootStackParamList } from '../types/navigation';
import { useAppSelector } from '../redux/hooks';
import { LoginScreen } from '../screens/auth/LoginScreen';
import { MainTabs } from './MainTabs';

const Stack = createNativeStackNavigator<RootStackParamList>();

export function RootNavigator() {
  const isAuthenticated = useAppSelector((state) => state.auth.isAuthenticated);

  return (
    <NavigationContainer>
      <Stack.Navigator screenOptions={{ headerShown: false }}>
        {isAuthenticated ? (
          <Stack.Screen name="MainTabs" component={MainTabs} />
        ) : (
          <>
            <Stack.Screen name="Login" component={LoginScreen} />
            <Stack.Screen name="Register" component={RegisterScreen} />
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### Tab Navigator

```typescript
// src/navigation/MainTabs.tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { Ionicons } from '@expo/vector-icons';
import type { TabParamList } from '../types/navigation';

const Tab = createBottomTabNavigator<TabParamList>();

export function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          let iconName: keyof typeof Ionicons.glyphMap;

          switch (route.name) {
            case 'HomeTab':
              iconName = focused ? 'home' : 'home-outline';
              break;
            case 'ProfileTab':
              iconName = focused ? 'person' : 'person-outline';
              break;
            default:
              iconName = 'ellipse';
          }

          return <Ionicons name={iconName} size={size} color={color} />;
        },
        tabBarActiveTintColor: '#007AFF',
        tabBarInactiveTintColor: 'gray',
      })}
    >
      <Tab.Screen name="HomeTab" component={HomeScreen} options={{ title: 'Home' }} />
      <Tab.Screen name="ProfileTab" component={ProfileScreen} options={{ title: 'Profile' }} />
    </Tab.Navigator>
  );
}
```

### Navigation in Components

```typescript
import { useNavigation, useRoute } from '@react-navigation/native';
import type { RootStackScreenProps } from '../types/navigation';

// In a screen component
export function HomeScreen({ navigation }: RootStackScreenProps<'Home'>) {
  return (
    <Button
      title="View User"
      onPress={() => navigation.navigate('UserDetail', { userId: '123' })}
    />
  );
}

// Using hooks
export function SomeComponent() {
  const navigation = useNavigation();
  const route = useRoute();

  const goToProfile = () => {
    navigation.navigate('ProfileTab');
  };

  return <Button onPress={goToProfile} title="Profile" />;
}
```

---

## Modal and Stack Presentation

### Expo Router Modal

```typescript
// app/_layout.tsx
<Stack>
  <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
  <Stack.Screen
    name="modal"
    options={{
      presentation: 'modal',
      headerShown: true,
      title: 'Modal',
    }}
  />
</Stack>
```

### React Navigation Modal

```typescript
<Stack.Navigator>
  <Stack.Group>
    <Stack.Screen name="Home" component={HomeScreen} />
  </Stack.Group>
  <Stack.Group screenOptions={{ presentation: 'modal' }}>
    <Stack.Screen name="Modal" component={ModalScreen} />
  </Stack.Group>
</Stack.Navigator>
```

---

## Summary

**Expo Router Checklist:**

- ✅ Create screens in `app/` directory (file-based routing)
- ✅ Use `_layout.tsx` for layouts and navigation containers
- ✅ Use `(groupName)/` for route grouping
- ✅ Use `[param].tsx` for dynamic routes
- ✅ Use `Link` component for navigation
- ✅ Use `useRouter()` for programmatic navigation
- ✅ Use `useLocalSearchParams()` for route params
- ✅ Use `Redirect` for protected routes

**React Navigation Checklist:**

- ✅ Define typed `ParamList` for each navigator
- ✅ Set up `NavigationContainer` at root
- ✅ Use `createNativeStackNavigator` for stack navigation
- ✅ Use `createBottomTabNavigator` for tabs
- ✅ Use typed `navigation.navigate()` for navigation
- ✅ Use `useRoute()` for accessing params

**See Also:**

- [file-organization.md](file-organization.md) - Route file structure
- [common-patterns.md](common-patterns.md) - Auth patterns
- [complete-examples.md](complete-examples.md) - Full navigation examples
