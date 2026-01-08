# Component Patterns

Modern React Native component architecture using NativeWind, React Native Paper, and TypeScript.

---

## Function Component Pattern (PREFERRED)

### Why Function Components

All components use function components with TypeScript for:

- Explicit type safety for props
- Consistent component signatures
- Clear prop interface documentation
- Better IDE autocomplete
- Hooks-based state and effects

### Basic Pattern

```typescript
import { View, Text, Pressable } from 'react-native';
import { cn } from '@/lib/utils';

interface MyComponentProps {
  /** User ID to display */
  userId: number;
  /** Optional callback when action occurs */
  onAction?: () => void;
}

export default function MyComponent({ userId, onAction }: MyComponentProps) {
  return (
    <View className="p-4">
      <Text className="text-base text-foreground">User: {userId}</Text>
      <Pressable onPress={onAction} className="bg-primary p-3 rounded-lg mt-2">
        <Text className="text-white text-center">Action</Text>
      </Pressable>
    </View>
  );
}
```

**Key Points:**

- Props interface defined separately with JSDoc comments
- Destructure props in parameters
- Default export at bottom
- Use NativeWind classes for styling
- Use `Pressable` for touch interactions

---

## React Native Core Components

### Touchable Components

```typescript
import { Pressable, TouchableOpacity, TouchableHighlight } from 'react-native';

// Pressable (RECOMMENDED) - Most flexible
<Pressable
  onPress={handlePress}
  className={({ pressed }) => cn(
    'p-4 rounded-lg bg-primary',
    pressed && 'opacity-70'
  )}
>
  <Text>Press Me</Text>
</Pressable>

// TouchableOpacity - Simple opacity feedback
<TouchableOpacity onPress={handlePress} activeOpacity={0.7}>
  <Text>Touch Me</Text>
</TouchableOpacity>

// TouchableHighlight - Background highlight feedback
<TouchableHighlight onPress={handlePress} underlayColor="#ddd">
  <View className="p-4">
    <Text>Highlight Me</Text>
  </View>
</TouchableHighlight>
```

### Text Component

```typescript
import { Text } from 'react-native';

// Basic text
<Text className="text-base text-foreground">Regular text</Text>

// Styled text
<Text className="text-xl font-bold text-primary">Heading</Text>

// Text with number of lines
<Text numberOfLines={2} className="text-sm text-muted-foreground">
  Long text that might overflow...
</Text>

// Selectable text
<Text selectable className="text-base">
  User can copy this text
</Text>
```

### View and SafeAreaView

```typescript
import { View, SafeAreaView } from 'react-native';
import { SafeAreaProvider } from 'react-native-safe-area-context';

// Screen with safe area
export default function MyScreen() {
  return (
    <SafeAreaView className="flex-1 bg-background">
      <View className="flex-1 p-4">
        {/* Screen content */}
      </View>
    </SafeAreaView>
  );
}

// In root layout
export default function RootLayout() {
  return (
    <SafeAreaProvider>
      {/* App content */}
    </SafeAreaProvider>
  );
}
```

### ScrollView and FlatList

```typescript
import { ScrollView, FlatList, RefreshControl } from 'react-native';

// ScrollView for small lists or mixed content
<ScrollView
  className="flex-1"
  contentContainerClassName="p-4 gap-4"
  showsVerticalScrollIndicator={false}
>
  {/* Content */}
</ScrollView>

// FlatList for long lists (virtualized)
<FlatList
  data={items}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => <ItemCard item={item} />}
  contentContainerClassName="p-4 gap-4"
  showsVerticalScrollIndicator={false}
  refreshControl={
    <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
  }
  ListEmptyComponent={<EmptyState />}
/>
```

---

## React Native Paper Components

### Installation

```bash
npm install react-native-paper react-native-vector-icons
```

### Button Component

```typescript
import { Button } from 'react-native-paper';

// Variants
<Button mode="contained" onPress={handlePress}>Primary</Button>
<Button mode="outlined" onPress={handlePress}>Outlined</Button>
<Button mode="text" onPress={handlePress}>Text</Button>
<Button mode="elevated" onPress={handlePress}>Elevated</Button>
<Button mode="contained-tonal" onPress={handlePress}>Tonal</Button>

// With icons
<Button icon="camera" mode="contained" onPress={handlePress}>
  Take Photo
</Button>

// Loading state
<Button loading={isLoading} mode="contained" onPress={handlePress}>
  Submit
</Button>

// Disabled
<Button disabled mode="contained">Disabled</Button>
```

### Card Component

