---
name: glassmorphism-component
description: Create glassmorphism (frosted glass) components in React Native Expo that work on both iOS and Android. Use this skill whenever the user mentions glassmorphism, frosted glass, glass effect, blur card, transparent blur, glass UI, liquid glass, or wants to create semi-transparent blurred components in React Native or Expo. Also trigger when the user asks about expo-blur cross-platform setup, BlurView on Android, or wants a modern glass-style card/modal/navbar/tab bar in their React Native app. This skill covers the full spectrum from simple glass cards to animated glassmorphic navigation bars, modals, and complex layered glass UIs.
---

# React Native Expo Glassmorphism Skill

Create production-ready glassmorphism components that work on **both iOS and Android** using Expo.

## What is Glassmorphism?

Glassmorphism is a design style that mimics frosted glass through four key properties:
1. **Background blur** — content behind the element is blurred
2. **Semi-transparency** — the element has a translucent fill
3. **Subtle border** — a light border (often white/translucent) to simulate glass edges
4. **Shadow/elevation** — soft shadow to create depth and floating appearance

## Core Libraries

| Package | Purpose | Install |
|---|---|---|
| `expo-blur` | Native blur on iOS, BlurView on Android (stable from SDK 55+) | `npx expo install expo-blur` |
| `expo-linear-gradient` | Gradient overlays, gradient borders | `npx expo install expo-linear-gradient` |
| `react-native-reanimated` | Animated blur intensity, spring physics | `npx expo install react-native-reanimated` |
| `expo-glass-effect` | Native iOS 26+ liquid glass (**experimental, iOS-only** — verify availability before using in production) | `npx expo install expo-glass-effect` |

## Critical: Android BlurView Setup (SDK 55+)

Starting with Expo SDK 55, `expo-blur` supports real blur on Android — but it requires wrapping blurred content in a `BlurTargetView`. This is the most common mistake developers make: using `BlurView` without a `BlurTargetView` on Android results in a plain semi-transparent overlay instead of actual blur.

### Picking the right `blurMethod`

Define this helper once per file — all examples below use it:

```tsx
import { Platform } from 'react-native';

// renderEffect = native Android 12+ (API 31+) RenderEffect — real, sharp glass blur
// dimezisBlurView = Dimezis library fallback for Android < 12 — softer, less accurate
// undefined = iOS handles blur natively, this prop is ignored
const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;
```

### Full cross-platform example

```tsx
import { useRef } from 'react';
import { View, Text, Platform } from 'react-native';
import { BlurView, BlurTargetView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

export function GlassCardWithAndroidBlur() {
  const targetRef = useRef(null);

  return (
    <View style={{ flex: 1 }}>
      {/* Step 1: Wrap the background content you want to blur */}
      <BlurTargetView ref={targetRef} style={{ ...StyleSheet.absoluteFillObject }}>
        {/* Your background content: images, gradients, etc. */}
      </BlurTargetView>

      {/* Step 2: BlurView references the target */}
      <BlurView
        blurTarget={targetRef}
        intensity={70}
        tint="default"
        blurMethod={blurMethod}
        blurReductionFactor={0.5}
        style={{
          borderRadius: 20,
          overflow: 'hidden',
          backgroundColor: 'rgba(255,255,255,0.08)',
          borderWidth: 1,
          borderColor: 'rgba(255,255,255,0.3)',
          padding: 20,
        }}
      >
        <Text style={{ color: '#fff' }}>Glass Card</Text>
      </BlurView>
    </View>
  );
}
```

**Key Android rules:**
- Always use `BlurTargetView` wrapping the content to blur
- Pass `blurTarget={ref}` to `BlurView`
- Use `blurMethod="renderEffect"` on Android 12+ (API ≥ 31) — this is the native GPU path
- Fall back to `blurMethod="dimezisBlurView"` for Android < 12
- Use `tint="default"` — `tint="light"` / `tint="dark"` add a heavy opacity wash that makes glass look opaque
- Control translucency via `backgroundColor: 'rgba(...)'` on the BlurView, not via `tint`
- Set `blurReductionFactor={0.5}` to balance intensity between iOS and Android (they differ significantly)
- Multiple `BlurView` components can share a single `BlurTargetView`
- `BlurView` must be a sibling (not a child) of `BlurTargetView`
- Add `overflow: 'hidden'` for `borderRadius` to work on blur views

**iOS-only shortcut:** On iOS, `BlurView` works without `BlurTargetView` — it automatically blurs whatever is behind it. But for cross-platform code, always use the `BlurTargetView` pattern.

## Android Glass Feel Tips

If it still looks like an opaque/colored button instead of glass, diagnose with this table:

