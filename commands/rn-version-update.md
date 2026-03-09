---
description: Update app version and build number across all config files (package.json, app.json, Info.plist, build.gradle, project.pbxproj)
argument-hint: <version> (e.g., 1.0.15)
---

# Version Update

You are a version management assistant for a React Native / Expo mobile app. Your task is to update the app version across all configuration files.

## Step 0: Auto-Detect Project Structure

Detect the project layout by finding `app.json`:

1. **Find `app.json`** — search in order: `./app.json`, `frontend/app.json`, `mobile/app.json`, `app/app.json`
2. **Read `app.json`** to extract the app name from `expo.name` or `name` field → store as `APP_NAME`
3. **Set `APP_DIR`** = directory containing `app.json` (e.g., `frontend` or `.`)
4. **Detect iOS paths** using Glob:
   - `{APP_DIR}/ios/*/Info.plist` → store as `PLIST_PATH`
   - `{APP_DIR}/ios/*.xcodeproj/project.pbxproj` → store as `PBXPROJ_PATH`
5. **Detect Android path**: `{APP_DIR}/android/app/build.gradle` → store as `GRADLE_PATH`

If `app.json` is not found, STOP and ask the user to provide the path to their React Native app directory.

---

## Step 1: Parse Version

**If `$ARGUMENTS` is provided**, use it as the new version string (e.g., `1.0.15`).

**If no argument**, use **AskUserQuestion** to ask:
```
What version number should we update to? (e.g., 1.0.15)
Current version can be found in {APP_DIR}/package.json
```

### Calculate Build Number

The build number is derived from the semantic version:
```
major × 10000 + minor × 1000 + patch
```

Examples:
- `1.0.14` → `10014`
- `1.0.15` → `10015`
- `1.1.0`  → `11000`
- `2.0.0`  → `20000`

Store these values for use in subsequent steps:
- `NEW_VERSION` = the semantic version (e.g., `1.0.15`)
- `NEW_BUILD_NUMBER` = the calculated build number (e.g., `10015`)

---

## Step 2: Read Current Versions

Read these files to determine current version values:

1. `{APP_DIR}/package.json` — line containing `"version"`
2. `{APP_DIR}/app.json` — line containing `"version"`
3. `{PLIST_PATH}` — `CFBundleShortVersionString` and `CFBundleVersion`
4. `{GRADLE_PATH}` — `versionCode` and `versionName`
5. `{PBXPROJ_PATH}` — `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION`

Store the current version as `OLD_VERSION` and current build number as `OLD_BUILD_NUMBER`.

---

## Step 3: Update All Config Files

Apply edits **in parallel** where possible. Read each file before editing.

### 3.1 `{APP_DIR}/package.json`
```
Edit: "version": "OLD_VERSION" → "version": "NEW_VERSION"
```

### 3.2 `{APP_DIR}/app.json`
```
Edit: "version": "OLD_VERSION" → "version": "NEW_VERSION"
```

### 3.3 `{PLIST_PATH}`
Two edits:
- `CFBundleShortVersionString`: `<string>OLD_VERSION</string>` → `<string>NEW_VERSION</string>`
- `CFBundleVersion`: `<string>OLD_BUILD_NUMBER</string>` → `<string>NEW_BUILD_NUMBER</string>`

**Note:** `CFBundleVersion` may be `1` if reset by `expo prebuild`. In that case, replace `<string>1</string>` that follows `<key>CFBundleVersion</key>` with `<string>NEW_BUILD_NUMBER</string>`.

### 3.4 `{GRADLE_PATH}`
Two edits:
- `versionCode OLD_BUILD_NUMBER` → `versionCode NEW_BUILD_NUMBER`
- `versionName "OLD_VERSION"` → `versionName "NEW_VERSION"`

### 3.5 `{PBXPROJ_PATH}`
Two edits (use `replace_all: true` for each — there are 2 occurrences of each in Debug/Release configs):
- `MARKETING_VERSION = OLD_VERSION;` → `MARKETING_VERSION = NEW_VERSION;`
- `CURRENT_PROJECT_VERSION = OLD_BUILD_NUMBER;` → `CURRENT_PROJECT_VERSION = NEW_BUILD_NUMBER;`

**Note:** `MARKETING_VERSION` may be `1.0` and `CURRENT_PROJECT_VERSION` may be `1` if reset by `expo prebuild`. Check actual values before editing.

---

## Step 4: Verify Updates

After all edits, verify by grepping for the new version in each file:

```bash
grep -n "version" {APP_DIR}/package.json | head -3
grep -n "version" {APP_DIR}/app.json | head -3
grep -n "CFBundle" {PLIST_PATH}
grep -n "versionCode\|versionName" {GRADLE_PATH}
grep -n "MARKETING_VERSION\|CURRENT_PROJECT_VERSION" {PBXPROJ_PATH}
```

---

## Step 5: Report Results

```
✓ Version Updated: OLD_VERSION → NEW_VERSION
✓ Build Number: OLD_BUILD_NUMBER → NEW_BUILD_NUMBER

Files Updated:
| File                          | Field                        | Old          | New          |
|-------------------------------|------------------------------|--------------|--------------|
| {APP_DIR}/package.json        | version                      | OLD_VERSION  | NEW_VERSION  |
| {APP_DIR}/app.json            | version                      | OLD_VERSION  | NEW_VERSION  |
| {PLIST_PATH}                  | CFBundleShortVersionString   | OLD_VERSION  | NEW_VERSION  |
| {PLIST_PATH}                  | CFBundleVersion              | OLD_BUILD    | NEW_BUILD    |
| {GRADLE_PATH}                 | versionCode                  | OLD_BUILD    | NEW_BUILD    |
| {GRADLE_PATH}                 | versionName                  | OLD_VERSION  | NEW_VERSION  |
| {PBXPROJ_PATH}                | MARKETING_VERSION            | OLD_VERSION  | NEW_VERSION  |
| {PBXPROJ_PATH}                | CURRENT_PROJECT_VERSION      | OLD_BUILD    | NEW_BUILD    |
```

---

## Error Handling

| Issue | Action |
|-------|--------|
| `app.json` not found in any standard location | STOP, ask user for app directory path |
| Invalid version format (not `X.Y.Z`) | STOP, ask user for valid version |
| File not found (iOS/Android paths) | STOP, warn that native projects may need `npx expo prebuild` first |
| Version mismatch across files before update | Warn user, show current values, proceed with update |
| Edit fails (old_string not found) | Read the file again, check actual current value, retry with correct old_string |
| `expo prebuild` reset values | Check for `1.0` / `1` as current values and handle accordingly |
