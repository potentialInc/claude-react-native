# Accessible Component Builder

Guide for building accessible React Native components that work with screen readers and meet WCAG 2.1 AA standards.

## When to Use

- Creating new interactive components
- Auditing existing components for accessibility
- When asked about "accessibility", "a11y", "screen reader", "VoiceOver", "TalkBack"

---

## Core Accessibility Props

### accessibilityLabel

Required on all interactive elements without visible text. The label should describe the element's purpose clearly:

**GOOD:**
```tsx
<Pressable accessibilityLabel="Delete item" accessibilityRole="button" onPress={onDelete}>
  <TrashIcon />
</Pressable>

<Pressable accessibilityLabel={`Remove ${item.name} from cart`} accessibilityRole="button" onPress={() => removeItem(item.id)}>
  <CloseIcon />
</Pressable>
```

**BAD:**
```tsx
<Pressable accessibilityLabel="button1" onPress={onDelete}>
  <TrashIcon />
</Pressable>

<Pressable accessibilityLabel="X" onPress={() => removeItem(item.id)}>
  <CloseIcon />
</Pressable>
```

### accessibilityRole

Map every interactive element to the correct semantic role:

| Role         | Use Case                            |
|-------------|-------------------------------------|
| `button`    | Tappable actions                    |
| `link`      | Navigation to another screen/URL    |
| `header`    | Section headings                    |
| `search`    | Search input containers             |
| `image`     | Informative images                  |
| `text`      | Static text blocks                  |
| `adjustable`| Sliders, steppers                   |
| `checkbox`  | Multi-select toggles                |
| `radio`     | Single-select options               |
| `switch`    | On/off toggles                      |
| `tab`       | Tab bar items                       |
| `summary`   | Summary/expandable triggers         |

**GOOD:**
```tsx
<Pressable accessibilityRole="button" onPress={onSubmit}>
  <Text>Submit</Text>
</Pressable>
```

**BAD:**
```tsx
<Pressable onPress={onSubmit}>
  <Text>Submit</Text>
</Pressable>
```

### accessibilityHint

Describe the result of the action, not the gesture:

**GOOD:**
```tsx
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Profile"
  accessibilityHint="Navigates to your profile settings"
  onPress={goToProfile}
>
  <ProfileIcon />
</Pressable>
```

**BAD:**
```tsx
<Pressable
  accessibilityHint="Tap to open"
  onPress={goToProfile}
>
  <ProfileIcon />
</Pressable>
```

### accessibilityState

Communicate dynamic state to screen readers:

```tsx
// Toggle / Checkbox
<Pressable
  accessibilityRole="checkbox"
  accessibilityLabel="Accept terms and conditions"
  accessibilityState={{ checked: isChecked }}
  onPress={() => setIsChecked(!isChecked)}
  className={cn("flex-row items-center gap-sm p-sm", isChecked && "bg-primary-50")}
>
  <CheckboxIcon checked={isChecked} />
  <Text className="text-on-surface dark:text-on-surface-dark">Accept terms</Text>
</Pressable>

// Disabled button
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Submit form"
  accessibilityState={{ disabled: !isValid, busy: isSubmitting }}
  disabled={!isValid || isSubmitting}
  onPress={handleSubmit}
  className={cn("bg-primary p-md rounded-md", !isValid && "opacity-50")}
>
  <Text className="text-on-primary">{isSubmitting ? "Submitting..." : "Submit"}</Text>
</Pressable>

// Expandable section
<Pressable
  accessibilityRole="button"
  accessibilityLabel="FAQ details"
  accessibilityState={{ expanded: isExpanded }}
  onPress={() => setIsExpanded(!isExpanded)}
>
  <Text className="font-sans-semibold">Details</Text>
</Pressable>
```

### accessibilityValue

For elements with numeric or text values:

```tsx
// Slider
<Slider
  accessibilityRole="adjustable"
  accessibilityLabel="Volume"
  accessibilityValue={{ min: 0, max: 100, now: volume, text: `${volume} percent` }}
  value={volume}
  onValueChange={setVolume}
/>

// Progress bar
<View
  accessibilityRole="progressbar"
  accessibilityLabel="Upload progress"
  accessibilityValue={{ min: 0, max: 100, now: progress, text: `${progress}% complete` }}
>
  <View className="h-2 rounded-full bg-gray-200 dark:bg-gray-700">
    <View className={cn("h-2 rounded-full bg-primary")} style={{ width: `${progress}%` }} />
  </View>
</View>
```

---

## Screen Reader Testing

### VoiceOver (iOS)

**Setup:** Settings > Accessibility > VoiceOver > Enable

**Key gestures:**
- Swipe right/left: Move to next/previous element
- Double-tap: Activate focused element
- Three-finger swipe: Scroll
- Rotor (two-finger rotate): Change navigation mode (headings, links, form controls)

