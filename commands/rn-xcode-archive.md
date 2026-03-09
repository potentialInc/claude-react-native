---
description: Build an Xcode archive for iOS — focused iOS-only shortcut (for full Android+iOS builds, use /rn-local-build)
argument-hint: (no arguments)
---

# Xcode Archive Build

You are a build assistant for an iOS app. Your task is to verify version consistency and build an Xcode archive.

## Step 0: Auto-Detect Project Structure

Detect the project layout:

1. **Find `app.json`** — search in order: `./app.json`, `frontend/app.json`, `mobile/app.json`, `app/app.json`
2. **Read `app.json`** to extract the app name from `expo.name` or `name` field → store as `APP_NAME`
3. **Set `APP_DIR`** = directory containing `app.json` (e.g., `frontend` or `.`)
4. **Detect iOS paths** using Glob:
   - `{APP_DIR}/ios/*/Info.plist` → store as `PLIST_PATH`
   - `{APP_DIR}/ios/*.xcodeproj/project.pbxproj` → store as `PBXPROJ_PATH`
   - `{APP_DIR}/ios/*.xcworkspace` → store as `WORKSPACE_PATH`
5. **Set `SCHEME`** = `APP_NAME` (derived from workspace name or app.json)

If `app.json` is not found, STOP and ask the user to provide the path to their React Native app directory.

---

## Step 1: Read Current iOS Version Info

Read these files to collect version values:

1. `{PLIST_PATH}` — `CFBundleShortVersionString` and `CFBundleVersion`
2. `{PBXPROJ_PATH}` — `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION`
3. `{APP_DIR}/app.json` — `version` (source of truth)

Store these values:
- `APP_JSON_VERSION` = version from app.json
- `PLIST_VERSION` = CFBundleShortVersionString
- `PLIST_BUILD` = CFBundleVersion
- `PBXPROJ_MARKETING` = MARKETING_VERSION
- `PBXPROJ_BUILD` = CURRENT_PROJECT_VERSION

---

## Step 2: Verify Version Consistency

Check that all values are in sync:

| Source | Field | Expected |
|--------|-------|----------|
| app.json | version | APP_JSON_VERSION |
| Info.plist | CFBundleShortVersionString | = APP_JSON_VERSION |
| Info.plist | CFBundleVersion | = calculated build number |
| project.pbxproj | MARKETING_VERSION | = APP_JSON_VERSION |
| project.pbxproj | CURRENT_PROJECT_VERSION | = calculated build number |

**Build number calculation**: `major × 10000 + minor × 1000 + patch`

### If values are out of sync:

This commonly happens after `expo prebuild --clean` which resets `MARKETING_VERSION` to `1.0` and `CURRENT_PROJECT_VERSION` to `1`.

Use **AskUserQuestion** to ask:
```
iOS version files are out of sync:
- app.json: APP_JSON_VERSION
- Info.plist version: PLIST_VERSION
- Info.plist build: PLIST_BUILD
- project.pbxproj MARKETING_VERSION: PBXPROJ_MARKETING
- project.pbxproj CURRENT_PROJECT_VERSION: PBXPROJ_BUILD

Options:
1. Fix to match app.json (APP_JSON_VERSION / BUILD_NUMBER) — Recommended
2. Cancel and fix manually
```

**If user chooses to fix**: Apply the necessary edits to bring all files in sync with app.json, using `replace_all: true` for project.pbxproj edits (2 occurrences each for Debug/Release configs).

### If values are in sync:

Proceed to Step 3.

---

## Step 3: Build Xcode Archive

```bash
xcodebuild \
  -workspace {WORKSPACE_PATH} \
  -scheme {SCHEME} \
  -configuration Release \
  -archivePath /tmp/{APP_NAME}.xcarchive \
  -allowProvisioningUpdates \
  archive
```

**Important notes:**
- Use `/tmp/{APP_NAME}.xcarchive` as archive path (the `build/` directory may be blocked by security policy)
- The `-allowProvisioningUpdates` flag is required for automatic signing via CLI
- This build can take 3-8 minutes — use `timeout: 600000` (10 min)
- Pipe output through `tail -5` to avoid excessive log output: append `2>&1 | tail -5`

---

## Step 4: Verify Archive

After successful build, verify the archive contains correct version info:

```bash
/usr/libexec/PlistBuddy -c "Print :ApplicationProperties:CFBundleShortVersionString" /tmp/{APP_NAME}.xcarchive/Info.plist
/usr/libexec/PlistBuddy -c "Print :ApplicationProperties:CFBundleVersion" /tmp/{APP_NAME}.xcarchive/Info.plist
```

---

## Step 5: Report Results

### Success:
```
✓ Xcode Archive Built Successfully

Archive: /tmp/{APP_NAME}.xcarchive
Version: X.Y.Z
Build:   XXXXX

Next steps:
- Open Xcode Organizer to distribute to App Store Connect
- Or use: xcodebuild -exportArchive -archivePath /tmp/{APP_NAME}.xcarchive -exportOptionsPlist ExportOptions.plist -exportPath /tmp/{APP_NAME}Export
```

### Failure:
```
✗ Xcode Archive Build Failed

Error: <error description>

Common fixes:
- Provisioning profile issues → Check signing team in Xcode (must be organization team, not personal)
- Sign In with Apple error → Personal teams don't support this capability, switch to organization team in Xcode signing settings
- Pod issues → Run: cd {APP_DIR}/ios && pod install
- Clean build → Run: xcodebuild clean -workspace {WORKSPACE_PATH} -scheme {SCHEME}
```

---

## Error Handling

| Issue | Action |
|-------|--------|
| `app.json` not found in any standard location | STOP, ask user for app directory path |
| `ARCHIVE FAILED` | Show last 30 lines of output, identify error |
| Provisioning profile not found | Suggest `-allowProvisioningUpdates` (already included) or check Xcode signing settings |
| Personal team can't support Sign In with Apple | STOP, tell user to switch to organization team in Xcode signing settings |
| Archive path blocked by security policy | Already handled — using `/tmp/` path |
| Build number mismatch in archive | Re-check Info.plist and project.pbxproj, fix and rebuild |
| CocoaPods out of date | Suggest `cd {APP_DIR}/ios && pod install` before retry |

---

## See Also
- **Full local build (Android + iOS)**: Use `/rn-local-build` for a complete build workflow including Android APK, version bumping, and environment checks
- **EAS cloud builds**: Use `/rn-eas-build` for building via Expo Application Services without local native toolchain
