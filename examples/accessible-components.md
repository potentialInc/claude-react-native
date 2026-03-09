# Accessible Component Examples

Complete accessible component implementations for React Native, following WCAG guidelines and platform-specific best practices.

---

## Example 1: Accessible Form

```typescript
// src/screens/AccessibleLoginScreen.tsx
import { useState, useRef, useCallback } from 'react';
import {
  View,
  TextInput as RNTextInput,
  AccessibilityInfo,
  KeyboardAvoidingView,
  Platform,
  Pressable,
} from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Text, TextInput, Button, HelperText, IconButton } from 'react-native-paper';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const loginSchema = z.object({
  email: z.string().email('Please enter a valid email address'),
  password: z.string().min(1, 'Password is required'),
});

type LoginFormData = z.infer<typeof loginSchema>;

interface AccessibleLoginProps {
  onSubmit: (data: LoginFormData) => Promise<void>;
}

export function AccessibleLoginScreen({ onSubmit }: AccessibleLoginProps) {
  const [passwordVisible, setPasswordVisible] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const passwordInputRef = useRef<RNTextInput>(null);

  const {
    control,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  });

  const announceErrors = useCallback((formErrors: typeof errors) => {
    const errorMessages: string[] = [];
    if (formErrors.email) errorMessages.push(`Email: ${formErrors.email.message}`);
    if (formErrors.password) errorMessages.push(`Password: ${formErrors.password.message}`);

    if (errorMessages.length > 0) {
      AccessibilityInfo.announceForAccessibility(
        `Form has ${errorMessages.length} error${errorMessages.length > 1 ? 's' : ''}. ${errorMessages.join('. ')}`
      );
    }
  }, []);

  const handleFormSubmit = handleSubmit(
    async (data) => {
      setIsSubmitting(true);
      AccessibilityInfo.announceForAccessibility('Signing in, please wait.');
      try {
        await onSubmit(data);
        AccessibilityInfo.announceForAccessibility('Sign in successful.');
      } catch {
        AccessibilityInfo.announceForAccessibility('Sign in failed. Please try again.');
      } finally {
        setIsSubmitting(false);
      }
    },
    (formErrors) => {
      announceErrors(formErrors);
    }
  );

  const togglePasswordVisibility = useCallback(() => {
    setPasswordVisible((prev) => !prev);
  }, []);

  return (
    <SafeAreaView className="flex-1 bg-background">
      <KeyboardAvoidingView
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
        className="flex-1"
      >
        <View className="flex-1 justify-center p-6 gap-4">
          <Text
            variant="headlineLarge"
            className="text-center mb-8"
            accessibilityRole="header"
          >
            Sign In
          </Text>

          {/* Email Field */}
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
                  autoComplete="email"
                  textContentType="emailAddress"
                  returnKeyType="next"
                  onSubmitEditing={() => passwordInputRef.current?.focus()}
                  accessibilityLabel="Email address"
                  accessibilityHint="Enter your email address to sign in"
                />
              )}
            />
            <HelperText type="error" visible={!!errors.email}>
              {errors.email?.message}
            </HelperText>
          </View>

          {/* Password Field with Visibility Toggle */}
          <View>
            <Controller
              control={control}
              name="password"
              render={({ field: { onChange, onBlur, value } }) => (
                <TextInput
                  ref={passwordInputRef}
                  label="Password"
                  mode="outlined"
                  value={value}
                  onChangeText={onChange}
                  onBlur={onBlur}
                  error={!!errors.password}
                  secureTextEntry={!passwordVisible}
                  autoComplete="password"
                  textContentType="password"
                  returnKeyType="done"
                  onSubmitEditing={handleFormSubmit}
                  accessibilityLabel="Password"
                  accessibilityHint="Enter your password to sign in"
                  right={
                    <TextInput.Icon
                      icon={passwordVisible ? 'eye-off' : 'eye'}
                      onPress={togglePasswordVisibility}
                      accessibilityLabel={
                        passwordVisible ? 'Hide password' : 'Show password'
                      }
                      accessibilityRole="button"
                    />
                  }
                />
              )}
            />
            <HelperText type="error" visible={!!errors.password}>
              {errors.password?.message}
            </HelperText>
          </View>

          {/* Submit Button */}
          <Pressable
            onPress={handleFormSubmit}
            disabled={isSubmitting}
            className="bg-primary py-4 rounded-lg items-center active:opacity-80 disabled:opacity-50"
            accessibilityRole="button"
            accessibilityLabel={isSubmitting ? 'Signing in' : 'Sign in'}
            accessibilityState={{ disabled: isSubmitting, busy: isSubmitting }}
          >
            <Text className="text-on-primary font-semibold text-base">
              {isSubmitting ? 'Signing In...' : 'Sign In'}
            </Text>
          </Pressable>
        </View>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}
```

