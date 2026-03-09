# Accessibility Guide

Comprehensive guide for building accessible React Native applications that comply with WCAG 2.1 AA standards.

## Table of Contents

- [React Native Accessibility API](#react-native-accessibility-api)
- [Interactive Elements](#interactive-elements)
- [Navigation Accessibility](#navigation-accessibility)
- [Forms Accessibility](#forms-accessibility)
- [Color and Visual](#color-and-visual)
- [Touch and Interaction](#touch-and-interaction)
- [Dynamic Type](#dynamic-type)
- [Motion and Animation](#motion-and-animation)
- [Testing](#testing)

---

## React Native Accessibility API

### Core Props

Every meaningful element should expose accessibility information:

```tsx
import { View, Text, Pressable } from 'react-native';

function ProductCard({ product }: { product: Product }) {
  return (
    <View
      accessible
      accessibilityRole="summary"
      accessibilityLabel={`${product.name}, $${product.price}`}
    >
      <Text>{product.name}</Text>
      <Text>${product.price}</Text>
    </View>
  );
}
```

### Key Accessibility Props

| Prop | Purpose | Example |
|------|---------|---------|
| `accessibilityLabel` | Screen reader text (replaces visual text) | `"Delete item"` |
| `accessibilityRole` | Element type for assistive tech | `"button"`, `"header"`, `"link"` |
| `accessibilityHint` | Describes action result | `"Removes item from cart"` |
| `accessibilityState` | Current state | `{ disabled, selected, checked, expanded }` |
| `accessibilityValue` | Current value | `{ min, max, now, text }` for sliders |
| `accessibilityLiveRegion` | Announce dynamic changes (Android) | `"polite"`, `"assertive"` |
| `accessibilityViewIsModal` | Limit focus to modal (iOS) | `true` on modal containers |

### AccessibilityInfo API

```typescript
import { AccessibilityInfo, Platform } from 'react-native';

// Check screen reader status
const isScreenReaderEnabled = await AccessibilityInfo.isScreenReaderEnabled();

// Announce dynamic changes
AccessibilityInfo.announceForAccessibility('Item added to cart');

// Check reduced motion preference
const isReduceMotionEnabled = await AccessibilityInfo.isReduceMotionEnabled();

// Listen for changes
const subscription = AccessibilityInfo.addEventListener(
  'screenReaderChanged',
  (isEnabled: boolean) => {
    // Adapt UI for screen reader
  }
);

// Clean up
subscription.remove();
```

---

## Interactive Elements

### Pressable Buttons

```tsx
// GOOD: Accessible Pressable with role and label
function DeleteButton({ onDelete, itemName }: DeleteButtonProps) {
  return (
    <Pressable
      onPress={onDelete}
      accessibilityRole="button"
      accessibilityLabel={`Delete ${itemName}`}
      accessibilityHint="Removes this item permanently"
      className="rounded-lg bg-red-500 px-4 py-3"
    >
      <TrashIcon />
    </Pressable>
  );
}

// BAD: No accessibility info — screen reader announces nothing useful
function DeleteButton({ onDelete }: DeleteButtonProps) {
  return (
    <Pressable onPress={onDelete} className="rounded-lg bg-red-500 px-4 py-3">
      <TrashIcon />
    </Pressable>
  );
}
```

### TextInput

```tsx
// GOOD: Labeled input with error association
function EmailInput({ error }: { error?: string }) {
  return (
    <View>
      <Text nativeID="email-label">Email Address</Text>
      <TextInput
        accessibilityLabel="Email Address"
        accessibilityLabelledBy="email-label"
        accessibilityState={{ disabled: false }}
        accessibilityErrorMessage={error}
        keyboardType="email-address"
        autoComplete="email"
        textContentType="emailAddress"
        className="rounded-lg border border-gray-300 px-4 py-3"
      />
      {error && (
        <Text accessibilityLiveRegion="polite" className="mt-1 text-red-500">
          {error}
        </Text>
      )}
    </View>
  );
}
```

### Switch / Toggle

```tsx
function NotificationToggle({ enabled, onToggle }: ToggleProps) {
  return (
    <View className="flex-row items-center justify-between">
      <Text nativeID="notif-label">Push Notifications</Text>
      <Switch
        value={enabled}
        onValueChange={onToggle}
        accessibilityLabel="Push Notifications"
        accessibilityRole="switch"
        accessibilityState={{ checked: enabled }}
      />
    </View>
  );
}
```

---

## Navigation Accessibility

### Screen Headers

```tsx
// Mark headers for screen readers to identify sections
function ScreenHeader({ title }: { title: string }) {
  return (
    <Text
      accessibilityRole="header"
      className="text-2xl font-bold text-gray-900"
    >
      {title}
    </Text>
  );
}
```

### Focus Management on Navigation

```typescript
import { useRef, useEffect } from 'react';
import { AccessibilityInfo, findNodeHandle } from 'react-native';
import { useFocusEffect } from '@react-navigation/native';

function DetailsScreen() {
  const headerRef = useRef<Text>(null);

  // Move screen reader focus to header when screen appears
  useFocusEffect(() => {
    const node = findNodeHandle(headerRef.current);
    if (node) {
      AccessibilityInfo.setAccessibilityFocus(node);
    }
  });

  return (
    <View>
      <Text ref={headerRef} accessibilityRole="header">
        Order Details
      </Text>
    </View>
  );
}
```

### Tab Bar Labels

```typescript
// In your tab navigator configuration
<Tab.Screen
  name="Home"
  component={HomeScreen}
  options={{
    tabBarAccessibilityLabel: 'Home tab',
    tabBarLabel: 'Home',
  }}
/>
```

---

## Forms Accessibility

### Label-Input Association

```tsx
function ContactForm() {
  return (
    <View>
      {/* Associate label with input via nativeID */}
      <Text nativeID="name-label">Full Name</Text>
      <TextInput
        accessibilityLabelledBy="name-label"
        accessibilityLabel="Full Name"
        className="rounded-lg border border-gray-300 px-4 py-3"
      />

      {/* Required field indication */}
      <Text nativeID="phone-label">
        Phone Number <Text accessibilityLabel="required">*</Text>
      </Text>
      <TextInput
        accessibilityLabelledBy="phone-label"
        accessibilityLabel="Phone Number, required"
        keyboardType="phone-pad"
        className="rounded-lg border border-gray-300 px-4 py-3"
      />
    </View>
  );
}
```

### Error Announcements

```tsx
function FormField({ label, error, children }: FormFieldProps) {
  return (
    <View>
      <Text>{label}</Text>
      {children}
      {error && (
        <Text
          accessibilityRole="alert"
          accessibilityLiveRegion="assertive"
          className="mt-1 text-sm text-red-500"
        >
          {error}
        </Text>
      )}
    </View>
  );
}
```

### Keyboard Configuration

```tsx
// Set appropriate keyboard types and return keys for form flow
<TextInput
  accessibilityLabel="Email"
  keyboardType="email-address"
  returnKeyType="next"
  onSubmitEditing={() => passwordRef.current?.focus()}
  autoComplete="email"
  textContentType="emailAddress"
/>
<TextInput
  ref={passwordRef}
  accessibilityLabel="Password"
  secureTextEntry
  returnKeyType="done"
  onSubmitEditing={handleSubmit}
  autoComplete="password"
  textContentType="password"
/>
```

---

## Color and Visual

### Contrast Requirements

| Element | Minimum Ratio (WCAG AA) |
|---------|------------------------|
| Normal text (< 18pt) | 4.5:1 |
| Large text (>= 18pt or 14pt bold) | 3:1 |
| UI components and graphical objects | 3:1 |

```tsx
// GOOD: Sufficient contrast
<Text className="text-gray-900">Dark text on light background</Text> // ~15:1

// BAD: Insufficient contrast
<Text className="text-gray-400">Light gray on white</Text> // ~3:1 — fails AA
```

### Don't Convey Information by Color Alone

```tsx
// GOOD: Color + icon + text
function StatusBadge({ status }: { status: 'success' | 'error' }) {
  return (
    <View className="flex-row items-center gap-1">
      {status === 'success' ? (
        <CheckIcon className="text-green-600" />
      ) : (
        <XIcon className="text-red-600" />
      )}
      <Text className={status === 'success' ? 'text-green-600' : 'text-red-600'}>
        {status === 'success' ? 'Completed' : 'Failed'}
      </Text>
    </View>
  );
}

// BAD: Color alone conveys meaning
function StatusDot({ isActive }: { isActive: boolean }) {
  return <View className={isActive ? 'bg-green-500' : 'bg-red-500'} />;
}
```

### Dark Mode Considerations

- Test all color combinations in both light and dark modes
- Use semantic color tokens that adapt to color scheme
- Ensure focus indicators are visible in both modes

---

## Touch and Interaction

### Minimum Touch Targets

All interactive elements must have a minimum size of 44x44 points:

```tsx
// GOOD: Meets 44x44 minimum
<Pressable
  onPress={onPress}
  accessibilityRole="button"
  accessibilityLabel="Close"
  className="h-11 w-11 items-center justify-center"
>
  <CloseIcon size={20} />
</Pressable>

// GOOD: Small visual element with hitSlop to expand touch area
<Pressable
  onPress={onPress}
  accessibilityRole="button"
  accessibilityLabel="Info"
  hitSlop={{ top: 12, bottom: 12, left: 12, right: 12 }}
>
  <InfoIcon size={20} />
</Pressable>

// BAD: Touch target too small
<Pressable onPress={onPress} className="h-6 w-6">
  <CloseIcon size={16} />
</Pressable>
```

### Gesture Alternatives

```tsx
// GOOD: Swipe action with button alternative
function SwipeableItem({ item, onDelete }: SwipeableItemProps) {
  return (
    <View>
      <SwipeableRow onSwipeLeft={onDelete}>
        <Text>{item.name}</Text>
      </SwipeableRow>
      {/* Accessible alternative for users who can't swipe */}
      <Pressable
        onPress={onDelete}
        accessibilityRole="button"
        accessibilityLabel={`Delete ${item.name}`}
      >
        <TrashIcon />
      </Pressable>
    </View>
  );
}
```

---

## Dynamic Type

### Font Scaling

```tsx
// GOOD: Allows font scaling with a reasonable maximum
<Text
  allowFontScaling
  maxFontSizeMultiplier={1.5}
  className="text-base text-gray-900"
>
  This text scales with system font size
</Text>

// BAD: Disabling font scaling entirely
<Text allowFontScaling={false}>
  This text never scales — inaccessible to users with low vision
</Text>
```

### Testing with Large Fonts

1. **iOS:** Settings > Accessibility > Display & Text Size > Larger Text
2. **Android:** Settings > Accessibility > Font size
3. **Verify:** No text truncation, no layout overflow, scrollable content works

---

## Motion and Animation

### Reduced Motion

```typescript
import { useEffect, useState } from 'react';
import { AccessibilityInfo } from 'react-native';

function useReducedMotion(): boolean {
  const [reduceMotion, setReduceMotion] = useState(false);

  useEffect(() => {
    AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion);

    const subscription = AccessibilityInfo.addEventListener(
      'reduceMotionChanged',
      setReduceMotion
    );

    return () => subscription.remove();
  }, []);

  return reduceMotion;
}
```

```tsx
import Animated, { FadeIn, withTiming } from 'react-native-reanimated';

function AnimatedCard({ children }: { children: React.ReactNode }) {
  const reduceMotion = useReducedMotion();

  return (
    <Animated.View
      entering={reduceMotion ? undefined : FadeIn.duration(300)}
      className="rounded-xl bg-white p-4 shadow-md"
    >
      {children}
    </Animated.View>
  );
}
```

### Auto-Playing Content

```tsx
// GOOD: Respects reduced motion, provides pause control
function AutoCarousel({ items }: { items: CarouselItem[] }) {
  const reduceMotion = useReducedMotion();
  const [isPaused, setIsPaused] = useState(reduceMotion);

  return (
    <View>
      <Carousel autoPlay={!isPaused} items={items} />
      <Pressable
        onPress={() => setIsPaused(!isPaused)}
        accessibilityRole="button"
        accessibilityLabel={isPaused ? 'Play carousel' : 'Pause carousel'}
      >
        <Text>{isPaused ? 'Play' : 'Pause'}</Text>
      </Pressable>
    </View>
  );
}
```

---

## Testing

### VoiceOver Testing (iOS)

1. Enable VoiceOver: Settings > Accessibility > VoiceOver (or triple-click side button)
2. Navigate through the app by swiping right (next element) and left (previous)
3. Verify:
   - Every interactive element is reachable
   - Labels are descriptive and concise
   - Roles are announced correctly ("button", "heading", etc.)
   - State changes are announced ("selected", "disabled")
   - Focus order follows visual layout
   - No orphaned or hidden elements that should be accessible

### TalkBack Testing (Android)

1. Enable TalkBack: Settings > Accessibility > TalkBack
2. Navigate by swiping right/left or using explore by touch
3. Verify the same checklist as VoiceOver above
4. Additionally check:
   - `accessibilityLiveRegion` announces dynamic updates
   - Custom actions are announced via Actions menu

### Automated Testing

```typescript
// Use RNTL queries that enforce accessibility
import { render, screen } from '@testing-library/react-native';

it('should have accessible form fields', () => {
  render(<LoginForm />);

  // These queries verify accessibility props exist
  expect(screen.getByRole('button', { name: 'Sign In' })).toBeOnTheScreen();
  expect(screen.getByLabelText('Email')).toBeOnTheScreen();
  expect(screen.getByLabelText('Password')).toBeOnTheScreen();
});

it('should announce errors to screen readers', async () => {
  render(<LoginForm />);

  await userEvent.setup().press(screen.getByRole('button', { name: 'Sign In' }));

  // Error text should have alert role for announcement
  const error = screen.getByRole('alert');
  expect(error).toHaveTextContent('Email is required');
});
```

### Accessibility Checklist

- [ ] All interactive elements have `accessibilityRole` and `accessibilityLabel`
- [ ] Touch targets are at least 44x44 points
- [ ] Color contrast meets WCAG AA (4.5:1 for text, 3:1 for UI)
- [ ] Information is not conveyed by color alone
- [ ] Font scaling works up to 1.5x without layout breaking
- [ ] Reduced motion preference is respected
- [ ] Focus order is logical and follows visual layout
- [ ] Dynamic content changes are announced
- [ ] Forms have labeled inputs and announce errors
- [ ] Gesture-based actions have accessible alternatives

---

## Related Resources

- [Accessible Component Skill](../skills/development/accessible-component.md) — building accessible components
- [Component Patterns Guide](./component-patterns.md) — reusable component architecture
