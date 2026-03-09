---
description: Build React Native app locally (Android APK via Gradle / iOS Archive via Xcode) with optional version bump
argument-hint: "[android|ios|both] [--bump patch|minor|major] [--version X.Y.Z] [--no-version]"
---

# App Build — Unified Build & Version Workflow

You are a build assistant for a React Native / Expo mobile app. Your task is to optionally bump the version and then build the app for Android, iOS, or both platforms.

---

## Phase 0: Auto-Detect Project Structure

Detect the project layout by finding `app.json`:

1. **Find `app.json`** — search in order: `./app.json`, `frontend/app.json`, `mobile/app.json`, `app/app.json`
2. **Read `app.json`** to extract the app name from `expo.name` or `name` field → store as `APP_NAME`
3. **Set `APP_DIR`** = directory containing `app.json` (e.g., `frontend` or `.`)
4. **Detect iOS paths** using Glob:
   - `{APP_DIR}/ios/*/Info.plist` → store as `PLIST_PATH`
   - `{APP_DIR}/ios/*.xcodeproj/project.pbxproj` → store as `PBXPROJ_PATH`
   - `{APP_DIR}/ios/*.xcworkspace` → store as `WORKSPACE_PATH`
   - Set `SCHEME` = `APP_NAME`
5. **Detect Android path**: `{APP_DIR}/android/app/build.gradle` → store as `GRADLE_PATH`
6. **Read current version** from `{APP_DIR}/package.json` → store as `CURRENT_VERSION` (e.g., `1.0.24`)
7. **Calculate current build number**: Parse `CURRENT_VERSION` as `major.minor.patch` → `major × 10000 + minor × 1000 + patch` → store as `CURRENT_BUILD_NUMBER`

If `app.json` is not found, STOP and ask the user to provide the path to their React Native app directory.

---

## Phase 1: Ask Platform

**If `$ARGUMENTS` contains `android`, `ios`, or `both`**, use that as the platform selection and skip this question.

**Otherwise**, use **AskUserQuestion** to ask:
- Question: "Which platform(s) do you want to build?"
- Options:
  1. `Android APK` — description: "Build release APK via Gradle assembleRelease"
  2. `iOS Archive` — description: "Build Xcode archive for App Store distribution"
  3. `Both` — description: "Build Android APK + iOS Archive in parallel"

Store the selection as `BUILD_PLATFORMS` (one of: `android`, `ios`, `both`).

---

## Phase 2: Ask Version Update

**If `$ARGUMENTS` contains `--no-version`**, skip version update entirely.
**If `$ARGUMENTS` contains `--bump patch|minor|major`**, use that bump type and skip this question.
**If `$ARGUMENTS` contains `--version X.Y.Z`**, use that exact version and skip this question.

**Otherwise**, calculate what each bump would produce and use **AskUserQuestion** to ask:
- Question: "Current version: CURRENT_VERSION (build: CURRENT_BUILD_NUMBER). Update version before building?"
- Options (compute the actual values from current version and show them inline):
  1. `Patch` — description: "CURRENT_VERSION → X.Y.(Z+1) (build: CURRENT_BUILD → NEW_BUILD)"
  2. `Minor` — description: "CURRENT_VERSION → X.(Y+1).0 (build: CURRENT_BUILD → NEW_BUILD)"
  3. `Major` — description: "CURRENT_VERSION → (X+1).0.0 (build: CURRENT_BUILD → NEW_BUILD)"
  4. `Manual` — description: "Enter a specific version number"
  5. `No version update` — description: "Keep CURRENT_VERSION (build: CURRENT_BUILD), just build"

**If user chooses "Manual"**, use **AskUserQuestion** to ask:
- Question: "Enter the new version number (format: X.Y.Z):"
- Options (the user MUST use "Other" to type their version — provide fallbacks):
  1. `Keep current` — description: "Stay on CURRENT_VERSION"
  2. `Next patch` — description: "Use next patch version (show computed value)"

Validate that the version matches `X.Y.Z` format. If invalid, ask again.

Store:
- `NEW_VERSION` = the new semantic version (or `null` if no update)
- `NEW_BUILD_NUMBER` = `major × 10000 + minor × 1000 + patch` (or `null` if no update)

---

## Phase 3: Environment Check