---

## Example 2: Accessible Modal

```typescript
// src/components/AccessibleModal.tsx
import { useEffect, useRef, useCallback, type ReactNode } from 'react';
import {
  View,
  Modal,
  Pressable,
  BackHandler,
  AccessibilityInfo,
  findNodeHandle,
  Platform,
} from 'react-native';
import { Text, IconButton } from 'react-native-paper';

interface AccessibleModalProps {
  visible: boolean;
  onDismiss: () => void;
  title: string;
  children: ReactNode;
}

export function AccessibleModal({
  visible,
  onDismiss,
  title,
  children,
}: AccessibleModalProps) {
  const firstFocusableRef = useRef<View>(null);
  const triggerRef = useRef<View>(null);

  // Focus the first element when modal opens
  useEffect(() => {
    if (visible && firstFocusableRef.current) {
      const reactTag = findNodeHandle(firstFocusableRef.current);
      if (reactTag) {
        AccessibilityInfo.setAccessibilityFocus(reactTag);
      }
    }
  }, [visible]);

  // Return focus to trigger when modal closes
  useEffect(() => {
    if (!visible && triggerRef.current) {
      const reactTag = findNodeHandle(triggerRef.current);
      if (reactTag) {
        AccessibilityInfo.setAccessibilityFocus(reactTag);
      }
    }
  }, [visible]);

  // Handle hardware back button on Android
  useEffect(() => {
    if (!visible) return;

    const subscription = BackHandler.addEventListener('hardwareBackPress', () => {
      onDismiss();
      return true;
    });

    return () => subscription.remove();
  }, [visible, onDismiss]);

  const handleBackdropPress = useCallback(() => {
    onDismiss();
  }, [onDismiss]);

  return (
    <Modal
      visible={visible}
      transparent
      animationType="fade"
      onRequestClose={onDismiss}
      statusBarTranslucent
    >
      {/* Backdrop */}
      <Pressable
        className="flex-1 bg-black/50 justify-center items-center p-6"
        onPress={handleBackdropPress}
        accessibilityLabel="Close dialog"
        accessibilityRole="button"
      >
        {/* Modal Content */}
        <Pressable
          className="bg-surface w-full rounded-2xl overflow-hidden max-h-[80%]"
          onPress={(e) => e.stopPropagation()}
          accessibilityViewIsModal={Platform.OS === 'ios'}
          {...(Platform.OS === 'android' && {
            importantForAccessibility: 'yes' as const,
          })}
          accessibilityRole="none"
        >
          {/* Header */}
          <View className="flex-row items-center justify-between px-4 py-3 border-b border-outline-variant">
            <Text
              ref={firstFocusableRef}
              variant="titleLarge"
              accessibilityRole="header"
            >
              {title}
            </Text>
            <IconButton
              icon="close"
              onPress={onDismiss}
              accessibilityLabel="Close dialog"
            />
          </View>

          {/* Body */}
          <View className="p-4">
            {children}
          </View>
        </Pressable>
      </Pressable>
    </Modal>
  );
}
```

