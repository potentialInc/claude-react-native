---
description: Build React Native app using EAS (Expo Application Services) cloud builds for iOS/Android
argument-hint: "[platform: ios|android|all] [profile: development|preview|production]"
---

# EAS Cloud Build Command

Build the React Native mobile app using Expo Application Services (EAS) cloud builds.

## Prerequisites

1. Expo CLI installed: `npm install -g expo-cli`
2. EAS CLI installed: `npm install -g eas-cli`
3. Authenticated with Expo: `eas login`

---

## Phase 0: Auto-Detect Project Structure

Detect the project layout by finding `app.json`:

1. **Find `app.json`** — search in order: `./app.json`, `frontend/app.json`, `mobile/app.json`, `app/app.json`
2. **Set `APP_DIR`** = directory containing `app.json` (e.g., `frontend` or `.`)
3. **Verify `eas.json`** exists in `{APP_DIR}/` — if not, warn that EAS may not be configured

If `app.json` is not found, STOP and ask the user to provide the path to their React Native app directory.

---

## Phase 1: Parse Arguments

Arguments: $ARGUMENTS

**Defaults:**
- Platform: `all` (both iOS and Android)
- Profile: `development`

**Available profiles:**
- `development` - Debug build with development server
- `preview` - Internal testing build
- `production` - Release build for app stores

---

## Phase 2: Pre-Build Checks

### 1. Verify Dependencies

```bash
cd {APP_DIR} && npm install
```

### 2. TypeScript Check

```bash
cd {APP_DIR} && npx tsc --noEmit
```

### 3. Lint Check

```bash
cd {APP_DIR} && npm run lint
```

If any checks fail, report errors and stop.

---

## Phase 3: Build Execution

### Development Build (Local)

```bash
# iOS Simulator
cd {APP_DIR} && npx expo run:ios

# Android Emulator
cd {APP_DIR} && npx expo run:android
```

### EAS Build (Cloud)

```bash
# iOS
cd {APP_DIR} && eas build --platform ios --profile {profile}

# Android
cd {APP_DIR} && eas build --platform android --profile {profile}

# Both platforms
cd {APP_DIR} && eas build --platform all --profile {profile}
```

---

## Phase 4: Build Output

### Development
- iOS: Opens in Simulator
- Android: Opens in Emulator

### Preview/Production
- Provides download links for:
  - iOS: `.ipa` file
  - Android: `.apk` or `.aab` file

---

## Common Issues

### iOS Build Fails
```
Check:
1. Valid Apple Developer account
2. Provisioning profiles configured in eas.json
3. Bundle identifier matches App Store Connect
```

### Android Build Fails
```
Check:
1. Keystore configured in eas.json
2. Package name is unique
3. SDK versions compatible
```

### Metro Bundler Issues
```bash
# Clear cache and restart
cd {APP_DIR} && npx expo start --clear
```

---

## Related Resources
- [Expo Build Documentation](https://docs.expo.dev/build/introduction/)
- [EAS Build Configuration](https://docs.expo.dev/build/eas-json/)

## See Also
- **Local native builds** (Gradle/Xcode): Use `/rn-local-build` for building APK/archive directly on your machine without EAS
- **Local iOS archive**: Use `/rn-local-build ios` for building an Xcode archive locally