| Symptom | Root cause | Fix |
|---|---|---|
| Solid colored overlay, no blur | Wrong blurMethod or API < 31 | Use `blurMethod="renderEffect"` on Android 12+ |
| Too white/washed out | `tint="light"` adding opacity wash | Switch to `tint="default"` + explicit `backgroundColor` |
| Blur too faint | iOS/Android intensity mismatch | Raise `intensity` to 60–80 on Android; use `blurReductionFactor={0.5}` |
| Blur not working at all | API level < 31, no real blur support | Add fallback: solid semi-transparent card for Android < 12 |
| Rounded corners don't clip blur | Missing `overflow: 'hidden'` | Add `overflow: 'hidden'` to BlurView style |
| Looks flat even with correct props | Boring background content | Glass needs rich, colorful content in `BlurTargetView` to look good |

## Additional Component Patterns

### GlassButton (LinearGradient border)

```tsx
import { Platform } from 'react-native';
import { LinearGradient } from 'expo-linear-gradient';
import { BlurView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

export function GlassButton({ label, onPress, targetRef }) {
  return (
    <Pressable onPress={onPress}>
      <LinearGradient
        colors={['rgba(255,255,255,0.5)', 'rgba(255,255,255,0.1)']}
        start={{ x: 0, y: 0 }}
        end={{ x: 1, y: 1 }}
        style={{ borderRadius: 14, padding: 1 }}
      >
        <BlurView
          blurTarget={targetRef}
          intensity={60}
          tint="default"
          blurMethod={blurMethod}
          blurReductionFactor={0.5}
          style={{
            borderRadius: 13,
            paddingHorizontal: 24,
            paddingVertical: 12,
            overflow: 'hidden',
            backgroundColor: 'rgba(255,255,255,0.1)',
          }}
        >
          <Text style={{ color: '#fff', fontWeight: '600', fontSize: 16 }}>{label}</Text>
        </BlurView>
      </LinearGradient>
    </Pressable>
  );
}
```

### GlassModal (fullscreen overlay)

```tsx
import { Modal, Platform } from 'react-native';
import { BlurView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

export function GlassModal({ visible, onClose, children, targetRef }) {
  return (
    <Modal visible={visible} transparent animationType="fade" onRequestClose={onClose}>
      {/* tint="default" + dark backgroundColor — avoids the heavy opacity wash of tint="dark" */}
      <BlurView
        blurTarget={targetRef}
        intensity={80}
        tint="default"
        blurMethod={blurMethod}
        blurReductionFactor={0.5}
        style={{
          flex: 1,
          justifyContent: 'center',
          alignItems: 'center',
          overflow: 'hidden',
          backgroundColor: 'rgba(0,0,0,0.35)',
        }}
      >
        <View
          style={{
            width: '85%',
            borderRadius: 24,
            padding: 24,
            backgroundColor: 'rgba(255,255,255,0.12)',
            borderWidth: 1,
            borderColor: 'rgba(255,255,255,0.25)',
          }}
        >
          {children}
        </View>
      </BlurView>
    </Modal>
  );
}
```

### GlassTabBar

```tsx
import { Platform } from 'react-native';
import { BlurView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

// In your Expo Router _layout.tsx — position the tab bar absolutely over a BlurTargetView
export function GlassTabBar({ targetRef, children }) {
  return (
    <BlurView
      blurTarget={targetRef}
      intensity={75}
      tint="default"
      blurMethod={blurMethod}
      blurReductionFactor={0.5}
      style={{
        position: 'absolute',
        bottom: 0,
        left: 0,
        right: 0,
        overflow: 'hidden',
        borderTopWidth: 1,
        borderTopColor: 'rgba(255,255,255,0.3)',
        backgroundColor: 'rgba(255,255,255,0.08)',
        paddingBottom: 20, // safe area — prefer useSafeAreaInsets()
      }}
    >
      {children}
    </BlurView>
  );
}
```

### Animated Blur Intensity (Reanimated)

```tsx
import { Platform } from 'react-native';
import Animated, { useSharedValue, useAnimatedProps, withSpring } from 'react-native-reanimated';
import { BlurView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

const AnimatedBlurView = Animated.createAnimatedComponent(BlurView);

export function AnimatedGlassCard({ targetRef }) {
  const intensity = useSharedValue(20);

  const animatedProps = useAnimatedProps(() => ({ intensity: intensity.value }));

  return (
    <Pressable onPress={() => { intensity.value = withSpring(80); }}>
      <AnimatedBlurView
        animatedProps={animatedProps}
        blurTarget={targetRef}
        tint="default"
        blurMethod={blurMethod}
        blurReductionFactor={0.5}
        style={{
          width: 300,
          height: 160,
          borderRadius: 20,
          overflow: 'hidden',
          backgroundColor: 'rgba(255,255,255,0.08)',
        }}
      />
    </Pressable>
  );
}
```