**Common issues to check:**
- Missing labels on icon-only buttons
- Wrong reading order (elements read in unexpected sequence)
- Decorative images announced unnecessarily
- Custom components not reachable by swipe navigation

### TalkBack (Android)

**Setup:** Settings > Accessibility > TalkBack > Enable

**Key gestures:**
- Swipe right/left: Move to next/previous element
- Double-tap: Activate focused element
- Explore by touch: Drag finger to hear what is under it
- Swipe up then right: Open context menu

**Common issues to check:**
- `importantForAccessibility="no"` not set on decorative elements
- Live regions not announcing dynamic content updates
- Focus not moving to new content after navigation

---

## Accessible Component Patterns

### Accessible Button

```typescript
// components/ui/AccessibleButton.tsx
import React from "react";
import { AccessibilityInfo, ActivityIndicator, Pressable, Text, View } from "react-native";
import { cn } from "@/lib/utils";

interface AccessibleButtonProps {
  label: string;
  hint?: string;
  onPress: () => void;
  disabled?: boolean;
  loading?: boolean;
  variant?: "primary" | "outline";
  children?: React.ReactNode;
}

export function AccessibleButton({
  label,
  hint,
  onPress,
  disabled = false,
  loading = false,
  variant = "primary",
  children,
}: AccessibleButtonProps) {
  const isDisabled = disabled || loading;

  const handlePress = () => {
    onPress();
  };

  return (
    <Pressable
      onPress={handlePress}
      disabled={isDisabled}
      accessibilityRole="button"
      accessibilityLabel={loading ? `${label}, loading` : label}
      accessibilityHint={hint}
      accessibilityState={{
        disabled: isDisabled,
        busy: loading,
      }}
      className={cn(
        "min-h-[44px] min-w-[44px] flex-row items-center justify-center rounded-md px-4 py-2.5",
        variant === "primary" && "bg-primary active:bg-primary-600",
        variant === "outline" && "border border-outline active:bg-surface-dim",
        isDisabled && "opacity-50",
      )}
    >
      {loading && (
        <ActivityIndicator
          size="small"
          color={variant === "primary" ? "#FFFFFF" : "#2563EB"}
          className="mr-sm"
        />
      )}
      {children ?? (
        <Text
          className={cn(
            "font-sans-medium text-sm",
            variant === "primary" && "text-on-primary",
            variant === "outline" && "text-primary",
          )}
        >
          {label}
        </Text>
      )}
    </Pressable>
  );
}
```

### Accessible Form

```typescript
// components/forms/AccessibleForm.tsx
import React from "react";
import { AccessibilityInfo, Platform, View, Text } from "react-native";
import { TextInput } from "react-native-paper";
import { useForm, useController, type Control, type FieldValues, type Path } from "react-hook-form";
import { cn } from "@/lib/utils";

interface AccessibleFieldProps<T extends FieldValues> {
  name: Path<T>;
  control: Control<T>;
  label: string;
  hint?: string;
  required?: boolean;
}

export function AccessibleField<T extends FieldValues>({
  name,
  control,
  label,
  hint,
  required = false,
}: AccessibleFieldProps<T>) {
  const {
    field: { value, onChange, onBlur },
    fieldState: { error },
  } = useController({ name, control });

  // Announce errors to screen readers
  React.useEffect(() => {
    if (error?.message) {
      AccessibilityInfo.announceForAccessibility(`Error: ${label} ${error.message}`);
    }
  }, [error?.message, label]);

  const accessibilityLabel = required ? `${label}, required` : label;

  return (
    <View className="mb-md">
      <TextInput
        label={label}
        value={value}
        onChangeText={onChange}
        onBlur={onBlur}
        error={!!error}
        mode="outlined"
        accessibilityLabel={accessibilityLabel}
        accessibilityHint={hint}
        accessibilityState={{ disabled: false }}
        className="bg-surface dark:bg-surface-dark"
      />
      {error?.message && (
        <Text
          accessibilityRole="alert"
          accessibilityLiveRegion={Platform.OS === "android" ? "assertive" : undefined}
          className="mt-xs text-xs text-error"
        >
          {error.message}
        </Text>
      )}
    </View>
  );
}
```

### Accessible Modal