```typescript
import { Card, Text } from 'react-native-paper';

<Card className="m-4">
  <Card.Cover source={{ uri: imageUrl }} />
  <Card.Title
    title="Card Title"
    subtitle="Card Subtitle"
    left={(props) => <Avatar.Icon {...props} icon="folder" />}
  />
  <Card.Content>
    <Text variant="bodyMedium">Card content goes here</Text>
  </Card.Content>
  <Card.Actions>
    <Button>Cancel</Button>
    <Button mode="contained">OK</Button>
  </Card.Actions>
</Card>
```

### Text Input Component

```typescript
import { TextInput } from 'react-native-paper';

// Basic input
<TextInput
  label="Email"
  value={email}
  onChangeText={setEmail}
  mode="outlined"
/>

// With icons
<TextInput
  label="Password"
  value={password}
  onChangeText={setPassword}
  mode="outlined"
  secureTextEntry={!showPassword}
  right={
    <TextInput.Icon
      icon={showPassword ? 'eye-off' : 'eye'}
      onPress={() => setShowPassword(!showPassword)}
    />
  }
/>

// Error state
<TextInput
  label="Email"
  value={email}
  onChangeText={setEmail}
  mode="outlined"
  error={!!errors.email}
/>
{errors.email && (
  <HelperText type="error">{errors.email}</HelperText>
)}
```

### FAB (Floating Action Button)

```typescript
import { FAB, Portal, FAB as FABGroup } from 'react-native-paper';

// Simple FAB
<FAB
  icon="plus"
  className="absolute bottom-4 right-4"
  onPress={handleAdd}
/>

// FAB Group (expandable)
<Portal>
  <FABGroup
    open={open}
    visible
    icon={open ? 'close' : 'plus'}
    actions={[
      { icon: 'plus', label: 'Add', onPress: handleAdd },
      { icon: 'star', label: 'Star', onPress: handleStar },
    ]}
    onStateChange={({ open }) => setOpen(open)}
  />
</Portal>
```

---

## NativeWind Styling

### cn() Utility for Class Merging

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

### Usage Examples

```typescript
import { cn } from '@/lib/utils';

// Conditional classes
<View className={cn(
  'p-4 bg-background rounded-lg',
  isActive && 'bg-primary',
  isDisabled && 'opacity-50'
)} />

// Dynamic Pressable styling
<Pressable
  className={({ pressed }) => cn(
    'p-4 rounded-lg bg-primary',
    pressed && 'bg-primary/80'
  )}
>
  <Text className="text-white">Press Me</Text>
</Pressable>

// Array of conditions
<View className={cn([
  'base-class',
  condition1 && 'class-1',
  condition2 && 'class-2',
])} />
```

---

## Component Structure Template

### Recommended Order

```typescript
import { useState, useCallback, useMemo, useEffect } from 'react';
import { View, Text, Pressable } from 'react-native';
import { useRouter } from 'expo-router';

// Redux
import { useAppDispatch, useAppSelector } from '@/redux/hooks';

// UI Components
import { Button, Card } from 'react-native-paper';

// Utilities
import { cn } from '@/lib/utils';

// Types
import type { User } from '@/types/user';

// 1. PROPS INTERFACE (with JSDoc)
interface MyComponentProps {
  /** The ID of the entity to display */
  entityId: number;
  /** Optional callback when action completes */
  onComplete?: () => void;
  /** Display mode */
  mode?: 'view' | 'edit';
}

// 2. COMPONENT DEFINITION
export default function MyComponent({
  entityId,
  onComplete,
  mode = 'view',
}: MyComponentProps) {
  // 3. HOOKS (in this order)
  // - Navigation
  const router = useRouter();

  // - Redux hooks
  const dispatch = useAppDispatch();
  const user = useAppSelector((state) => state.user.currentUser);
  const loading = useAppSelector((state) => state.user.loading);

  // - Local state
  const [selectedItem, setSelectedItem] = useState<string | null>(null);
  const [isEditing, setIsEditing] = useState(mode === 'edit');

  // - Memoized values
  const filteredData = useMemo(() => {
    return data.filter((item) => item.active);
  }, [data]);

  // - Effects
  useEffect(() => {
    // Setup
    return () => {
      // Cleanup
    };
  }, []);

  // 4. EVENT HANDLERS (with useCallback for passed handlers)
  const handleItemSelect = useCallback((itemId: string) => {
    setSelectedItem(itemId);
  }, []);

  const handleSave = useCallback(async () => {
    try {
      // Save logic
      onComplete?.();
    } catch (error) {
      console.error('Failed to save:', error);
    }
  }, [onComplete]);

  // 5. RENDER
  return (
    <View className="flex-1 p-4">
      <Card>
        <Card.Content>
          <Text className={cn('text-base', loading && 'opacity-50')}>
            {user?.email}
          </Text>
          <Button
            mode="contained"
            onPress={handleSave}
            loading={loading}
            disabled={loading}
          >
            Save
          </Button>
        </Card.Content>
      </Card>
    </View>
  );
}
```