### Usage

```typescript
import { useState } from 'react';
import { View, Pressable } from 'react-native';
import { Text, Button } from 'react-native-paper';
import { AccessibleModal } from '~/components/AccessibleModal';

export function SettingsScreen() {
  const [deleteModalVisible, setDeleteModalVisible] = useState(false);

  return (
    <View className="flex-1 p-4">
      <Pressable
        onPress={() => setDeleteModalVisible(true)}
        className="bg-destructive/10 p-4 rounded-lg active:opacity-80"
        accessibilityRole="button"
        accessibilityLabel="Delete account"
        accessibilityHint="Opens a confirmation dialog"
      >
        <Text className="text-destructive font-medium">Delete Account</Text>
      </Pressable>

      <AccessibleModal
        visible={deleteModalVisible}
        onDismiss={() => setDeleteModalVisible(false)}
        title="Delete Account"
      >
        <Text className="mb-4">
          Are you sure you want to delete your account? This action cannot be undone.
        </Text>
        <View className="flex-row gap-3 justify-end">
          <Button mode="outlined" onPress={() => setDeleteModalVisible(false)}>
            Cancel
          </Button>
          <Button mode="contained" buttonColor="#DC2626" textColor="#FFFFFF" onPress={() => {}}>
            Delete
          </Button>
        </View>
      </AccessibleModal>
    </View>
  );
}
```

---

## Example 3: Accessible Navigation Tab Bar

```typescript
// src/components/AccessibleTabBar.tsx
import { View, Pressable } from 'react-native';
import { Text, Icon } from 'react-native-paper';
import type { BottomTabBarProps } from '@react-navigation/bottom-tabs';

interface TabConfig {
  icon: string;
  label: string;
  badgeCount?: number;
}

const TAB_CONFIG: Record<string, TabConfig> = {
  index: { icon: 'home', label: 'Home' },
  messages: { icon: 'message', label: 'Messages', badgeCount: 0 },
  search: { icon: 'magnify', label: 'Search' },
  profile: { icon: 'account', label: 'Profile' },
};

export function AccessibleTabBar({
  state,
  descriptors,
  navigation,
}: BottomTabBarProps) {
  return (
    <View
      className="flex-row bg-surface border-t border-outline-variant pb-6 pt-2"
      accessibilityRole="tablist"
    >
      {state.routes.map((route, index) => {
        const { options } = descriptors[route.key];
        const isFocused = state.index === index;
        const config = TAB_CONFIG[route.name] ?? {
          icon: 'circle',
          label: options.title ?? route.name,
        };

        const badgeCount = config.badgeCount ?? 0;

        // Build an accessible label that includes badge count
        let accessibilityLabel = config.label;
        if (badgeCount > 0) {
          accessibilityLabel = `${config.label}, ${badgeCount} new`;
        }

        const handlePress = () => {
          const event = navigation.emit({
            type: 'tabPress',
            target: route.key,
            canPreventDefault: true,
          });

          if (!isFocused && !event.defaultPrevented) {
            navigation.navigate(route.name);
          }
        };

        const handleLongPress = () => {
          navigation.emit({
            type: 'tabLongPress',
            target: route.key,
          });
        };

        return (
          <Pressable
            key={route.key}
            onPress={handlePress}
            onLongPress={handleLongPress}
            className="flex-1 items-center justify-center py-2 gap-1"
            accessibilityRole="tab"
            accessibilityState={{ selected: isFocused }}
            accessibilityLabel={accessibilityLabel}
          >
            <View className="relative">
              <Icon
                source={config.icon}
                size={24}
                color={isFocused ? '#6366F1' : '#6B7280'}
              />
              {badgeCount > 0 && (
                <View className="absolute -top-1 -right-2 bg-destructive rounded-full min-w-[18px] h-[18px] items-center justify-center px-1">
                  <Text className="text-white text-xs font-bold">
                    {badgeCount > 99 ? '99+' : badgeCount}
                  </Text>
                </View>
              )}
            </View>
            <Text
              className={`text-xs ${isFocused ? 'text-primary font-semibold' : 'text-muted-foreground'}`}
            >
              {config.label}
            </Text>
          </Pressable>
        );
      })}
    </View>
  );
}
```