### Dark Mode Variant

```tsx
import { Platform, useColorScheme } from 'react-native';
import { BlurView } from 'expo-blur';

const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

export function AdaptiveGlassCard({ targetRef, children }) {
  const scheme = useColorScheme();
  const isDark = scheme === 'dark';

  return (
    <BlurView
      blurTarget={targetRef}
      intensity={isDark ? 75 : 60}
      tint="default"
      blurMethod={blurMethod}
      blurReductionFactor={0.5}
      style={{
        borderRadius: 20,
        overflow: 'hidden',
        borderWidth: 1,
        borderColor: isDark ? 'rgba(255,255,255,0.15)' : 'rgba(255,255,255,0.4)',
        backgroundColor: isDark ? 'rgba(0,0,0,0.25)' : 'rgba(255,255,255,0.08)',
      }}
    >
      {children}
    </BlurView>
  );
}
```

## Quick-Start: Minimal Glass Card

This is the simplest cross-platform glass card. Copy this as a starting point:

```tsx
import { useRef } from 'react';
// Note: StyleSheet used intentionally — BlurView/BlurTargetView don't support NativeWind className
import { View, Text, Platform, StyleSheet, ImageBackground } from 'react-native';
import { BlurView, BlurTargetView } from 'expo-blur';

// renderEffect = real native blur on Android 12+ | dimezisBlurView = fallback for older Android
const blurMethod = Platform.OS === 'android'
  ? (Platform.Version >= 31 ? 'renderEffect' : 'dimezisBlurView')
  : undefined;

export default function GlassDemo() {
  const targetRef = useRef(null);

  return (
    <View style={{ flex: 1 }}>
      <BlurTargetView ref={targetRef} style={StyleSheet.absoluteFill}>
        <ImageBackground
          source={{ uri: 'https://images.unsplash.com/photo-1506905925346-21bda4d32df4?w=800' }}
          style={StyleSheet.absoluteFill}
        />
      </BlurTargetView>

      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <BlurView
          blurTarget={targetRef}
          intensity={65}
          tint="default"
          blurMethod={blurMethod}
          blurReductionFactor={0.5}
          style={{
            width: 300,
            padding: 24,
            borderRadius: 20,
            overflow: 'hidden',
            borderWidth: 1,
            borderColor: 'rgba(255,255,255,0.3)',
            backgroundColor: 'rgba(255,255,255,0.08)',
          }}
        >
          <Text style={{ fontSize: 20, fontWeight: '700', color: '#fff' }}>
            Glass Card
          </Text>
          <Text style={{ fontSize: 14, color: 'rgba(255,255,255,0.8)', marginTop: 8 }}>
            This works on both iOS and Android
          </Text>
        </BlurView>
      </View>
    </View>
  );
}
```

## Design Token Presets

Use `tint="default"` for all styles — control the glass tint via `backgroundColor` instead of `tint`. iOS intensity and Android intensity need different values; use `blurReductionFactor={0.5}` to help bridge the gap.

| Style | iOS intensity | Android intensity | Tint | Border | backgroundColor |
|---|---|---|---|---|---|
| Light glass | 50–65 | 65–80 | `"default"` | `rgba(255,255,255,0.3)` | `rgba(255,255,255,0.08)` |
| Dark glass | 60–80 | 75–90 | `"default"` | `rgba(255,255,255,0.15)` | `rgba(0,0,0,0.25)` |
| Frosted | 80–100 | 90–100 | `"default"` | `rgba(255,255,255,0.4)` | `rgba(255,255,255,0.15)` |
| Subtle | 25–40 | 40–55 | `"default"` | `rgba(255,255,255,0.15)` | `rgba(255,255,255,0.04)` |
| Colored | 50–65 | 65–80 | `"default"` | `rgba(COLOR,0.3)` | `rgba(COLOR,0.08)` |

## Checklist Before Shipping

- [ ] Test on both iOS simulator and Android emulator/device
- [ ] Using `blurMethod="renderEffect"` on Android 12+ (API ≥ 31) — not `"dimezisBlurView"` as primary
- [ ] Using `tint="default"` — avoid `tint="light"/"dark"` (they add a heavy opacity wash)
- [ ] `backgroundColor: 'rgba(...)'` set on BlurView for translucency control (not relying on tint)
- [ ] `blurReductionFactor={0.5}` set to calibrate iOS vs Android intensity
- [ ] Verify `BlurTargetView` wraps background content on Android
- [ ] `overflow: 'hidden'` on all BlurView components with borderRadius
- [ ] Avoid nesting BlurViews (performance killer)
- [ ] Check that BlurView renders *after* dynamic content (FlatList etc.)
- [ ] Provide fallback styling for Android SDK < 31 if targeting old devices