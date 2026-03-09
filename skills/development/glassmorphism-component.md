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
| `expo-glass-effect` | Native iOS 26+ liquid glass (optional, iOS-only) | `npx expo install expo-glass-effect` |

## Critical: Android BlurView Setup (SDK 55+)

Starting with Expo SDK 55, `expo-blur` supports real blur on Android — but it requires wrapping blurred content in a `BlurTargetView`. This is the most common mistake developers make: using `BlurView` without a `BlurTargetView` on Android results in a plain semi-transparent overlay instead of actual blur.

```tsx
import { useRef } from 'react';
import { View, Text, StyleSheet, Platform } from 'react-native';
import { BlurView, BlurTargetView } from 'expo-blur';

export function GlassCardWithAndroidBlur() {
  const targetRef = useRef(null);

  return (
    <View style={styles.container}>
      {/* Step 1: Wrap the background content you want to blur */}
      <BlurTargetView ref={targetRef} style={StyleSheet.absoluteFill}>
        {/* Your background content: images, gradients, etc. */}
      </BlurTargetView>

      {/* Step 2: BlurView references the target */}
      <BlurView
        blurTarget={targetRef}
        intensity={60}
        tint="light"
        blurMethod="dimezisBlurView"
        style={styles.glassCard}
      >
        <Text style={styles.text}>Glass Card</Text>
      </BlurView>
    </View>
  );
}
```

**Key Android rules:**
- Always use `BlurTargetView` wrapping the content to blur
- Pass `blurTarget={ref}` to `BlurView`
- Set `blurMethod="dimezisBlurView"` for real blur on Android
- Multiple `BlurView` components can share a single `BlurTargetView`
- `BlurView` must be a sibling (not a child) of `BlurTargetView`
- Add `overflow: 'hidden'` for `borderRadius` to work on blur views

**iOS-only shortcut:** On iOS, `BlurView` works without `BlurTargetView` — it automatically blurs whatever is behind it. But for cross-platform code, always use the `BlurTargetView` pattern.

## Read the Reference File

For complete component patterns, read the reference file:

- **`references/components.md`** — Full implementation patterns including:
  - Basic GlassCard component (cross-platform)
  - GlassButton with gradient border
  - GlassModal overlay
  - GlassNavBar / GlassTabBar
  - Animated glassmorphism with Reanimated
  - Dark mode glassmorphism
  - Performance optimization tips
  - Common pitfalls and fixes

Read it with: `view /path/to/rn-glassmorphism/references/components.md`

## Quick-Start: Minimal Glass Card

This is the simplest cross-platform glass card. Copy this as a starting point:

```tsx
import { useRef } from 'react';
import { View, Text, StyleSheet, ImageBackground } from 'react-native';
import { BlurView, BlurTargetView } from 'expo-blur';

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
          intensity={40}
          tint="light"
          blurMethod="dimezisBlurView"
          style={{
            width: 300,
            padding: 24,
            borderRadius: 20,
            overflow: 'hidden',
            borderWidth: 1,
            borderColor: 'rgba(255,255,255,0.3)',
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

Use these as starting points for different glass styles:

| Style | Intensity | Tint | Border | Background Overlay |
|---|---|---|---|---|
| Light glass | 40-60 | `"light"` | `rgba(255,255,255,0.3)` | `rgba(255,255,255,0.1)` |
| Dark glass | 50-80 | `"dark"` | `rgba(255,255,255,0.15)` | `rgba(0,0,0,0.2)` |
| Frosted | 80-100 | `"default"` | `rgba(255,255,255,0.4)` | `rgba(255,255,255,0.25)` |
| Subtle | 20-30 | `"light"` | `rgba(255,255,255,0.15)` | `rgba(255,255,255,0.05)` |
| Colored | 40-60 | `"default"` | `rgba(COLOR,0.3)` | `rgba(COLOR,0.1)` |

## Checklist Before Shipping

- [ ] Test on both iOS simulator and Android emulator/device
- [ ] Verify `BlurTargetView` wraps background content on Android
- [ ] `overflow: 'hidden'` on all BlurView components with borderRadius
- [ ] Avoid nesting BlurViews (performance killer)
- [ ] Check that BlurView renders *after* dynamic content (FlatList etc.)
- [ ] Provide fallback styling for Android SDK < 31 if targeting old devices
- [ ] Test with `blurReductionFactor` prop to match iOS/Android intensity