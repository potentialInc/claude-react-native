# Styling Guide

Modern styling patterns using NativeWind 4.x (Tailwind CSS for React Native) and React Native Paper.

---

## Core Concepts

### NativeWind 4.x

NativeWind brings Tailwind CSS to React Native with utility-first styling:

```typescript
import { View, Text } from 'react-native';

// Direct utility classes
<View className="p-4 mb-3 flex flex-col gap-2">
  <Text className="text-base text-foreground">Content</Text>
</View>
```

### Installation

```bash
npm install nativewind tailwindcss
npx tailwindcss init
```

### Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: [
    './app/**/*.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        primary: '#007AFF',
        secondary: '#5856D6',
        destructive: '#FF3B30',
        background: '#FFFFFF',
        foreground: '#000000',
        muted: '#8E8E93',
        'muted-foreground': '#636366',
      },
    },
  },
  plugins: [],
};
```

```javascript
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
  };
};
```

---

## cn() Utility

Use `cn()` for conditional class merging (combines `clsx` + `tailwind-merge`):

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

// With Pressable
<Pressable
  className={({ pressed }) => cn(
    'p-4 rounded-lg bg-primary',
    pressed && 'opacity-80'
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

## Color System

### Defining Theme Colors

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        // Semantic colors
        background: 'rgb(var(--background) / <alpha-value>)',
        foreground: 'rgb(var(--foreground) / <alpha-value>)',
        primary: {
          DEFAULT: 'rgb(var(--primary) / <alpha-value>)',
          foreground: 'rgb(var(--primary-foreground) / <alpha-value>)',
        },
        secondary: {
          DEFAULT: 'rgb(var(--secondary) / <alpha-value>)',
          foreground: 'rgb(var(--secondary-foreground) / <alpha-value>)',
        },
        muted: {
          DEFAULT: 'rgb(var(--muted) / <alpha-value>)',
          foreground: 'rgb(var(--muted-foreground) / <alpha-value>)',
        },
        destructive: {
          DEFAULT: 'rgb(var(--destructive) / <alpha-value>)',
          foreground: 'rgb(var(--destructive-foreground) / <alpha-value>)',
        },
      },
    },
  },
};
```

### Using Colors

```typescript
// Via Tailwind classes (preferred)
<View className="bg-background" />
<Text className="text-foreground" />
<Text className="text-primary" />
<Text className="text-muted-foreground" />
<View className="bg-destructive" />

// Semantic usage
<Text className="text-muted-foreground">Secondary text</Text>
<Pressable className="bg-primary">
  <Text className="text-primary-foreground">Button</Text>
</Pressable>
```

---

## Dark Mode

### Setup with NativeWind

```typescript
// app/_layout.tsx
import { useColorScheme } from 'nativewind';
import { View } from 'react-native';

export default function RootLayout() {
  const { colorScheme, setColorScheme } = useColorScheme();

  return (
    <View className={colorScheme === 'dark' ? 'dark' : ''}>
      {/* App content */}
    </View>
  );
}
```

### Dark Mode Classes

```typescript
// Automatic with semantic colors (preferred)
<View className="bg-background" />
// Light: white bg, Dark: dark bg (based on theme config)

// Explicit dark mode variants
<View className="bg-white dark:bg-gray-900" />
<Text className="text-gray-900 dark:text-white" />

// Toggle dark mode
const { toggleColorScheme } = useColorScheme();
<Pressable onPress={toggleColorScheme}>
  <Text>Toggle Dark Mode</Text>
</Pressable>
```

---

## Common Utility Classes

### Spacing

```typescript
// Padding
<View className="p-4" />      // All sides: 16px
<View className="px-4" />     // Horizontal: 16px
<View className="py-4" />     // Vertical: 16px
<View className="pt-4" />     // Top only: 16px
<View className="p-2" />      // All sides: 8px

// Margin
<View className="m-4" />      // All sides
<View className="mx-auto" />  // Center horizontally (not common in RN)
<View className="mt-4 mb-2" />

// Gap (for flex)
<View className="gap-4" />    // 16px gap
<View className="gap-2" />    // 8px gap
```

### Flexbox

```typescript
// Basic flex (default in React Native)
<View className="flex" />
<View className="flex-col" />  // Column (default)
<View className="flex-row" />  // Row

// Alignment
<View className="items-center" />
<View className="justify-center" />
<View className="justify-between" />
<View className="items-center justify-center" />

// Flex grow/shrink
<View className="flex-1" />      // flex: 1
<View className="flex-none" />   // flex: none
<View className="flex-grow" />   // flexGrow: 1
<View className="flex-shrink-0" /> // flexShrink: 0

// Common pattern - full screen centered
<View className="flex-1 items-center justify-center">
  <Text>Centered Content</Text>
</View>
```

