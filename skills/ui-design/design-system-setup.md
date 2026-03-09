# Design System Setup

Guide for setting up a complete design system with NativeWind theming, React Native Paper customization, and dark mode support.

## When to Use

- Starting a new project or design system
- Adding dark mode support
- Creating a reusable component library
- When asked about "design system", "theme", "dark mode", "design tokens"

---

## Theme Configuration

### NativeWind / Tailwind Config

A complete `tailwind.config.js` with custom design tokens, dark mode, and React Native content paths:

```javascript
// tailwind.config.js
const { hairlineWidth } = require("nativewind/theme");

/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: "class",
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}", "./providers/**/*.{ts,tsx}"],
  presets: [require("nativewind/preset")],
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: "#2563EB",
          50: "#EFF6FF",
          100: "#DBEAFE",
          200: "#BFDBFE",
          300: "#93C5FD",
          400: "#60A5FA",
          500: "#2563EB",
          600: "#1D4ED8",
          700: "#1E40AF",
          800: "#1E3A8A",
          900: "#1E3B5C",
        },
        secondary: {
          DEFAULT: "#7C3AED",
          50: "#F5F3FF",
          100: "#EDE9FE",
          500: "#7C3AED",
          600: "#6D28D9",
          700: "#5B21B6",
        },
        surface: {
          DEFAULT: "#FFFFFF",
          dim: "#F5F5F5",
          bright: "#FAFAFA",
          dark: "#1A1A2E",
          "dark-dim": "#16162A",
          "dark-bright": "#1E1E36",
        },
        "on-primary": "#FFFFFF",
        "on-secondary": "#FFFFFF",
        "on-surface": {
          DEFAULT: "#1C1B1F",
          variant: "#49454F",
          dark: "#E6E1E5",
          "dark-variant": "#CAC4D0",
        },
        error: {
          DEFAULT: "#DC2626",
          50: "#FEF2F2",
          500: "#DC2626",
          600: "#B91C1C",
        },
        success: {
          DEFAULT: "#16A34A",
          50: "#F0FDF4",
          500: "#16A34A",
          600: "#15803D",
        },
        warning: {
          DEFAULT: "#D97706",
          50: "#FFFBEB",
          500: "#D97706",
          600: "#B45309",
        },
        outline: {
          DEFAULT: "#79747E",
          variant: "#CAC4D0",
        },
      },
      fontFamily: {
        sans: ["Inter"],
        "sans-medium": ["Inter-Medium"],
        "sans-semibold": ["Inter-SemiBold"],
        "sans-bold": ["Inter-Bold"],
        mono: ["JetBrainsMono"],
      },
      spacing: {
        xs: "4px",
        sm: "8px",
        md: "16px",
        lg: "24px",
        xl: "32px",
        "2xl": "48px",
        "3xl": "64px",
      },
      borderRadius: {
        none: "0px",
        sm: "4px",
        md: "8px",
        lg: "12px",
        xl: "16px",
        "2xl": "24px",
        full: "9999px",
      },
      borderWidth: {
        hairline: hairlineWidth(),
      },
    },
  },
  plugins: [],
};
```

### React Native Paper Theme

Customize Paper's MD3 theme to match NativeWind tokens:

```typescript
// theme/paperTheme.ts
import { MD3LightTheme, MD3DarkTheme, configureFonts } from "react-native-paper";
import type { MD3Theme } from "react-native-paper";

const fontConfig = {
  fontFamily: "Inter",
  displayLarge: { fontFamily: "Inter-Bold", fontSize: 57, lineHeight: 64 },
  displayMedium: { fontFamily: "Inter-Bold", fontSize: 45, lineHeight: 52 },
  headlineLarge: { fontFamily: "Inter-SemiBold", fontSize: 32, lineHeight: 40 },
  headlineMedium: { fontFamily: "Inter-SemiBold", fontSize: 28, lineHeight: 36 },
  titleLarge: { fontFamily: "Inter-SemiBold", fontSize: 22, lineHeight: 28 },
  titleMedium: { fontFamily: "Inter-Medium", fontSize: 16, lineHeight: 24 },
  bodyLarge: { fontFamily: "Inter", fontSize: 16, lineHeight: 24 },
  bodyMedium: { fontFamily: "Inter", fontSize: 14, lineHeight: 20 },
  labelLarge: { fontFamily: "Inter-Medium", fontSize: 14, lineHeight: 20 },
  labelMedium: { fontFamily: "Inter-Medium", fontSize: 12, lineHeight: 16 },
} as const;

export const lightTheme: MD3Theme = {
  ...MD3LightTheme,
  colors: {
    ...MD3LightTheme.colors,
    primary: "#2563EB",
    onPrimary: "#FFFFFF",
    primaryContainer: "#DBEAFE",
    secondary: "#7C3AED",
    onSecondary: "#FFFFFF",
    surface: "#FFFFFF",
    onSurface: "#1C1B1F",
    surfaceVariant: "#F5F5F5",
    error: "#DC2626",
    onError: "#FFFFFF",
    outline: "#79747E",
    outlineVariant: "#CAC4D0",
  },
  fonts: configureFonts({ config: fontConfig }),
};

export const darkTheme: MD3Theme = {
  ...MD3DarkTheme,
  colors: {
    ...MD3DarkTheme.colors,
    primary: "#60A5FA",
    onPrimary: "#1E3A8A",
    primaryContainer: "#1E40AF",
    secondary: "#A78BFA",
    onSecondary: "#3B0764",
    surface: "#1A1A2E",
    onSurface: "#E6E1E5",
    surfaceVariant: "#16162A",
    error: "#FCA5A5",
    onError: "#7F1D1D",
    outline: "#938F99",
    outlineVariant: "#49454F",
  },
  fonts: configureFonts({ config: fontConfig }),
};
```