Read `{APP_DIR}/.env` and find the value of `EXPO_PUBLIC_API_URL`.

**If the URL is NOT the production API** (compare against the project's known production URL):
- Print a warning:
  ```
  ⚠ WARNING: API URL is not set to production!
  Current: <current_value>
  Expected production URL: <production_url>
  ```
- Use **AskUserQuestion** to ask:
  - Question: "API URL is not set to production. Continue?"
  - Options:
    1. `Continue` — description: "Build with current URL (development/staging)"
    2. `Stop` — description: "I'll update .env manually first"

  If user chooses "Stop", STOP execution.

**If the URL IS the production API**, print: `Environment: Production (✓)` and proceed.

**If `.env` file is missing**, warn the user and proceed.

---

## Phase 4: Pre-Build Checks

### 1. TypeScript Check

```bash
cd {APP_DIR} && npx tsc --noEmit
```

### 2. Lint Check

```bash
cd {APP_DIR} && npm run lint
```

If either check fails, report the errors and ask the user whether to continue or stop:
- Use **AskUserQuestion**:
  - Question: "Pre-build checks found issues (see errors above). Continue with build?"
  - Options:
    1. `Continue` — description: "Build anyway (not recommended for production)"
    2. `Stop` — description: "Fix issues first"

If user chooses "Stop", STOP execution.

---

## Phase 5: Version Update

**Skip this phase entirely if user chose "No version update" or `--no-version`.**

Read each file before editing. Apply edits in parallel where possible.

### 5.1 `{APP_DIR}/package.json`
```
Edit: "version": "CURRENT_VERSION" → "version": "NEW_VERSION"
```

### 5.2 `{APP_DIR}/app.json`
```
Edit: "version": "CURRENT_VERSION" → "version": "NEW_VERSION"
```

### 5.3 `{PLIST_PATH}`
Two edits:
- `CFBundleShortVersionString`: `<string>CURRENT_VERSION</string>` → `<string>NEW_VERSION</string>`
- `CFBundleVersion`: `<string>CURRENT_BUILD_NUMBER</string>` → `<string>NEW_BUILD_NUMBER</string>`

**Note:** `CFBundleVersion` may be `1` if reset by `expo prebuild`. Check actual value before editing.

### 5.4 `{GRADLE_PATH}`
Two edits:
- `versionCode CURRENT_BUILD_NUMBER` → `versionCode NEW_BUILD_NUMBER`
- `versionName "CURRENT_VERSION"` → `versionName "NEW_VERSION"`

### 5.5 `{PBXPROJ_PATH}`
Two edits (use `replace_all: true` — there are 2 occurrences each for Debug/Release configs):
- `MARKETING_VERSION = CURRENT_VERSION;` → `MARKETING_VERSION = NEW_VERSION;`
- `CURRENT_PROJECT_VERSION = CURRENT_BUILD_NUMBER;` → `CURRENT_PROJECT_VERSION = NEW_BUILD_NUMBER;`

**Note:** Values may be `1.0` / `1` if reset by `expo prebuild`. Check actual values before editing.

### 5.6 Verify & Report

After all edits, verify by grepping for the new version in each file. Print:
```
✓ Version Updated: CURRENT_VERSION → NEW_VERSION
✓ Build Number: CURRENT_BUILD_NUMBER → NEW_BUILD_NUMBER
```

---

## Phase 6: Build Execution

### If `BUILD_PLATFORMS` is `both`:

Launch **two parallel agents** (Agent tool, single message with two tool calls):

- **Agent 1 — Android Build**: Execute Phase 6A instructions below. Provide all detected paths and version info in the agent prompt.
- **Agent 2 — iOS Build**: Execute Phase 6B instructions below. Provide all detected paths and version info in the agent prompt.

### If `BUILD_PLATFORMS` is `android`:

Execute Phase 6A directly (no agent needed).

### If `BUILD_PLATFORMS` is `ios`:

Execute Phase 6B directly (no agent needed).

---

### Phase 6A: Android APK Build

1. **Clean previous build:**
   ```bash
   cd {APP_DIR}/android && ./gradlew clean 2>&1 | tail -3
   ```
   Use `timeout: 120000` (2 min).

2. **Build release APK:**
   ```bash
   cd {APP_DIR}/android && ./gradlew assembleRelease 2>&1 | tail -10
   ```
   Use `timeout: 600000` (10 min).

3. **Locate and verify output APK:**
   ```bash
   ls -lh {APP_DIR}/android/app/build/outputs/apk/release/*.apk
   ```

4. **Store result:**
   - `ANDROID_APK_PATH` = path to the generated `.apk` file
   - `ANDROID_APK_SIZE` = file size
   - `ANDROID_BUILD_SUCCESS` = true/false

---

### Phase 6B: iOS Archive Build

1. **Verify iOS version consistency** (only if version was NOT updated in Phase 5):

   Read `{PLIST_PATH}`, `{PBXPROJ_PATH}`, and `{APP_DIR}/app.json` to check all version values match.

   **If values are out of sync** (common after `expo prebuild --clean`), use **AskUserQuestion**:
   - Question: "iOS version files are out of sync. Fix to match app.json?"
   - Show current values from each file
   - Options:
     1. `Fix` — description: "Sync all files to app.json version (APP_JSON_VERSION / BUILD_NUMBER) — Recommended"
     2. `Cancel` — description: "Cancel iOS build"

   If user chooses to fix, apply edits using `replace_all: true` for project.pbxproj.

2. **Build Xcode archive:**
   ```bash
   xcodebuild \
     -workspace {WORKSPACE_PATH} \
     -scheme {SCHEME} \
     -configuration Release \
     -archivePath /tmp/{APP_NAME}.xcarchive \
     -allowProvisioningUpdates \
     archive 2>&1 | tail -5
   ```
   Use `timeout: 600000` (10 min).

3. **Verify archive version:**
   ```bash
   /usr/libexec/PlistBuddy -c "Print :ApplicationProperties:CFBundleShortVersionString" /tmp/{APP_NAME}.xcarchive/Info.plist
   /usr/libexec/PlistBuddy -c "Print :ApplicationProperties:CFBundleVersion" /tmp/{APP_NAME}.xcarchive/Info.plist
   ```

4. **Store result:**
   - `IOS_ARCHIVE_PATH` = `/tmp/{APP_NAME}.xcarchive`
   - `IOS_BUILD_SUCCESS` = true/false

---

## Phase 7: Build Report

Print a final summary. Show only the platforms that were built.

```
Build Complete — {APP_NAME}

Version: X.Y.Z (build: XXXXX)
Environment: production / development

Android APK:
  ✓ Path: {ANDROID_APK_PATH}
    Size: {ANDROID_APK_SIZE}
  — or —
  ✗ Build failed (see errors above)

iOS Archive:
  ✓ Path: /tmp/{APP_NAME}.xcarchive
  — or —
  ✗ Build failed (see errors above)

Next Steps:
  • Android: Install APK on device/emulator
  • iOS: Open Xcode Organizer to distribute to App Store Connect
    Or: xcodebuild -exportArchive -archivePath /tmp/{APP_NAME}.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath /tmp/{APP_NAME}Export
```

---

## Error Handling

| Issue | Action |
|-------|--------|
| `app.json` not found | STOP, ask user for app directory path |
| Invalid version format (not `X.Y.Z`) | STOP, ask user for valid version |
| Native project files missing (iOS/Android) | STOP, warn that `npx expo prebuild` may be needed |
| `.env` file missing | Warn, proceed |
| `EXPO_PUBLIC_API_URL` not production | Ask user to confirm or stop |
| Version mismatch across files | Warn, show values, offer to fix |
| `expo prebuild` reset values (`1.0` / `1`) | Detect and handle — use actual values in old_string |
| Edit fails (old_string not found) | Read file again, find actual value, retry with correct old_string |
| Android build failed | Show last 30 lines, suggest: check ANDROID_HOME, signing config, or try `-Dorg.gradle.jvmargs="-Xmx4g"` |
| iOS archive failed | Show last 30 lines, suggest: check signing team (org not personal), run `pod install`, or clean build |
| Build timeout | Inform user, suggest running manually with longer timeout |

---

## See Also
- **EAS cloud builds**: Use `/rn-eas-build` for building via Expo Application Services (cloud-based, no local native toolchain required)
- **iOS archive only**: Use `/rn-xcode-archive` for a focused iOS-only Xcode archive workflow (subset of Phase 6B above)
