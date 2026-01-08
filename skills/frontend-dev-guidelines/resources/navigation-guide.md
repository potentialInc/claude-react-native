# Navigation Guide

React Navigation 6.x implementation for React Native apps with TypeScript support.

---

## React Navigation Overview

The project uses **React Navigation 6.x** with:

- Stack Navigator for screen transitions
- Bottom Tab Navigator for main navigation
- Drawer Navigator for side menus
- TypeScript integration for type-safe navigation
- Deep linking support

---

## Installation

### Required Packages

```bash
# Core packages
npm install @react-navigation/native
npm install react-native-screens react-native-safe-area-context

# Navigator types (install as needed)
npm install @react-navigation/native-stack    # Stack Navigator
npm install @react-navigation/bottom-tabs     # Bottom Tabs
npm install @react-navigation/drawer          # Drawer Navigator
```

### iOS Setup

```bash
cd ios && pod install
```

### Android Setup

Update `MainActivity.java` or `MainActivity.kt` for react-native-screens.

---

## Basic Setup

### Navigation Container

```typescript
// App.tsx
import { NavigationContainer } from '@react-navigation/native';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import RootNavigator from './src/navigation/RootNavigator';

export default function App() {
  return (
    <SafeAreaProvider>
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </SafeAreaProvider>
  );
}
```

---

## Stack Navigator

### Basic Stack Setup

```typescript
// src/navigation/RootNavigator.tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import HomeScreen from '../screens/HomeScreen';
import DetailsScreen from '../screens/DetailsScreen';
import ProfileScreen from '../screens/ProfileScreen';

export type RootStackParamList = {
  Home: undefined;
  Details: { itemId: number; title: string };
  Profile: { userId: string };
};

const Stack = createNativeStackNavigator<RootStackParamList>();

export default function RootNavigator() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen
        name="Home"
        component={HomeScreen}
        options={{ title: 'Home' }}
      />
      <Stack.Screen
        name="Details"
        component={DetailsScreen}
        options={{ title: 'Details' }}
      />
      <Stack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: 'Profile' }}
      />
    </Stack.Navigator>
  );
}
```

### Screen Options

```typescript
<Stack.Navigator
  screenOptions={{
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#fff',
    headerTitleStyle: { fontWeight: 'bold' },
  }}
>
  <Stack.Screen
    name="Home"
    component={HomeScreen}
    options={{
      title: 'Welcome',
      headerShown: true,
      headerBackVisible: false,
    }}
  />
  <Stack.Screen
    name="Details"
    component={DetailsScreen}
    options={({ route }) => ({
      title: route.params.title,
    })}
  />
</Stack.Navigator>
```

---

## Bottom Tab Navigator

### Tab Setup

```typescript
// src/navigation/MainTabNavigator.tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import Icon from 'react-native-vector-icons/MaterialCommunityIcons';
import HomeScreen from '../screens/HomeScreen';
import SearchScreen from '../screens/SearchScreen';
import ProfileScreen from '../screens/ProfileScreen';

export type MainTabParamList = {
  Home: undefined;
  Search: undefined;
  Profile: undefined;
};

const Tab = createBottomTabNavigator<MainTabParamList>();

export default function MainTabNavigator() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          let iconName: string;

          switch (route.name) {
            case 'Home':
              iconName = focused ? 'home' : 'home-outline';
              break;
            case 'Search':
              iconName = focused ? 'magnify' : 'magnify';
              break;
            case 'Profile':
              iconName = focused ? 'account' : 'account-outline';
              break;
            default:
              iconName = 'circle';
          }

          return <Icon name={iconName} size={size} color={color} />;
        },
        tabBarActiveTintColor: '#6200EE',
        tabBarInactiveTintColor: 'gray',
      })}
    >
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

### Tab with Badge

```typescript
<Tab.Screen
  name="Notifications"
  component={NotificationsScreen}
  options={{
    tabBarBadge: 3,
    tabBarBadgeStyle: { backgroundColor: 'red' },
  }}
/>
```

---

## Navigation Patterns

### Using useNavigation Hook

```typescript
import { useNavigation } from '@react-navigation/native';
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import type { RootStackParamList } from '../navigation/RootNavigator';

type NavigationProp = NativeStackNavigationProp<RootStackParamList>;