### Combined Theme Provider

A unified provider wrapping both NativeWind dark mode and React Native Paper:

```typescript
// providers/ThemeProvider.tsx
import React, { createContext, useCallback, useContext, useEffect, useMemo, useState } from "react";
import { useColorScheme as useSystemColorScheme } from "react-native";
import { PaperProvider } from "react-native-paper";
import { useColorScheme } from "nativewind";
import { MMKV } from "react-native-mmkv";
import { lightTheme, darkTheme } from "@/theme/paperTheme";

type ThemeMode = "light" | "dark" | "system";

interface ThemeContextValue {
  mode: ThemeMode;
  isDark: boolean;
  setMode: (mode: ThemeMode) => void;
  toggleTheme: () => void;
}

const storage = new MMKV();
const THEME_KEY = "user-theme-mode";

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const systemScheme = useSystemColorScheme();
  const { setColorScheme } = useColorScheme();

  const [mode, setModeState] = useState<ThemeMode>(() => {
    const stored = storage.getString(THEME_KEY);
    return (stored as ThemeMode) ?? "system";
  });

  const isDark = mode === "system" ? systemScheme === "dark" : mode === "dark";

  const setMode = useCallback((newMode: ThemeMode) => {
    setModeState(newMode);
    storage.set(THEME_KEY, newMode);
  }, []);

  const toggleTheme = useCallback(() => {
    setMode(isDark ? "light" : "dark");
  }, [isDark, setMode]);

  useEffect(() => {
    setColorScheme(isDark ? "dark" : "light");
  }, [isDark, setColorScheme]);

  const paperTheme = isDark ? darkTheme : lightTheme;

  const value = useMemo(
    () => ({ mode, isDark, setMode, toggleTheme }),
    [mode, isDark, setMode, toggleTheme],
  );

  return (
    <ThemeContext.Provider value={value}>
      <PaperProvider theme={paperTheme}>{children}</PaperProvider>
    </ThemeContext.Provider>
  );
}

export function useTheme(): ThemeContextValue {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error("useTheme must be used within a ThemeProvider");
  }
  return context;
}
```

---

## Design Tokens

### Color Tokens

Use semantic naming that describes purpose, not appearance:

**GOOD:**
```tsx
<Pressable className="bg-primary">
  <Text className="text-on-primary">Submit</Text>
</Pressable>
<View className="bg-surface dark:bg-surface-dark">
  <Text className="text-on-surface dark:text-on-surface-dark">Content</Text>
</View>
```

**BAD:**
```tsx
<Pressable className="bg-blue-500">
  <Text className="text-white">Submit</Text>
</Pressable>
```

### Typography Scale

| Token        | Font              | Size | Line Height | Usage                  |
|-------------|-------------------|------|-------------|------------------------|
| `heading-1` | Inter-Bold        | 32px | 40px        | Screen titles          |
| `heading-2` | Inter-SemiBold    | 28px | 36px        | Section headers        |
| `heading-3` | Inter-SemiBold    | 22px | 28px        | Card titles            |
| `heading-4` | Inter-Medium      | 18px | 24px        | Subsection headers     |
| `body`      | Inter             | 16px | 24px        | Primary content        |
| `body-sm`   | Inter             | 14px | 20px        | Secondary content      |
| `caption`   | Inter             | 12px | 16px        | Help text, timestamps  |
| `label`     | Inter-Medium      | 14px | 20px        | Form labels, buttons   |

