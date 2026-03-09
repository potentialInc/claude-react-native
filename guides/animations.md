# Animations Guide

Patterns for performant animations in React Native using Reanimated 3 and Gesture Handler.

## Table of Contents

- [Why Reanimated over Animated API](#why-reanimated-over-animated-api)
- [Core Concepts](#core-concepts)
- [Layout Animations](#layout-animations)
- [Gesture-Driven Animations](#gesture-driven-animations)
- [Shared Element Transitions](#shared-element-transitions)
- [Skeleton Loading](#skeleton-loading)
- [Lottie Animations](#lottie-animations)
- [Performance Considerations](#performance-considerations)

---

## Why Reanimated over Animated API

The built-in `Animated` API runs animations on the JS thread, which means any JS work (data fetching, state updates, navigation) can cause frame drops. Reanimated runs animations as **worklets on the UI thread**, keeping them at 60/120fps regardless of JS load.

| Feature | Animated API | Reanimated 3 |
|---------|-------------|--------------|
| Execution thread | JS thread | UI thread (worklets) |
| Gesture integration | Limited | Native via Gesture Handler |
| Layout animations | Manual | Built-in (entering/exiting) |
| Shared element transitions | Not supported | Supported |
| Performance under JS load | Drops frames | Smooth |

**When Animated API is still fine:**
- Simple one-off opacity fades on mount
- Basic translateY for keyboard avoidance
- Apps with no gestures or complex transitions

For everything else, use Reanimated.

---

## Core Concepts

### Shared Values

Shared values are the building blocks — they live on both JS and UI threads:

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withDecay,
  Easing,
} from 'react-native-reanimated';

function FadeInCard() {
  const opacity = useSharedValue(0);
  const translateY = useSharedValue(20);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [{ translateY: translateY.value }],
  }));

  useEffect(() => {
    opacity.value = withTiming(1, { duration: 300 });
    translateY.value = withTiming(0, { duration: 300 });
  }, []);

  return (
    <Animated.View
      style={animatedStyle}
      className="rounded-xl bg-white p-4 shadow-md"
    >
      <Text className="text-lg font-semibold">Card Content</Text>
    </Animated.View>
  );
}
```

### Animation Functions

```typescript
// Timing — linear or eased animation to a target value
opacity.value = withTiming(1, {
  duration: 300,
  easing: Easing.bezier(0.25, 0.1, 0.25, 1),
});

// Spring — physics-based animation
scale.value = withSpring(1, {
  damping: 15,
  stiffness: 150,
  mass: 1,
});

// Decay — velocity-based (fling gestures)
translateX.value = withDecay({
  velocity: velocityX,
  clamp: [-200, 200], // optional bounds
});
```

### Interpolation

```typescript
import { interpolate, Extrapolation } from 'react-native-reanimated';

function ParallaxHeader({ scrollY }: { scrollY: Animated.SharedValue<number> }) {
  const animatedStyle = useAnimatedStyle(() => {
    const headerHeight = interpolate(
      scrollY.value,
      [0, 200],
      [300, 80],
      Extrapolation.CLAMP
    );

    const headerOpacity = interpolate(
      scrollY.value,
      [0, 150, 200],
      [1, 0.5, 0],
      Extrapolation.CLAMP
    );

    return {
      height: headerHeight,
      opacity: headerOpacity,
    };
  });

  return (
    <Animated.View style={animatedStyle} className="bg-blue-600">
      <Text className="text-2xl font-bold text-white">Header</Text>
    </Animated.View>
  );
}
```

---

## Layout Animations

Layout animations automatically animate components when they enter, exit, or change layout position.

### Entering Animations

```tsx
import Animated, {
  FadeIn,
  FadeInDown,
  SlideInRight,
  ZoomIn,
} from 'react-native-reanimated';

function NotificationList({ notifications }: { notifications: Notification[] }) {
  return (
    <View className="gap-3 p-4">
      {notifications.map((item, index) => (
        <Animated.View
          key={item.id}
          entering={FadeInDown.delay(index * 100).springify()}
          className="rounded-lg bg-white p-4 shadow-sm"
        >
          <Text className="font-medium text-gray-900">{item.title}</Text>
          <Text className="text-sm text-gray-600">{item.body}</Text>
        </Animated.View>
      ))}
    </View>
  );
}
```

### Exiting Animations

```tsx
import Animated, { FadeOut, SlideOutLeft } from 'react-native-reanimated';
import { useState } from 'react';

function DismissableItem({ item, onDismiss }: DismissableItemProps) {
  const [isVisible, setIsVisible] = useState(true);

  const handleDismiss = () => {
    setIsVisible(false);
    // Call onDismiss after animation completes
    setTimeout(() => onDismiss(item.id), 300);
  };

  if (!isVisible) return null;

  return (
    <Animated.View
      exiting={SlideOutLeft.duration(300)}
      className="rounded-lg bg-white p-4"
    >
      <Text>{item.name}</Text>
      <Pressable
        onPress={handleDismiss}
        accessibilityRole="button"
        accessibilityLabel={`Dismiss ${item.name}`}
      >
        <Text className="text-red-500">Dismiss</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### Layout Transitions for List Reordering

```tsx
import Animated, { LinearTransition } from 'react-native-reanimated';

function SortableList({ items }: { items: ListItem[] }) {
  return (
    <View className="gap-2 p-4">
      {items.map((item) => (
        <Animated.View
          key={item.id}
          layout={LinearTransition.springify()}
          className="rounded-lg bg-white p-4 shadow-sm"
        >
          <Text className="font-medium">{item.name}</Text>
        </Animated.View>
      ))}
    </View>
  );
}
```

### Custom Entering Animation

```typescript
import { withTiming, withDelay } from 'react-native-reanimated';
import type { EntryAnimationsValues } from 'react-native-reanimated';

function customEntering(values: EntryAnimationsValues) {
  'worklet';
  const animations = {
    opacity: withDelay(200, withTiming(1, { duration: 300 })),
    transform: [
      { translateY: withDelay(200, withTiming(0, { duration: 400 })) },
      { scale: withDelay(200, withTiming(1, { duration: 300 })) },
    ],
  };

  const initialValues = {
    opacity: 0,
    transform: [{ translateY: 50 }, { scale: 0.9 }],
  };

  return { animations, initialValues };
}

// Usage
<Animated.View entering={customEntering}>
  <Text>Custom animated content</Text>
</Animated.View>
```

---

## Gesture-Driven Animations

### Gesture Handler v2 with Reanimated

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';

function SwipeableCard({ onSwipeOff }: { onSwipeOff: () => void }) {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const rotation = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY;
      rotation.value = event.translationX / 20; // tilt as you drag
    })
    .onEnd((event) => {
      const shouldDismiss = Math.abs(event.translationX) > 150;

      if (shouldDismiss) {
        const direction = event.translationX > 0 ? 1 : -1;
        translateX.value = withSpring(direction * 500);
        runOnJS(onSwipeOff)();
      } else {
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        rotation.value = withSpring(0);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { rotateZ: `${rotation.value}deg` },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View
        style={animatedStyle}
        className="h-96 w-72 items-center justify-center rounded-2xl bg-white shadow-xl"
      >
        <Text className="text-xl font-bold">Swipe me</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### Pinch to Zoom

```tsx
function ZoomableImage({ uri }: { uri: string }) {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);

  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = savedScale.value * event.scale;
    })
    .onEnd(() => {
      if (scale.value < 1) {
        scale.value = withSpring(1);
        savedScale.value = 1;
      } else {
        savedScale.value = scale.value;
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={pinchGesture}>
      <Animated.Image
        source={{ uri }}
        style={animatedStyle}
        className="h-80 w-full"
        resizeMode="contain"
        accessibilityRole="image"
      />
    </GestureDetector>
  );
}
```

### Combining Multiple Gestures

```tsx
function DraggableZoomable() {
  const scale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });

  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = event.scale;
    })
    .onEnd(() => {
      scale.value = withSpring(1);
    });

  // Simultaneous: both gestures can be active at the same time
  const composedGesture = Gesture.Simultaneous(panGesture, pinchGesture);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <GestureDetector gesture={composedGesture}>
      <Animated.View style={animatedStyle} className="h-64 w-64 rounded-xl bg-blue-500" />
    </GestureDetector>
  );
}
```

---

## Shared Element Transitions

Shared element transitions animate an element smoothly between two screens during navigation.

### Basic Setup

```tsx
// screens/PhotoGrid.tsx
import Animated from 'react-native-reanimated';

function PhotoGrid({ photos }: { photos: Photo[] }) {
  const router = useRouter();

  return (
    <FlatList
      data={photos}
      numColumns={3}
      renderItem={({ item }) => (
        <Pressable
          onPress={() => router.push(`/photo/${item.id}`)}
          accessibilityRole="button"
          accessibilityLabel={`View ${item.title}`}
        >
          <Animated.Image
            source={{ uri: item.thumbnail }}
            sharedTransitionTag={`photo-${item.id}`}
            className="m-0.5 aspect-square flex-1"
          />
        </Pressable>
      )}
    />
  );
}

// screens/PhotoDetail.tsx
function PhotoDetail() {
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View className="flex-1 bg-black">
      <Animated.Image
        source={{ uri: getPhotoUrl(id) }}
        sharedTransitionTag={`photo-${id}`}
        className="h-full w-full"
        resizeMode="contain"
      />
    </View>
  );
}
```

---

## Skeleton Loading

### Shimmer Effect with Reanimated

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  interpolate,
} from 'react-native-reanimated';
import { LinearGradient } from 'expo-linear-gradient';

function SkeletonLoader({ width, height }: { width: number; height: number }) {
  const shimmerPosition = useSharedValue(-1);

  useEffect(() => {
    shimmerPosition.value = withRepeat(
      withTiming(1, { duration: 1200 }),
      -1, // infinite
      false
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      {
        translateX: interpolate(
          shimmerPosition.value,
          [-1, 1],
          [-width, width]
        ),
      },
    ],
  }));

  return (
    <View
      style={{ width, height }}
      className="overflow-hidden rounded-lg bg-gray-200"
    >
      <Animated.View style={[{ width, height }, animatedStyle]}>
        <LinearGradient
          colors={['transparent', 'rgba(255,255,255,0.4)', 'transparent']}
          start={{ x: 0, y: 0 }}
          end={{ x: 1, y: 0 }}
          style={{ width, height }}
        />
      </Animated.View>
    </View>
  );
}
```

### Skeleton Card Placeholder

```tsx
function ProductCardSkeleton() {
  return (
    <View className="rounded-xl bg-white p-4 shadow-sm">
      <SkeletonLoader width={280} height={200} />
      <View className="mt-3 gap-2">
        <SkeletonLoader width={200} height={20} />
        <SkeletonLoader width={100} height={16} />
        <SkeletonLoader width={80} height={24} />
      </View>
    </View>
  );
}

function ProductListSkeleton({ count }: { count: number }) {
  return (
    <View className="gap-4 p-4">
      {Array.from({ length: count }).map((_, index) => (
        <ProductCardSkeleton key={index} />
      ))}
    </View>
  );
}
```

> **MotiView alternative:** The [Moti](https://moti.fyi) library provides `MotiView` with a `Skeleton` component that handles shimmer effects out of the box, built on top of Reanimated.

---

## Lottie Animations

### Setup and Basic Usage

```bash
npx expo install lottie-react-native
```

```tsx
import LottieView from 'lottie-react-native';
import { useRef } from 'react';

function SuccessAnimation() {
  const animationRef = useRef<LottieView>(null);

  return (
    <View className="items-center justify-center">
      <LottieView
        ref={animationRef}
        source={require('@/assets/animations/success.json')}
        autoPlay
        loop={false}
        style={{ width: 200, height: 200 }}
        onAnimationFinish={() => {
          // Navigate or update state after animation
        }}
      />
    </View>
  );
}
```

### Controlled Playback

```tsx
function LoadingAnimation({ isLoading }: { isLoading: boolean }) {
  const animationRef = useRef<LottieView>(null);

  useEffect(() => {
    if (isLoading) {
      animationRef.current?.play();
    } else {
      animationRef.current?.pause();
    }
  }, [isLoading]);

  return (
    <LottieView
      ref={animationRef}
      source={require('@/assets/animations/loading-spinner.json')}
      loop
      style={{ width: 80, height: 80 }}
    />
  );
}
```

### When to Use Lottie vs Reanimated

| Use Lottie When | Use Reanimated When |
|-----------------|---------------------|
| Complex illustrations (100+ elements) | Interactive/gesture-driven animations |
| Animations designed in After Effects | Value depends on user input |
| Brand animations, onboarding | Layout transitions |
| Loading spinners with complex graphics | Scroll-linked animations |
| One-shot celebratory effects | Continuously updating based on state |

---

## Performance Considerations

### Run on UI Thread

```typescript
// GOOD: useAnimatedStyle runs as a worklet on the UI thread
const animatedStyle = useAnimatedStyle(() => {
  'worklet';
  return {
    opacity: opacity.value,
    transform: [{ scale: scale.value }],
  };
});

// BAD: Animated.Value with JS callback runs on JS thread — drops frames under load
const opacity = new Animated.Value(0);
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true, // helps, but still JS-initiated
}).start();
```

### Avoid Layout Thrashing

```typescript
// GOOD: Animate transform/opacity (GPU-composited, no layout recalculation)
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateY: translateY.value }],
  opacity: opacity.value,
}));