export default function HomeScreen() {
  const navigation = useNavigation<NavigationProp>();

  const handlePress = () => {
    // Navigate to screen
    navigation.navigate('Details', { itemId: 123, title: 'Item Title' });
  };

  const handleReplace = () => {
    // Replace current screen
    navigation.replace('Profile', { userId: 'user123' });
  };

  const handleGoBack = () => {
    // Go back to previous screen
    navigation.goBack();
  };

  const handleReset = () => {
    // Reset navigation state
    navigation.reset({
      index: 0,
      routes: [{ name: 'Home' }],
    });
  };

  return (
    <View className="flex-1 p-4">
      <Button mode="contained" onPress={handlePress}>
        Go to Details
      </Button>
    </View>
  );
}
```

### Navigation Methods

| Method | Description |
|--------|-------------|
| `navigate(name, params)` | Navigate to screen (won't duplicate if already on stack) |
| `push(name, params)` | Push screen onto stack (can duplicate) |
| `goBack()` | Go back to previous screen |
| `replace(name, params)` | Replace current screen |
| `reset(state)` | Reset navigation state |
| `popToTop()` | Go back to first screen in stack |

---

## Route Parameters

### Passing Parameters

```typescript
// Navigate with params
navigation.navigate('Details', {
  itemId: 42,
  title: 'Product Name',
});

// Navigate with optional params
navigation.navigate('Search', {
  query: 'shoes',
  filters: { category: 'clothing' },
});
```

### Accessing Parameters with useRoute

```typescript
import { useRoute } from '@react-navigation/native';
import type { RouteProp } from '@react-navigation/native';
import type { RootStackParamList } from '../navigation/RootNavigator';

type DetailsRouteProp = RouteProp<RootStackParamList, 'Details'>;

export default function DetailsScreen() {
  const route = useRoute<DetailsRouteProp>();

  // Type-safe access to params
  const { itemId, title } = route.params;

  return (
    <View className="flex-1 p-4">
      <Text className="text-xl font-bold">{title}</Text>
      <Text className="text-muted-foreground">Item ID: {itemId}</Text>
    </View>
  );
}
```

### Initial Params

```typescript
<Stack.Screen
  name="Details"
  component={DetailsScreen}
  initialParams={{ itemId: 0, title: 'Default Title' }}
/>
```

---

## TypeScript Integration

### Defining Param Lists

```typescript
// src/navigation/types.ts
export type RootStackParamList = {
  Home: undefined;
  Details: { itemId: number; title: string };
  Profile: { userId: string };
  Settings: undefined;
};

export type AuthStackParamList = {
  Login: undefined;
  Register: { referralCode?: string };
  ForgotPassword: { email?: string };
};

export type MainTabParamList = {
  Home: undefined;
  Search: { query?: string };
  Profile: undefined;
};
```

### Typed Navigation Props

```typescript
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import type { RouteProp } from '@react-navigation/native';
import type { RootStackParamList } from './types';

// For components receiving navigation as props
export type DetailsScreenNavigationProp = NativeStackNavigationProp<
  RootStackParamList,
  'Details'
>;

export type DetailsScreenRouteProp = RouteProp<RootStackParamList, 'Details'>;

interface DetailsScreenProps {
  navigation: DetailsScreenNavigationProp;
  route: DetailsScreenRouteProp;
}

export default function DetailsScreen({ navigation, route }: DetailsScreenProps) {
  // ...
}
```

### Global Type Declaration

```typescript
// src/navigation/types.ts
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

This enables automatic typing for `useNavigation()` without explicit type parameters.

---

## Nested Navigators

### Stack Inside Tabs

```typescript
// src/navigation/HomeStackNavigator.tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';

export type HomeStackParamList = {
  HomeMain: undefined;
  HomeDetails: { itemId: number };
};

const Stack = createNativeStackNavigator<HomeStackParamList>();

export default function HomeStackNavigator() {
  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      <Stack.Screen name="HomeMain" component={HomeMainScreen} />
      <Stack.Screen name="HomeDetails" component={HomeDetailsScreen} />
    </Stack.Navigator>
  );
}

// In MainTabNavigator.tsx
<Tab.Screen name="Home" component={HomeStackNavigator} />
```

### Navigating Between Nested Navigators

```typescript
// Navigate to nested screen
navigation.navigate('Home', {
  screen: 'HomeDetails',
  params: { itemId: 42 },
});
```

---

## Protected Routes

### Auth Flow Pattern

```typescript
// src/navigation/RootNavigator.tsx
import { useAppSelector } from '~/redux/store/hooks';

export default function RootNavigator() {
  const isAuthenticated = useAppSelector((state) => !!state.auth.token);

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isAuthenticated ? (
        // Authenticated screens
        <>
          <Stack.Screen name="Main" component={MainTabNavigator} />
          <Stack.Screen name="Settings" component={SettingsScreen} />
        </>
      ) : (
        // Auth screens
        <>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}
```

### Auth Context Pattern