### Spacing Scale

All spacing based on a 4px base unit:

| Token  | Value | Usage                     |
|--------|-------|---------------------------|
| `xs`   | 4px   | Tight inline spacing      |
| `sm`   | 8px   | Related element gaps      |
| `md`   | 16px  | Standard padding/gaps     |
| `lg`   | 24px  | Section spacing           |
| `xl`   | 32px  | Large section spacing     |
| `2xl`  | 48px  | Page-level spacing        |

### Border Radius

| Token  | Value  | Usage                   |
|--------|--------|-------------------------|
| `none` | 0px    | Sharp corners           |
| `sm`   | 4px    | Subtle rounding         |
| `md`   | 8px    | Cards, inputs           |
| `lg`   | 12px   | Modals, sheets          |
| `xl`   | 16px   | Prominent containers    |
| `full` | 9999px | Pills, avatars          |

---

## Component Library Structure

```
components/
├── ui/
│   ├── Button.tsx          # Pressable + NativeWind variants
│   ├── Card.tsx            # Paper Card + NativeWind
│   ├── Input.tsx           # Paper TextInput + NativeWind
│   ├── Typography.tsx      # Text variants (heading, body, caption)
│   ├── Avatar.tsx
│   ├── Badge.tsx
│   ├── Chip.tsx
│   ├── Divider.tsx
│   └── index.ts            # Barrel export
├── forms/
│   ├── FormField.tsx       # RHF-connected wrapper
│   └── FormError.tsx       # Error message display
└── layout/
    ├── Container.tsx       # Safe-area padded wrapper
    ├── Row.tsx             # Horizontal flex
    └── Spacer.tsx          # Flexible spacing
```

### Button Component Example

A fully typed Button with variants, sizes, loading state, and NativeWind styling:

```typescript
// components/ui/Button.tsx
import React from "react";
import { ActivityIndicator, Pressable, Text, View } from "react-native";
import { cn } from "@/lib/utils";

type ButtonVariant = "primary" | "secondary" | "outline" | "ghost";
type ButtonSize = "sm" | "md" | "lg";

interface ButtonProps {
  label: string;
  onPress: () => void;
  variant?: ButtonVariant;
  size?: ButtonSize;
  loading?: boolean;
  disabled?: boolean;
  leftIcon?: React.ReactNode;
  className?: string;
}

const variantStyles: Record<ButtonVariant, string> = {
  primary: "bg-primary active:bg-primary-600",
  secondary: "bg-secondary active:bg-secondary-600",
  outline: "border border-outline bg-transparent active:bg-surface-dim dark:active:bg-surface-dark-dim",
  ghost: "bg-transparent active:bg-surface-dim dark:active:bg-surface-dark-dim",
};

const variantTextStyles: Record<ButtonVariant, string> = {
  primary: "text-on-primary",
  secondary: "text-on-secondary",
  outline: "text-primary",
  ghost: "text-primary",
};

const sizeStyles: Record<ButtonSize, string> = {
  sm: "px-3 py-1.5 rounded-sm",
  md: "px-4 py-2.5 rounded-md",
  lg: "px-6 py-3.5 rounded-lg",
};

const sizeTextStyles: Record<ButtonSize, string> = {
  sm: "text-xs font-sans-medium",
  md: "text-sm font-sans-medium",
  lg: "text-base font-sans-semibold",
};

export function Button({
  label,
  onPress,
  variant = "primary",
  size = "md",
  loading = false,
  disabled = false,
  leftIcon,
  className,
}: ButtonProps) {
  const isDisabled = disabled || loading;

  return (
    <Pressable
      onPress={onPress}
      disabled={isDisabled}
      accessibilityRole="button"
      accessibilityLabel={label}
      accessibilityState={{ disabled: isDisabled, busy: loading }}
      className={cn(
        "flex-row items-center justify-center",
        variantStyles[variant],
        sizeStyles[size],
        isDisabled && "opacity-50",
        className,
      )}
    >
      {loading ? (
        <ActivityIndicator
          size="small"
          color={variant === "primary" || variant === "secondary" ? "#FFFFFF" : "#2563EB"}
          className="mr-sm"
        />
      ) : leftIcon ? (
        <View className="mr-sm">{leftIcon}</View>
      ) : null}
      <Text className={cn(variantTextStyles[variant], sizeTextStyles[size])}>{label}</Text>
    </Pressable>
  );
}
```

### cn() Utility

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

### Input Component Example

Paper TextInput wrapped with NativeWind and React Hook Form integration:

