---
description: Run unit tests for React Native app using Jest + React Native Testing Library
argument-hint: "[file-pattern or 'all'] [--coverage] [--watch]"
---

# Run Unit Tests Command

Run unit tests for the React Native mobile app using Jest and React Native Testing Library.

---

## Phase 1: Parse Arguments

Arguments: $ARGUMENTS

**Defaults:**
- Tests: `all`
- Coverage: off
- Watch: off

**Argument Parsing:**
- First arg = file path pattern or `all`
- `--coverage` flag enables coverage report
- `--watch` flag enables watch mode
- `--onlyFailures` flag re-runs only previously failed tests

---

## Phase 2: Prerequisites Check

### Verify Configuration

```bash
# Check jest.config exists
ls jest.config.ts jest.config.js 2>/dev/null || echo "ERROR: No jest.config found"

# Check setup file exists
ls jest.setup.ts jest.setup.js 2>/dev/null || echo "WARNING: No jest.setup file found"

# Check dependencies are installed
npx jest --version || echo "ERROR: Jest not installed — run npm install"
```

### Required Dependencies

Verify these are in `package.json`:
- `jest`
- `@testing-library/react-native`
- `@testing-library/jest-native`
- `jest-expo` (if using Expo)

---

## Phase 3: Run Tests

### Single File

```bash
npx jest path/to/file.test.tsx
```

### All Tests

```bash
npx jest
```

### With Coverage

```bash
npx jest --coverage
```

### Watch Mode

```bash
npx jest --watch
```

### Failed Only

```bash
npx jest --onlyFailures
```

### Specific Pattern

```bash
npx jest --testPathPattern="login"
```

### Verbose Output

```bash
npx jest --verbose
```

---

## Phase 4: Report Results

### Parse Jest Output

1. Extract pass/fail counts from Jest summary line
2. List all failed tests with file:line references
3. Show error messages and stack traces for failures
4. If `--coverage` was used, show coverage summary table:

| File | Statements | Branches | Functions | Lines |
|------|-----------|----------|-----------|-------|
| src/components/ | 85% | 72% | 90% | 84% |

### Suggest Missing Tests

If `--coverage` was used:
- Identify files with <50% coverage
- Suggest specific test cases for uncovered branches
- Prioritize untested user-facing components and hooks

---

## Common Issues

### Module Not Found

```
Check jest.config moduleNameMapper:
moduleNameMapper: {
  '^@/(.*)$': '<rootDir>/src/$1',
}
```

### Transform Errors

```
Check transformIgnorePatterns in jest.config:
transformIgnorePatterns: [
  'node_modules/(?!((jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)/)',
]
```

### Timeout Errors

```
Increase testTimeout in jest.config:
testTimeout: 10000,

Or per-test:
jest.setTimeout(15000);
```

### Act Warnings

```
Wrap state-changing operations in act():
await act(async () => {
  fireEvent.press(button);
});
```

---

## Related Resources

- [unit-test-generator.md](../skills/e2e-testing/unit-test-generator.md)
- [test-mocking-patterns.md](../skills/e2e-testing/test-mocking-patterns.md)
- [mobile-testing.md](../guides/mobile-testing.md)