### Typography

```typescript
// Font size
<Text className="text-xs" />    // 12px
<Text className="text-sm" />    // 14px
<Text className="text-base" />  // 16px
<Text className="text-lg" />    // 18px
<Text className="text-xl" />    // 20px
<Text className="text-2xl" />   // 24px
<Text className="text-3xl" />   // 30px

// Font weight
<Text className="font-normal" />    // 400
<Text className="font-medium" />    // 500
<Text className="font-semibold" />  // 600
<Text className="font-bold" />      // 700

// Text color
<Text className="text-foreground" />
<Text className="text-muted-foreground" />
<Text className="text-primary" />
<Text className="text-destructive" />

// Text alignment
<Text className="text-center" />
<Text className="text-left" />
<Text className="text-right" />

// Line height
<Text className="leading-tight" />
<Text className="leading-normal" />
<Text className="leading-relaxed" />
```

### Sizing

```typescript
// Width
<View className="w-full" />     // width: '100%'
<View className="w-1/2" />      // width: '50%'
<View className="w-auto" />     // width: 'auto'
<View className="w-20" />       // width: 80px
<View className="w-40" />       // width: 160px

// Height
<View className="h-full" />     // height: '100%'
<View className="h-20" />       // height: 80px
<View className="min-h-screen" /> // minHeight: screen height

// Fixed sizes
<View className="w-10 h-10" />  // 40x40px
<View className="size-10" />    // 40x40px (shorthand)
```

### Borders & Rounded

```typescript
// Border
<View className="border" />           // 1px border
<View className="border-2" />         // 2px border
<View className="border-primary" />   // Colored border
<View className="border-b" />         // Bottom only

// Rounded corners
<View className="rounded" />          // Small radius
<View className="rounded-md" />       // Medium radius
<View className="rounded-lg" />       // Large radius
<View className="rounded-xl" />       // Extra large
<View className="rounded-full" />     // Circular
<View className="rounded-t-lg" />     // Top only
```

### Shadows (Platform-Specific)

```typescript
// iOS shadows
<View className="shadow-sm" />
<View className="shadow" />
<View className="shadow-md" />
<View className="shadow-lg" />

// Note: Shadows work differently on Android
// Use elevation for Android
import { Platform } from 'react-native';

<View
  className="shadow-md"
  style={Platform.OS === 'android' ? { elevation: 4 } : {}}
/>
```

---

## React Native-Specific Patterns

### Safe Area

```typescript
import { SafeAreaView } from 'react-native-safe-area-context';

// Full screen with safe area
<SafeAreaView className="flex-1 bg-background">
  <View className="flex-1 p-4">
    {/* Content */}
  </View>
</SafeAreaView>

// Specific edges
<SafeAreaView
  className="flex-1 bg-background"
  edges={['top', 'left', 'right']}
>
  {/* Content */}
</SafeAreaView>
```

### Absolute Positioning

```typescript
// Absolute positioning
<View className="relative flex-1">
  {/* Main content */}
  <View className="absolute bottom-4 right-4">
    {/* Floating element */}
  </View>
</View>

// FAB-style button
<Pressable className="absolute bottom-4 right-4 w-14 h-14 rounded-full bg-primary items-center justify-center shadow-lg">
  <Ionicons name="add" size={24} color="white" />
</Pressable>
```

### ScrollView Styling

```typescript
<ScrollView
  className="flex-1 bg-background"
  contentContainerClassName="p-4 gap-4"
  showsVerticalScrollIndicator={false}
>
  {/* Content */}
</ScrollView>
```

### FlatList Styling

```typescript
<FlatList
  data={items}
  className="flex-1"
  contentContainerClassName="p-4 gap-3"
  renderItem={({ item }) => (
    <View className="p-4 bg-card rounded-lg border border-border">
      <Text className="text-foreground">{item.title}</Text>
    </View>
  )}
/>
```

---

## Common Component Styling

### Card Pattern

```typescript
<View className="rounded-xl bg-card border border-border p-4 shadow-sm">
  <Text className="text-lg font-semibold text-foreground">Title</Text>
  <Text className="text-sm text-muted-foreground mt-1">Description</Text>
</View>
```

### Input Pattern

```typescript
<View className="gap-2">
  <Text className="text-sm font-medium text-foreground">Label</Text>
  <TextInput
    className="h-12 px-4 rounded-lg border border-input bg-background text-foreground"
    placeholderTextColor="#999"
    placeholder="Enter text..."
  />
</View>
```

