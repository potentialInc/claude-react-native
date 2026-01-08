---
name: frontend-dev-guidelines
description: Frontend development guidelines for React Native/TypeScript applications. Modern patterns including React Navigation, Redux Toolkit, NativeWind (Tailwind for RN), React Native Paper components, React Hook Form + Zod validation, and Axios HTTP service. Use when creating components, screens, features, fetching data, styling, navigation, or working with mobile app code.
---

# React Native Development Guidelines

## Purpose

Comprehensive guide for modern React Native development with TypeScript, React Navigation, Redux Toolkit, NativeWind, and React Native Paper.

## When to Use This Skill

- Creating new components or screens
- Building new features
- Fetching data with Axios + Redux
- Setting up navigation with React Navigation
- Styling components with NativeWind (Tailwind CSS)
- Using React Native Paper components
- Form handling with React Hook Form + Zod
- TypeScript best practices
- Platform-specific code (iOS/Android)

---

## Tech Stack Overview

| Category | Technology |
|----------|------------|
| Framework | React Native 0.73+ |
| Navigation | React Navigation 6.x |
| Build Tool | Metro Bundler |
| State Management | Redux Toolkit |
| Server State | TanStack Query |
| Styling | NativeWind (Tailwind CSS for RN) |
| UI Components | React Native Paper |
| Data Fetching | Axios (HttpService class) |
| Forms | React Hook Form + Zod |
| Icons | React Native Vector Icons |
| Testing | Jest + React Native Testing Library |
| TypeScript | 5.x |

---

## Quick Start

### New Screen Checklist

Creating a screen? Follow this checklist:

1. Create screen component in `src/screens/`
2. Add navigation route
3. Use NativeWind for styling
4. Add TypeScript types
5. Handle loading/error states

### New Component Checklist

Creating a component? Follow this checklist:

1. Create in `src/components/`
2. Use NativeWind classes
3. Add proper TypeScript props
4. Consider platform differences (iOS/Android)

---

## Resources

For detailed guidance, read these resources in order:

1. [Component Patterns](./resources/component-patterns.md) - Screen and component structure
2. [Styling Guide](./resources/styling-guide.md) - NativeWind patterns
3. [Navigation Guide](./resources/navigation-guide.md) - React Navigation setup
4. [Data Fetching](./resources/data-fetching.md) - Axios + TanStack Query
5. [TypeScript Standards](./resources/typescript-standards.md) - Type safety patterns
6. [Common Patterns](./resources/common-patterns.md) - Reusable patterns