### Usage

```typescript
// src/app/(tabs)/_layout.tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { AccessibleTabBar } from '~/components/AccessibleTabBar';

const Tab = createBottomTabNavigator();

export default function TabLayout() {
  return (
    <Tab.Navigator
      tabBar={(props) => <AccessibleTabBar {...props} />}
      screenOptions={{ headerShown: true }}
    >
      <Tab.Screen name="index" component={HomeScreen} options={{ title: 'Home' }} />
      <Tab.Screen name="messages" component={MessagesScreen} options={{ title: 'Messages' }} />
      <Tab.Screen name="search" component={SearchScreen} options={{ title: 'Search' }} />
      <Tab.Screen name="profile" component={ProfileScreen} options={{ title: 'Profile' }} />
    </Tab.Navigator>
  );
}
```

---

## Example 4: Accessible List with Section Headers

```typescript
// src/screens/ContactsScreen.tsx
import { useCallback } from 'react';
import { View, SectionList, RefreshControl, AccessibilityInfo } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { Text, Card, Divider } from 'react-native-paper';
import { useQuery } from '@tanstack/react-query';
import { httpService } from '~/services/httpService';

interface Contact {
  id: string;
  name: string;
  phone: string;
}

interface ContactSection {
  title: string;
  data: Contact[];
}

function groupContactsByLetter(contacts: Contact[]): ContactSection[] {
  const grouped = new Map<string, Contact[]>();

  contacts.forEach((contact) => {
    const letter = contact.name.charAt(0).toUpperCase();
    const existing = grouped.get(letter) ?? [];
    grouped.set(letter, [...existing, contact]);
  });

  return Array.from(grouped.entries())
    .sort(([a], [b]) => a.localeCompare(b))
    .map(([title, data]) => ({ title, data }));
}

export function ContactsScreen() {
  const { data: contacts, isLoading, refetch, isRefetching } = useQuery({
    queryKey: ['contacts'],
    queryFn: async () => {
      const response = await httpService.get<Contact[]>('/contacts');
      return response.data;
    },
  });

  const sections = groupContactsByLetter(contacts ?? []);
  const totalCount = contacts?.length ?? 0;

  // Announce item count when data loads
  const handleContentSizeChange = useCallback(() => {
    if (totalCount > 0) {
      AccessibilityInfo.announceForAccessibility(
        `Loaded ${totalCount} contact${totalCount !== 1 ? 's' : ''}`
      );
    }
  }, [totalCount]);

  const handleRefresh = useCallback(async () => {
    await refetch();
    AccessibilityInfo.announceForAccessibility('Contacts refreshed');
  }, [refetch]);

  const renderSectionHeader = useCallback(
    ({ section }: { section: ContactSection }) => (
      <View
        className="bg-background px-4 py-2"
        accessibilityRole="header"
      >
        <Text variant="titleSmall" className="text-muted-foreground font-bold">
          {section.title}
        </Text>
      </View>
    ),
    []
  );

  const renderItem = useCallback(
    ({ item }: { item: Contact }) => (
      <Card className="mx-4 mb-1">
        <Card.Title
          title={item.name}
          subtitle={item.phone}
          accessibilityLabel={`${item.name}, phone number ${item.phone}`}
        />
      </Card>
    ),
    []
  );

  const renderEmpty = useCallback(
    () => (
      <View
        className="flex-1 items-center justify-center p-8"
        accessible
        accessibilityLabel="No contacts found"
        accessibilityRole="text"
      >
        <Text className="text-muted-foreground text-lg">No contacts found</Text>
        <Text className="text-muted-foreground mt-2 text-center">
          Your contacts will appear here once added.
        </Text>
      </View>
    ),
    []
  );

  if (isLoading) {
    return (
      <SafeAreaView className="flex-1 bg-background items-center justify-center">
        <Text accessibilityRole="text" accessibilityLabel="Loading contacts">
          Loading contacts...
        </Text>
      </SafeAreaView>
    );
  }

  return (
    <SafeAreaView className="flex-1 bg-background">
      <SectionList
        sections={sections}
        keyExtractor={(item) => item.id}
        renderSectionHeader={renderSectionHeader}
        renderItem={renderItem}
        ListEmptyComponent={renderEmpty}
        ItemSeparatorComponent={Divider}
        stickySectionHeadersEnabled
        onContentSizeChange={handleContentSizeChange}
        refreshControl={
          <RefreshControl
            refreshing={isRefetching}
            onRefresh={handleRefresh}
            accessibilityLabel="Pull to refresh contacts"
          />
        }
      />
    </SafeAreaView>
  );
}
```