```typescript
// components/ui/AccessibleModal.tsx
import React, { useEffect, useRef } from "react";
import { AccessibilityInfo, Modal, Pressable, View, Text, findNodeHandle } from "react-native";
import { cn } from "@/lib/utils";

interface AccessibleModalProps {
  visible: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

export function AccessibleModal({ visible, onClose, title, children }: AccessibleModalProps) {
  const titleRef = useRef<Text>(null);

  useEffect(() => {
    if (visible && titleRef.current) {
      // Move screen reader focus to modal title when opened
      const node = findNodeHandle(titleRef.current);
      if (node) {
        AccessibilityInfo.setAccessibilityFocus(node);
      }
    }
  }, [visible]);

  return (
    <Modal
      visible={visible}
      transparent
      animationType="fade"
      onRequestClose={onClose}
    >
      <Pressable
        className="flex-1 items-center justify-center bg-black/50"
        onPress={onClose}
        accessibilityRole="button"
        accessibilityLabel="Close modal"
      >
        <View
          className="mx-md w-full max-w-sm rounded-lg bg-surface p-lg dark:bg-surface-dark"
          accessibilityViewIsModal={true}
          importantForAccessibility="yes"
          onStartShouldSetResponder={() => true}
        >
          <Text
            ref={titleRef}
            accessibilityRole="header"
            className="mb-md font-sans-semibold text-lg text-on-surface dark:text-on-surface-dark"
          >
            {title}
          </Text>

          {children}

          <Pressable
            onPress={onClose}
            accessibilityRole="button"
            accessibilityLabel="Close"
            accessibilityHint="Closes this dialog"
            className="mt-lg min-h-[44px] items-center justify-center rounded-md bg-primary px-4 py-2.5"
          >
            <Text className="font-sans-medium text-on-primary">Close</Text>
          </Pressable>
        </View>
      </Pressable>
    </Modal>
  );
}
```

### Accessible List

```typescript
// Accessible FlatList with announcements
import React from "react";
import { FlatList, Text, View } from "react-native";

interface Item {
  id: string;
  title: string;
}

interface AccessibleListProps {
  items: Item[];
  headerTitle: string;
}

export function AccessibleList({ items, headerTitle }: AccessibleListProps) {
  return (
    <FlatList
      data={items}
      accessibilityRole="list"
      accessibilityLabel={`${headerTitle}, ${items.length} items`}
      ListHeaderComponent={
        <Text
          accessibilityRole="header"
          className="mb-sm px-md font-sans-semibold text-lg text-on-surface dark:text-on-surface-dark"
        >
          {headerTitle}
        </Text>
      }
      ListEmptyComponent={
        <Text
          accessibilityLiveRegion="polite"
          className="p-lg text-center text-on-surface-variant dark:text-on-surface-dark-variant"
        >
          No items found
        </Text>
      }
      renderItem={({ item, index }) => (
        <View
          accessibilityLabel={`${item.title}, item ${index + 1} of ${items.length}`}
          className="border-b border-outline-variant px-md py-sm dark:border-outline"
        >
          <Text className="text-on-surface dark:text-on-surface-dark">{item.title}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}
    />
  );
}
```

### Accessible Image

```tsx
{/* Informative image: describe content */}
<Image
  source={{ uri: user.avatarUrl }}
  accessibilityRole="image"
  accessibilityLabel={`Profile photo of ${user.displayName}`}
  className="h-12 w-12 rounded-full"
/>

{/* Decorative image: hide from screen reader */}
<Image
  source={require("@/assets/decorative-wave.png")}
  accessibilityElementsHidden={true}
  importantForAccessibility="no-hide-descendants"
  className="absolute bottom-0 h-24 w-full"
/>
```

---

## Design Requirements

### Color Contrast

WCAG AA minimum contrast ratios:

| Text Type  | Minimum Ratio | Example                              |
|-----------|---------------|--------------------------------------|
| Normal text (< 18px) | 4.5:1 | `#49454F` on `#FFFFFF` = 7.1:1     |
| Large text (>= 18px bold or >= 24px) | 3:1 | `#79747E` on `#FFFFFF` = 4.6:1 |
| UI components & icons | 3:1 | Border, focus indicator              |

**GOOD:**
```tsx
<Text className="text-on-surface dark:text-on-surface-dark">
  High contrast text on surface
</Text>
```

**BAD:**
```tsx
<Text className="text-gray-400">
  Low contrast text that may fail WCAG AA
</Text>
```

### Touch Targets

Minimum interactive area: 44x44pt (iOS) / 48x48dp (Android):

**GOOD:**
```tsx
<Pressable
  onPress={onPress}
  accessibilityRole="button"
  className="min-h-[44px] min-w-[44px] items-center justify-center p-sm"
>
  <Icon size={24} />
</Pressable>

{/* Small visual element with expanded hit area */}
<Pressable
  onPress={onPress}
  hitSlop={{ top: 12, bottom: 12, left: 12, right: 12 }}
  accessibilityRole="button"
  accessibilityLabel="More options"
>
  <Icon size={16} />
</Pressable>
```

**BAD:**
```tsx
<Pressable onPress={onPress} className="h-6 w-6">
  <Icon size={16} />
</Pressable>
```

