---
description: Audit React Native components for accessibility compliance
argument-hint: "[component-file or directory]"
---

# Accessibility Audit Command

Audit React Native components for accessibility compliance, covering labels, roles, touch targets, and platform-specific requirements.

---

## Phase 1: Parse Arguments

Arguments: $ARGUMENTS

**Defaults:**
- Target: `src/components/` and `src/screens/` (or `src/app/` for Expo Router)

**Argument Parsing:**
- First arg = component file path or directory to audit

---

## Phase 2: Scan for Components

### Identify Target Files

```bash
# Find all TSX files with JSX content
find {target} -type f -name "*.tsx" \
  ! -path "*/node_modules/*" \
  ! -path "*/__tests__/*" \
  ! -path "*.test.*"
```

### Categorize Components

- **Interactive:** Components containing `Pressable`, `Button`, `TextInput`, `Switch`
- **Media:** Components containing `Image`, `ImageBackground`
- **Navigation:** Components with navigation actions or links
- **Informational:** Components displaying data (lists, cards, text)

---

## Phase 3: Accessibility Audit

### Check 1: Missing Labels on Interactive Elements (Critical)

Scan for `Pressable`, `Button`, `IconButton`, and `Switch` without `accessibilityLabel`:

```typescript
// BAD — no label
<Pressable onPress={handleDelete}>
  <Icon name="trash" />
</Pressable>

// GOOD — labeled
<Pressable
  onPress={handleDelete}
  accessibilityLabel="Delete item"
  accessibilityRole="button"
>
  <Icon name="trash" />
</Pressable>
```

### Check 2: Images Without Labels (Critical)

Scan for `Image` and `ImageBackground` without `accessibilityLabel`:

```typescript
// BAD — unlabeled content image
<Image source={userAvatar} />

// GOOD — labeled
<Image source={userAvatar} accessibilityLabel="User profile photo" />

// GOOD — decorative image
<Image source={bgPattern} accessibilityElementsHidden={true} />
```

### Check 3: TextInput Without Labels (Critical)

Scan for `TextInput` without `accessibilityLabel`:

```typescript
// BAD — relies only on placeholder
<TextInput placeholder="Enter email" />

// GOOD — explicit label
<TextInput
  placeholder="Enter email"
  accessibilityLabel="Email address"
/>
```

### Check 4: Missing accessibilityRole (Warning)

Scan interactive elements without `accessibilityRole`:

| Element | Expected Role |
|---------|--------------|
| `Pressable` (navigation) | `link` |
| `Pressable` (action) | `button` |
| `TextInput` | `search` or default |
| `Switch` | `switch` |
| `Image` | `image` |
| Heading `Text` | `header` |

### Check 5: Touch Target Size (Warning)

Scan for explicit small dimensions on interactive elements:

```typescript
// BAD — too small (below 44x44pt)
<Pressable className="h-6 w-6" onPress={handlePress}>

// GOOD — meets minimum
<Pressable className="h-11 w-11" onPress={handlePress}>

// GOOD — uses hitSlop for small visual elements
<Pressable
  className="h-6 w-6"
  hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}
  onPress={handlePress}
>
```

### Check 6: Color Contrast (Info)

Check theme configuration and hardcoded colors:
- Text on background must meet WCAG AA (4.5:1 for normal text, 3:1 for large text)
- Look for light gray text on white backgrounds
- Flag hardcoded colors that may have low contrast

### Check 7: Missing accessibilityHint (Info)

Scan for non-obvious interactive elements without `accessibilityHint`:

```typescript
// GOOD — hint explains non-obvious behavior
<Pressable
  onPress={handleLongAction}
  accessibilityLabel="Submit report"
  accessibilityHint="Sends the report to your manager for review"
  accessibilityRole="button"
>
```

---

## Phase 4: Generate A11y Report

### Issues by Severity

**Critical (Must Fix)**
- Missing `accessibilityLabel` on interactive elements
- Missing `accessibilityLabel` on content images
- Missing `accessibilityLabel` on text inputs

**Warning (Should Fix)**
- Missing `accessibilityRole` on interactive elements
- Touch targets below 44x44pt without `hitSlop`
- Missing `accessibilityState` for toggleable elements

**Info (Consider)**
- Missing `accessibilityHint` on non-obvious actions
- Potential color contrast issues
- Missing `accessibilityLiveRegion` on dynamic content

### Report Format

| Severity | File:Line | Issue | Remediation |
|----------|-----------|-------|-------------|
| Critical | `Button.tsx:15` | Pressable missing accessibilityLabel | Add `accessibilityLabel="Action description"` |
| Warning | `IconBtn.tsx:8` | Touch target 24x24pt | Add `hitSlop={10}` or increase to 44x44pt |
| Info | `Card.tsx:22` | Pressable missing accessibilityHint | Add `accessibilityHint="..."` |

### Platform-Specific Notes

**iOS:**
- `accessibilityLabel` maps to VoiceOver label
- `accessibilityHint` is read after a pause by VoiceOver
- Use `accessibilityElementsHidden` for decorative elements

**Android:**
- `accessibilityLabel` maps to TalkBack content description
- Use `importantForAccessibility="no"` for decorative elements
- `accessibilityLiveRegion="polite"` for dynamic updates

---

## Related Resources

- [accessible-component.md](../skills/ui-design/accessible-component.md)
- [common-patterns.md](../guides/common-patterns.md)