---

## Example 5: Dynamic Type Support

```typescript
// src/components/DynamicTypeCard.tsx
import { View, Pressable, useWindowDimensions } from 'react-native';
import { Text, Card, Icon } from 'react-native-paper';
import { useReducedMotion } from 'react-native-reanimated';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
} from 'react-native-reanimated';

interface DynamicTypeCardProps {
  title: string;
  description: string;
  icon: string;
  onPress: () => void;
}

/**
 * A card component that adapts to Dynamic Type / font scaling settings.
 * - Uses flexWrap and minHeight instead of fixed dimensions
 * - Respects allowFontScaling with maxFontSizeMultiplier to prevent overflow
 * - Provides reduced-motion alternatives for animations
 */
export function DynamicTypeCard({
  title,
  description,
  icon,
  onPress,
}: DynamicTypeCardProps) {
  const { fontScale } = useWindowDimensions();
  const shouldReduceMotion = useReducedMotion();
  const scale = useSharedValue(1);

  const isLargeText = fontScale > 1.3;

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  const handlePressIn = () => {
    if (shouldReduceMotion) {
      // Skip scale animation for reduced motion preference
      return;
    }
    scale.value = withSpring(0.97);
  };

  const handlePressOut = () => {
    if (shouldReduceMotion) {
      return;
    }
    scale.value = withTiming(1, { duration: 150 });
  };

  return (
    <Pressable
      onPress={onPress}
      onPressIn={handlePressIn}
      onPressOut={handlePressOut}
      accessibilityRole="button"
      accessibilityLabel={`${title}. ${description}`}
    >
      <Animated.View style={animatedStyle}>
        <Card className="overflow-hidden">
          <View
            className={`p-4 gap-3 ${isLargeText ? 'flex-col' : 'flex-row items-center'}`}
          >
            {/* Icon container -- fixed minimum size */}
            <View className="items-center justify-center min-w-[48px] min-h-[48px]">
              <Icon source={icon} size={32} />
            </View>

            {/* Text container -- flexible, wraps at large font scales */}
            <View className="flex-1 flex-shrink gap-1">
              <Text
                variant="titleMedium"
                className="font-semibold"
                allowFontScaling
                maxFontSizeMultiplier={1.8}
                numberOfLines={isLargeText ? 3 : 2}
              >
                {title}
              </Text>
              <Text
                variant="bodyMedium"
                className="text-muted-foreground"
                allowFontScaling
                maxFontSizeMultiplier={1.6}
                numberOfLines={isLargeText ? 5 : 3}
              >
                {description}
              </Text>
            </View>
          </View>
        </Card>
      </Animated.View>
    </Pressable>
  );
}
```

### Testing with Different Font Scales