### Dynamic Type

Support system font scaling without breaking layout:

```tsx
// Allow scaling with a cap to prevent layout overflow
<Text
  allowFontScaling={true}
  maxFontSizeMultiplier={1.5}
  className="text-base text-on-surface dark:text-on-surface-dark"
>
  This text scales up to 150% of base size
</Text>

// Critical UI elements may need tighter limits
<Text
  maxFontSizeMultiplier={1.2}
  className="text-xs font-sans-medium text-on-surface-variant"
>
  Tab label
</Text>
```

### Reduced Motion

Respect user preference for reduced motion:

```typescript
import { useEffect, useState } from "react";
import { AccessibilityInfo } from "react-native";
import Animated, { useAnimatedStyle, withTiming, withSpring } from "react-native-reanimated";

export function useReducedMotion(): boolean {
  const [reduceMotion, setReduceMotion] = useState(false);

  useEffect(() => {
    AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion);

    const subscription = AccessibilityInfo.addEventListener(
      "reduceMotionChanged",
      setReduceMotion,
    );

    return () => subscription.remove();
  }, []);

  return reduceMotion;
}

// Usage in a component
function AnimatedCard({ visible }: { visible: boolean }) {
  const reduceMotion = useReducedMotion();

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: reduceMotion
      ? (visible ? 1 : 0)
      : withTiming(visible ? 1 : 0, { duration: 300 }),
    transform: [
      {
        translateY: reduceMotion
          ? 0
          : withSpring(visible ? 0 : 20),
      },
    ],
  }));

  return (
    <Animated.View style={animatedStyle} className="rounded-lg bg-surface p-md dark:bg-surface-dark">
      <Text className="text-on-surface dark:text-on-surface-dark">Animated content</Text>
    </Animated.View>
  );
}
```

---

## Accessibility Testing

### Manual Testing Checklist

- [ ] Navigate the entire screen using only VoiceOver/TalkBack swipe gestures
- [ ] Verify every interactive element is reachable and has a meaningful label
- [ ] Confirm labels make sense without visual context ("Delete" alone is unclear -- "Delete photo from gallery" is better)
- [ ] Test with system font size set to largest setting (200%)
- [ ] Test with inverted colors and high contrast mode enabled
- [ ] Verify focus moves logically after navigation, modal open/close, and dynamic content changes
- [ ] Check that alerts and errors are announced automatically
- [ ] Confirm decorative elements are hidden from screen reader

### Automated Testing

```typescript
// Detox accessibility assertion
it("should have accessible login button", async () => {
  const loginButton = element(by.label("Log in"));
  await expect(loginButton).toBeVisible();
  await expect(loginButton).toHaveAccessibilityRole("button");
});

// React Native Testing Library
import { render, screen } from "@testing-library/react-native";

it("renders accessible form field", () => {
  render(<AccessibleField name="email" control={control} label="Email address" required />);

  const input = screen.getByLabelText("Email address, required");
  expect(input).toBeTruthy();
});

it("announces error to screen reader", async () => {
  const announceForAccessibility = jest.spyOn(AccessibilityInfo, "announceForAccessibility");

  // Trigger validation error
  fireEvent.press(screen.getByRole("button", { name: "Submit" }));

  await waitFor(() => {
    expect(announceForAccessibility).toHaveBeenCalledWith(
      expect.stringContaining("Error"),
    );
  });
});
```

---

## GOOD vs BAD Summary

| Pattern | GOOD | BAD |
|---------|------|-----|
| Icon button | `accessibilityLabel="Delete item"` with `accessibilityRole="button"` | No label, no role |
| Hint text | `accessibilityHint="Navigates to profile settings"` | `accessibilityHint="Tap to open"` |
| Touch target | `className="min-h-[44px] min-w-[44px]"` | Small tap target without `hitSlop` |
| Decorative image | `accessibilityElementsHidden={true}` | Screen reader reads "image" with no value |
| Error feedback | `AccessibilityInfo.announceForAccessibility(msg)` | Error shown visually only |
| Motion | Check `isReduceMotionEnabled` and provide static fallback | Always animate regardless of preference |
| Color contrast | 4.5:1 ratio minimum for body text | Light gray text on white background |
| Dynamic type | `maxFontSizeMultiplier={1.5}` | `allowFontScaling={false}` |
| Modal focus | `accessibilityViewIsModal` + focus on title | Focus stays behind modal overlay |
| State changes | `accessibilityState={{ checked, disabled }}` | No state communicated to screen reader |

---

## Related Resources

- [Component Patterns](../../guides/component-patterns.md) -- reusable component architecture
- [Design System Setup](./design-system-setup.md) -- theme configuration and tokens
