---
description: Review React Native code for quality, patterns compliance, and issues
argument-hint: "[file or directory] [--severity critical|warning|all]"
---

# Code Review Command

Review React Native code for quality, adherence to project patterns, performance, and accessibility.

---

## Phase 1: Parse Arguments

Arguments: $ARGUMENTS

**Defaults:**
- Target: current directory
- Severity: `all`

**Argument Parsing:**
- First arg = file path or directory
- `--severity critical` shows only critical issues
- `--severity warning` shows warnings and critical
- `--severity all` shows everything (default)

---

## Phase 2: Scan Target Files

### Identify Files

```bash
# Find all TypeScript/TSX files, skip node_modules and test files
find {target} -type f \( -name "*.tsx" -o -name "*.ts" \) \
  ! -path "*/node_modules/*" \
  ! -path "*/__tests__/*" \
  ! -path "*.test.*" \
  ! -path "*.spec.*"
```

### Categorize Files

- **Components:** `.tsx` files exporting React components
- **Hooks:** files matching `use*.ts`
- **Screens:** files in `screens/` or `app/` directories
- **Services:** files in `services/` or `api/` directories
- **Store:** files in `store/` or `slices/` directories

---

## Phase 3: Review Checklist Execution

### 1. TypeScript Quality (Critical)

- [ ] No `any` type usage — use `unknown` or proper types
- [ ] Explicit return types on exported functions
- [ ] Proper import paths (use path aliases like `@/`)
- [ ] No `@ts-ignore` or `@ts-expect-error` without explanation
- [ ] Interfaces/types defined for all props

### 2. Component Patterns (Critical)

- [ ] Uses `Pressable` — never `TouchableOpacity`
- [ ] Uses NativeWind `className` — never `StyleSheet.create` as primary
- [ ] Uses React Native Paper components where appropriate
- [ ] Function components only — no class components
- [ ] Props destructured with TypeScript interface

### 3. Performance (Warning)

- [ ] `React.memo` on list item components
- [ ] `FlatList` used for lists — never `ScrollView` with `.map()`
- [ ] No inline function definitions in FlatList `renderItem`
- [ ] `useCallback` for event handlers passed to children
- [ ] `useMemo` for expensive computations
- [ ] Images have explicit dimensions

### 4. Accessibility (Warning)

- [ ] `accessibilityLabel` on all interactive elements
- [ ] `accessibilityRole` on Pressable, Button, TextInput
- [ ] Touch targets minimum 44x44pt
- [ ] `accessibilityHint` on non-obvious actions

### 5. State Management (Critical)

- [ ] Redux Toolkit for client state (auth, UI preferences)
- [ ] TanStack Query for server data — never Redux for API caching
- [ ] No prop drilling beyond 2 levels
- [ ] `useSelector` with specific selectors — never select entire state

### 6. Error Handling (Critical)

- [ ] All async operations wrapped in try/catch
- [ ] API errors handled with user-facing messages
- [ ] Error boundaries around screen-level components
- [ ] Loading and error states handled in UI

### 7. Security (Critical)

- [ ] No hardcoded API keys, tokens, or secrets
- [ ] Sensitive data stored in secure storage (not AsyncStorage/MMKV for secrets)
- [ ] No `console.log` with sensitive data
- [ ] API base URLs from environment config

---

## Phase 4: Generate Report

### Issue Table

| Severity | File:Line | Issue | Suggestion |
|----------|-----------|-------|------------|
| Critical | `src/screens/Login.tsx:45` | Uses `any` type | Define `LoginFormData` interface |
| Warning | `src/components/UserCard.tsx:12` | Missing `React.memo` | Wrap with `memo()` — used in FlatList |
| Info | `src/hooks/useAuth.ts:30` | Missing return type | Add `: UseAuthReturn` |

### Summary

```
Critical: X issues
Warning:  X issues
Info:     X issues
Total:    X issues across Y files

Overall Quality Score: B
```

### Quality Score Criteria

| Grade | Criteria |
|-------|----------|
| A | 0 critical, 0-2 warnings |
| B | 0 critical, 3-5 warnings |
| C | 1-2 critical, any warnings |
| D | 3-5 critical issues |
| F | 6+ critical issues or security violations |

---

## Related Resources

- [code-review-checklist.md](../skills/code-quality/code-review-checklist.md)
- [frontend-dev-guidelines.md](../skills/development/frontend-dev-guidelines.md)
- [common-patterns.md](../guides/common-patterns.md)