```typescript
// src/components/__tests__/DynamicTypeCard.test.tsx
import { render, screen } from '@testing-library/react-native';
import { useWindowDimensions } from 'react-native';
import { PaperProvider } from 'react-native-paper';
import { DynamicTypeCard } from '../DynamicTypeCard';

jest.mock('react-native/Libraries/Utilities/useWindowDimensions', () =>
  jest.fn(() => ({ width: 375, height: 812, fontScale: 1.0 }))
);

jest.mock('react-native-reanimated', () => {
  const actual = jest.requireActual('react-native-reanimated/mock');
  return {
    ...actual,
    useReducedMotion: jest.fn(() => false),
  };
});

const defaultProps = {
  title: 'Settings',
  description: 'Manage your account preferences and app configuration',
  icon: 'cog',
  onPress: jest.fn(),
};

function renderCard(fontScale = 1.0) {
  (useWindowDimensions as jest.Mock).mockReturnValue({
    width: 375,
    height: 812,
    fontScale,
  });

  return render(
    <PaperProvider>
      <DynamicTypeCard {...defaultProps} />
    </PaperProvider>
  );
}

describe('DynamicTypeCard', () => {
  it('renders in row layout at normal font scale', () => {
    renderCard(1.0);
    expect(screen.getByText('Settings')).toBeOnTheScreen();
    expect(screen.getByText(defaultProps.description)).toBeOnTheScreen();
  });

  it('switches to column layout at large font scale', () => {
    renderCard(1.5);
    expect(screen.getByText('Settings')).toBeOnTheScreen();
    // At fontScale > 1.3, the component uses flex-col layout
  });

  it('has correct accessibility label', () => {
    renderCard(1.0);
    expect(
      screen.getByLabelText('Settings. Manage your account preferences and app configuration')
    ).toBeOnTheScreen();
  });
});
```

### useReducedMotion Usage

```typescript
// src/components/AnimatedHeader.tsx
import { View } from 'react-native';
import { Text } from 'react-native-paper';
import Animated, {
  useAnimatedScrollHandler,
  useSharedValue,
  useAnimatedStyle,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';
import { useReducedMotion } from 'react-native-reanimated';

interface AnimatedHeaderProps {
  title: string;
}

export function AnimatedHeader({ title }: AnimatedHeaderProps) {
  const scrollY = useSharedValue(0);
  const shouldReduceMotion = useReducedMotion();

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  const headerStyle = useAnimatedStyle(() => {
    if (shouldReduceMotion) {
      // No animation -- just toggle between two states
      return {
        height: scrollY.value > 50 ? 60 : 120,
        opacity: 1,
      };
    }

    return {
      height: interpolate(
        scrollY.value,
        [0, 100],
        [120, 60],
        Extrapolation.CLAMP
      ),
      opacity: interpolate(
        scrollY.value,
        [0, 60],
        [1, 0.8],
        Extrapolation.CLAMP
      ),
    };
  });

  return (
    <View className="flex-1 bg-background">
      <Animated.View
        className="bg-primary justify-end px-4 pb-3"
        style={headerStyle}
      >
        <Text
          variant="headlineMedium"
          className="text-on-primary"
          accessibilityRole="header"
        >
          {title}
        </Text>
      </Animated.View>

      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
      >
        {/* Scrollable content */}
      </Animated.ScrollView>
    </View>
  );
}
```

---

## Summary

These accessibility examples cover:

1. **Accessible Forms** -- Labels, hints, error announcements, password toggle state
2. **Accessible Modals** -- Focus management, viewIsModal, backdrop handling, back button
3. **Accessible Tab Bar** -- Tab roles, selected state, badge count in labels
4. **Accessible Lists** -- Section headers as role="header", item count announcement, empty state
5. **Dynamic Type** -- Font scaling with maxFontSizeMultiplier, adaptive layout, reduced motion

**See Also:**
- [complete-examples.md](./complete-examples.md) -- Full screen implementations
- [testing-examples.md](./testing-examples.md) -- Testing accessible components
- [error-handling-examples.md](./error-handling-examples.md) -- Error handling patterns
- [../guides/component-patterns.md](../guides/component-patterns.md) -- Component architecture