```typescript
// src/context/AuthContext.tsx
import { createContext, useContext, useState } from 'react';

interface AuthContextType {
  isAuthenticated: boolean;
  login: (token: string) => void;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  const login = (token: string) => {
    // Store token
    setIsAuthenticated(true);
  };

  const logout = () => {
    // Clear token
    setIsAuthenticated(false);
  };

  return (
    <AuthContext.Provider value={{ isAuthenticated, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};
```

---

## Deep Linking

### Configuration

```typescript
// App.tsx
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: 'home',
      Details: 'details/:itemId',
      Profile: 'user/:userId',
      NotFound: '*',
    },
  },
};

<NavigationContainer linking={linking}>
  <RootNavigator />
</NavigationContainer>
```

### URL Examples

| URL | Screen | Params |
|-----|--------|--------|
| `myapp://home` | Home | - |
| `myapp://details/123` | Details | `{ itemId: '123' }` |
| `myapp://user/abc` | Profile | `{ userId: 'abc' }` |

### iOS Configuration (Info.plist)

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

### Android Configuration (AndroidManifest.xml)

```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="myapp" />
</intent-filter>
```

---

## Header Customization

### Custom Header Component

```typescript
<Stack.Screen
  name="Home"
  component={HomeScreen}
  options={{
    header: ({ navigation, route, options }) => (
      <View className="flex-row items-center p-4 bg-primary">
        <Text className="text-xl font-bold text-white">
          {options.title || route.name}
        </Text>
      </View>
    ),
  }}
/>
```

### Header with Back Button

```typescript
import { Appbar } from 'react-native-paper';

<Stack.Screen
  name="Details"
  component={DetailsScreen}
  options={{
    header: ({ navigation, route }) => (
      <Appbar.Header>
        <Appbar.BackAction onPress={() => navigation.goBack()} />
        <Appbar.Content title={route.params?.title || 'Details'} />
        <Appbar.Action icon="dots-vertical" onPress={() => {}} />
      </Appbar.Header>
    ),
  }}
/>
```

---

## Navigation Events

### Focus and Blur Events

```typescript
import { useFocusEffect } from '@react-navigation/native';
import { useCallback } from 'react';

export default function ProfileScreen() {
  useFocusEffect(
    useCallback(() => {
      // Called when screen comes into focus
      console.log('Screen focused');
      fetchUserData();

      return () => {
        // Called when screen goes out of focus
        console.log('Screen blurred');
      };
    }, [])
  );

  return <View />;
}
```

### Navigation Listeners

```typescript
useEffect(() => {
  const unsubscribe = navigation.addListener('focus', () => {
    // Screen was focused
  });

  return unsubscribe;
}, [navigation]);
```

---

## Common Patterns

### Modal Screens

```typescript
<Stack.Navigator>
  <Stack.Group>
    <Stack.Screen name="Home" component={HomeScreen} />
    <Stack.Screen name="Details" component={DetailsScreen} />
  </Stack.Group>

  <Stack.Group screenOptions={{ presentation: 'modal' }}>
    <Stack.Screen name="CreatePost" component={CreatePostScreen} />
    <Stack.Screen name="Settings" component={SettingsScreen} />
  </Stack.Group>
</Stack.Navigator>
```

### Preventing Go Back

```typescript
useEffect(() => {
  const unsubscribe = navigation.addListener('beforeRemove', (e) => {
    if (!hasUnsavedChanges) {
      return;
    }

    e.preventDefault();

    Alert.alert(
      'Discard changes?',
      'You have unsaved changes. Are you sure you want to discard them?',
      [
        { text: "Don't leave", style: 'cancel' },
        {
          text: 'Discard',
          style: 'destructive',
          onPress: () => navigation.dispatch(e.data.action),
        },
      ]
    );
  });

  return unsubscribe;
}, [navigation, hasUnsavedChanges]);
```

---

## Summary

**Navigation Checklist:**

- ✅ Use `@react-navigation/native-stack` for Stack Navigator
- ✅ Use `@react-navigation/bottom-tabs` for Tab Navigator
- ✅ Define `ParamList` types for type-safe navigation
- ✅ Use `useNavigation` hook for navigation actions
- ✅ Use `useRoute` hook for accessing parameters
- ✅ Handle auth state with conditional navigator rendering
- ✅ Configure deep linking for external URLs
- ✅ Use `useFocusEffect` for screen focus events

**See Also:**

- [component-patterns.md](component-patterns.md) - Screen component patterns
- [typescript-standards.md](typescript-standards.md) - TypeScript patterns
- [common-patterns.md](common-patterns.md) - Auth patterns
- [complete-examples.md](complete-examples.md) - Full navigation examples