// BAD: Animating width/height causes layout recalculation every frame
const animatedStyle = useAnimatedStyle(() => ({
  width: width.value,
  height: height.value,
  marginTop: margin.value,
}));
```

### Reduced Motion

Always respect the user's reduced motion preference:

```typescript
import { useReducedMotion } from 'react-native-reanimated';

function AnimatedButton({ children, onPress }: AnimatedButtonProps) {
  const reduceMotion = useReducedMotion();
  const scale = useSharedValue(1);

  const tapGesture = Gesture.Tap()
    .onBegin(() => {
      if (!reduceMotion) {
        scale.value = withSpring(0.95);
      }
    })
    .onFinalize(() => {
      scale.value = withSpring(1);
      runOnJS(onPress)();
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={tapGesture}>
      <Animated.View style={animatedStyle}>
        <Pressable accessibilityRole="button">{children}</Pressable>
      </Animated.View>
    </GestureDetector>
  );
}
```

### FlatList Item Animations

```tsx
// GOOD: Animate items as they appear in the viewport
function AnimatedFlatList<T>({ data, renderItem }: AnimatedFlatListProps<T>) {
  return (
    <FlatList
      data={data}
      renderItem={({ item, index }) => (
        <Animated.View entering={FadeInDown.delay(index * 50).duration(300)}>
          {renderItem({ item, index })}
        </Animated.View>
      )}
      // Limit initial render to avoid animating too many items at once
      initialNumToRender={10}
      maxToRenderPerBatch={5}
      windowSize={5}
    />
  );
}
```

---

## Related Resources

- [Performance Guide](./performance.md) — general performance optimization
- [Styling Guide](./styling-guide.md) — NativeWind and design tokens
- [Design System Setup Skill](../skills/ui-design/design-system-setup.md) — building a component library