### Button Pattern

```typescript
// Primary button
<Pressable
  className={({ pressed }) => cn(
    'h-12 px-6 rounded-lg bg-primary items-center justify-center',
    pressed && 'opacity-80'
  )}
>
  <Text className="text-base font-semibold text-primary-foreground">
    Button
  </Text>
</Pressable>

// Outline button
<Pressable
  className={({ pressed }) => cn(
    'h-12 px-6 rounded-lg border border-primary items-center justify-center',
    pressed && 'bg-primary/10'
  )}
>
  <Text className="text-base font-semibold text-primary">
    Outline
  </Text>
</Pressable>
```

### Screen Layout Pattern

```typescript
import { SafeAreaView } from 'react-native-safe-area-context';

export default function MyScreen() {
  return (
    <SafeAreaView className="flex-1 bg-background">
      {/* Header */}
      <View className="px-4 py-3 border-b border-border">
        <Text className="text-xl font-bold text-foreground">Title</Text>
      </View>

      {/* Content */}
      <ScrollView
        className="flex-1"
        contentContainerClassName="p-4 gap-4"
      >
        {/* Screen content */}
      </ScrollView>
    </SafeAreaView>
  );
}
```

---

## Platform-Specific Styling

### Platform Select

```typescript
import { Platform, StyleSheet } from 'react-native';

// With NativeWind, combine with Platform
<View
  className="p-4 rounded-lg bg-card"
  style={Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.1,
      shadowRadius: 4,
    },
    android: {
      elevation: 3,
    },
  })}
/>
```

### Conditional Classes by Platform

```typescript
import { Platform } from 'react-native';

<View className={cn(
  'p-4 rounded-lg bg-card',
  Platform.OS === 'ios' && 'shadow-md',
  Platform.OS === 'android' && 'elevation-3'
)} />
```

---

## What NOT to Do

### Avoid StyleSheet.create for Simple Styles

```typescript
// ❌ AVOID - StyleSheet for simple layouts
const styles = StyleSheet.create({
  container: {
    padding: 16,
    marginBottom: 12,
  },
});
<View style={styles.container} />

// ✅ PREFERRED - NativeWind classes
<View className="p-4 mb-3" />
```

### Avoid Inline Style Objects

```typescript
// ❌ AVOID - Inline style objects
<View style={{ padding: 16, marginBottom: 12 }}>
  Content
</View>

// ✅ PREFERRED - NativeWind classes
<View className="p-4 mb-3">
  Content
</View>
```

### When to Use StyleSheet

Use StyleSheet.create only when:
- Platform-specific styles (shadows, elevation)
- Complex animations with Reanimated
- Dynamic values that can't be expressed in Tailwind

```typescript
// ✅ OK - Platform-specific shadows
const styles = StyleSheet.create({
  shadow: Platform.select({
    ios: { shadowColor: '#000', shadowOffset: { width: 0, height: 2 } },
    android: { elevation: 4 },
  }),
});

<View className="p-4 rounded-lg bg-card" style={styles.shadow} />
```

---

## NativeWind 4.x Specific Features

### CSS Variables

```typescript
// Define in global.css
:root {
  --primary: 0 122 255;  /* RGB values */
}

// Use in tailwind.config.js
colors: {
  primary: 'rgb(var(--primary) / <alpha-value>)',
}

// Use in components
<View className="bg-primary" />
<View className="bg-primary/50" />  // 50% opacity
```

### Arbitrary Values

```typescript
// Custom sizes
<View className="w-[200px] h-[100px]" />

// Custom colors
<Text className="text-[#FF6B6B]" />

// Custom spacing
<View className="p-[13px] m-[7px]" />
```

### Group Modifiers

```typescript
<View className="group">
  <Text className="text-foreground group-hover:text-primary">
    Text changes on parent hover
  </Text>
</View>
```

---

## Summary

**NativeWind Styling Checklist:**

- ✅ Use NativeWind utility classes directly
- ✅ Use `cn()` for conditional class merging
- ✅ Use semantic color variables (`bg-background`, `text-foreground`)
- ✅ Use `flex-1` for full-height screens
- ✅ Use `SafeAreaView` for device-safe content
- ✅ Use `contentContainerClassName` for ScrollView/FlatList
- ✅ Platform-specific shadows via StyleSheet
- ❌ Avoid inline style objects
- ❌ Avoid StyleSheet.create for simple layouts
- ❌ Don't mix styling approaches inconsistently

**See Also:**

- [component-patterns.md](component-patterns.md) - Component structure
- [complete-examples.md](complete-examples.md) - Full styling examples