---

## Screen Component Pattern

### Basic Screen

```typescript
import { View, ScrollView } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Stack } from 'expo-router';

export default function ProfileScreen() {
  return (
    <>
      <Stack.Screen
        options={{
          title: 'Profile',
          headerShown: true,
        }}
      />
      <SafeAreaView className="flex-1 bg-background" edges={['bottom']}>
        <ScrollView
          className="flex-1"
          contentContainerClassName="p-4 gap-4"
          showsVerticalScrollIndicator={false}
        >
          {/* Screen content */}
        </ScrollView>
      </SafeAreaView>
    </>
  );
}
```

### Screen with Loading State

```typescript
import { View, ActivityIndicator } from 'react-native';
import { useQuery } from '@tanstack/react-query';

export default function DataScreen() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
  });

  if (isLoading) {
    return (
      <View className="flex-1 items-center justify-center">
        <ActivityIndicator size="large" color="#007AFF" />
      </View>
    );
  }

  if (error) {
    return (
      <View className="flex-1 items-center justify-center p-4">
        <Text className="text-destructive text-center">
          Error loading data. Please try again.
        </Text>
        <Button onPress={refetch} className="mt-4">
          Retry
        </Button>
      </View>
    );
  }

  return (
    <View className="flex-1">
      {/* Render data */}
    </View>
  );
}
```

---

## Platform-Specific Code

### Platform Detection

```typescript
import { Platform } from 'react-native';

// Simple check
if (Platform.OS === 'ios') {
  // iOS specific code
}

// Platform select
const styles = {
  shadow: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.25,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
  }),
};

// Version check
if (Platform.OS === 'android' && Platform.Version >= 23) {
  // Android 6.0+ specific code
}
```

### Platform-Specific Files

```
components/
├── Button.tsx          # Default (shared logic)
├── Button.ios.tsx      # iOS specific
└── Button.android.tsx  # Android specific
```

React Native automatically picks the right file based on platform.

---

## New Architecture Patterns

### TurboModules

For React Native 0.76+, use TurboModules for native module access:

```typescript
// Native modules are automatically lazy-loaded
import { NativeModules } from 'react-native';

// Access TurboModule
const { MyModule } = NativeModules;
```

### Fabric Components

Fabric components use the new rendering system:

```typescript
// Most React Native components are already Fabric-compatible
// Just use them normally with the new architecture enabled
import { View, Text, Image } from 'react-native';
```

---

## Component Communication

### Props Down, Events Up

```typescript
// Parent
function Parent() {
  const [selectedId, setSelectedId] = useState<string | null>(null);

  return (
    <Child
      data={data} // Props down
      onSelect={setSelectedId} // Events up
    />
  );
}

// Child
interface ChildProps {
  data: Data[];
  onSelect: (id: string) => void;
}

export default function Child({ data, onSelect }: ChildProps) {
  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <Pressable onPress={() => onSelect(item.id)}>
          <Text>{item.name}</Text>
        </Pressable>
      )}
    />
  );
}
```

### Using Redux for Shared State

```typescript
import { useAppDispatch, useAppSelector } from '@/redux/hooks';
import { selectItem } from '@/redux/slices/itemSlice';

function Child() {
  const dispatch = useAppDispatch();
  const selectedId = useAppSelector((state) => state.item.selectedId);

  const handleSelect = (id: string) => {
    dispatch(selectItem(id));
  };

  return (
    <Pressable onPress={() => handleSelect('123')}>
      <Text>Select</Text>
    </Pressable>
  );
}
```

---

## Summary

**Modern React Native Component Recipe:**

1. Function components with TypeScript props interface
2. NativeWind classes for styling
3. React Native Paper for Material Design components
4. `cn()` utility for conditional classes
5. `Pressable` for touch interactions
6. `useAppSelector` and `useAppDispatch` for Redux
7. `useRouter` from Expo Router for navigation
8. Event handlers with `useCallback` when passed to children
9. Default export at bottom

**See Also:**

- [styling-guide.md](styling-guide.md) - NativeWind patterns
- [common-patterns.md](common-patterns.md) - Form, modal, list patterns
- [complete-examples.md](complete-examples.md) - Full working examples