```typescript
// components/ui/Input.tsx
import React from "react";
import { View, Text } from "react-native";
import { TextInput } from "react-native-paper";
import { useController, type Control, type FieldValues, type Path } from "react-hook-form";
import { cn } from "@/lib/utils";

interface InputProps<T extends FieldValues> {
  name: Path<T>;
  control: Control<T>;
  label: string;
  placeholder?: string;
  secureTextEntry?: boolean;
  className?: string;
}

export function Input<T extends FieldValues>({
  name,
  control,
  label,
  placeholder,
  secureTextEntry,
  className,
}: InputProps<T>) {
  const {
    field: { value, onChange, onBlur },
    fieldState: { error },
  } = useController({ name, control });

  return (
    <View className={cn("mb-md", className)}>
      <TextInput
        label={label}
        placeholder={placeholder}
        value={value}
        onChangeText={onChange}
        onBlur={onBlur}
        secureTextEntry={secureTextEntry}
        error={!!error}
        mode="outlined"
        accessibilityLabel={label}
        accessibilityHint={placeholder}
        className="bg-surface dark:bg-surface-dark"
      />
      {error?.message && (
        <Text className="mt-xs text-caption text-error dark:text-error-50">{error.message}</Text>
      )}
    </View>
  );
}
```

---

## Dark Mode Implementation

### Detection and Toggle

```typescript
// Hook for components that need theme info
import { useTheme } from "@/providers/ThemeProvider";

function SettingsScreen() {
  const { mode, isDark, setMode, toggleTheme } = useTheme();

  return (
    <View className="bg-surface dark:bg-surface-dark flex-1 p-md">
      <Text className="text-on-surface dark:text-on-surface-dark">
        Current: {isDark ? "Dark" : "Light"}
      </Text>

      <Pressable onPress={toggleTheme} accessibilityRole="button">
        <Text>Toggle Theme</Text>
      </Pressable>

      {/* Three-way selector: Light / Dark / System */}
      <Pressable onPress={() => setMode("system")} accessibilityRole="radio">
        <Text>Follow System</Text>
      </Pressable>
    </View>
  );
}
```

### NativeWind Dark Classes

```tsx
{/* Surface and text colors adapt automatically */}
<View className="bg-white dark:bg-gray-900 rounded-lg p-md">
  <Text className="text-gray-900 dark:text-gray-100 font-sans-semibold text-lg">
    Card Title
  </Text>
  <Text className="text-gray-600 dark:text-gray-400 font-sans text-sm mt-xs">
    Card description adapts to theme.
  </Text>
</View>

{/* Use cn() for conditional dark styling */}
<View className={cn("rounded-md p-md", isDark ? "border-gray-700" : "border-gray-200")} />
```

### StatusBar and Navigation Bar

```typescript
// app/_layout.tsx
import { StatusBar } from "expo-status-bar";
import { useTheme } from "@/providers/ThemeProvider";

export default function RootLayout() {
  const { isDark } = useTheme();

  return (
    <ThemeProvider>
      <StatusBar style={isDark ? "light" : "dark"} />
      <Stack
        screenOptions={{
          headerStyle: { backgroundColor: isDark ? "#1A1A2E" : "#FFFFFF" },
          headerTintColor: isDark ? "#E6E1E5" : "#1C1B1F",
        }}
      />
    </ThemeProvider>
  );
}
```

---

## GOOD vs BAD Patterns

| Pattern | GOOD | BAD |
|---------|------|-----|
| Color tokens | `className="bg-primary text-on-primary"` | `className="bg-blue-500 text-white"` |
| Component variants | `<Button variant="outline" />` | `<OutlineButton />` separate component |
| Conditional classes | `cn("p-md", isActive && "bg-primary")` | `className={"p-md " + (isActive ? "bg-primary" : "")}` |
| Theme access | `const { isDark } = useTheme()` | `import { colors } from "@/theme"` directly |
| Dark mode | `className="bg-white dark:bg-gray-900"` | `style={{ backgroundColor: isDark ? '#000' : '#fff' }}` |
| Spacing | `className="p-md gap-sm"` | `style={{ padding: 16, gap: 8 }}` |
| Press feedback | `<Pressable className="active:bg-primary-600">` | `<TouchableOpacity activeOpacity={0.7}>` |

---

## Related Resources

- [Styling Guide](../../guides/styling-guide.md) -- NativeWind setup and patterns
- [Component Patterns](../../guides/component-patterns.md) -- reusable component architecture
- [Glassmorphism Component](../development/glassmorphism-component.md) -- advanced visual effect skill
