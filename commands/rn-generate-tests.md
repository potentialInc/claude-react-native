---
description: Generate unit or E2E tests for React Native components, hooks, or screens
argument-hint: "[target-file] [--type unit|e2e|both]"
---

# Generate Tests Command

Analyze a React Native component, hook, or screen and generate comprehensive unit or E2E tests.

---

## Phase 1: Parse Arguments

Arguments: $ARGUMENTS

**Defaults:**
- Type: `unit`

**Argument Parsing:**
- First arg = path to the target file (required)
- `--type unit` generates Jest + RNTL tests (default)
- `--type e2e` generates Maestro YAML or Detox tests
- `--type both` generates both unit and E2E tests

---

## Phase 2: Analyze Target File

### Read the File Completely

```bash
# Read the target file
cat {target-file}
```

### Identify Component Type

| Type | Indicators |
|------|-----------|
| **Presentational** | Props in, JSX out, no side effects |
| **Form** | Uses `useForm`, `Controller`, `TextInput`, submit handler |
| **Screen** | In `screens/` or `app/`, uses navigation, fetches data |
| **Hook** | Filename `use*.ts`, returns state/handlers |
| **Service** | API calls, data transformation |

### Identify Dependencies

- **Navigation:** `useNavigation`, `useRouter`, `useLocalSearchParams`
- **Redux:** `useSelector`, `useDispatch`, `useAppSelector`
- **TanStack Query:** `useQuery`, `useMutation`, `useQueryClient`
- **HTTP:** `httpService`, `axios`, API calls
- **Context:** `useContext`, custom context hooks

### Identify User Interactions

- Button/Pressable presses
- TextInput changes
- FlatList scrolling
- Switch toggles
- Form submissions
- Pull-to-refresh
- Swipe gestures

### Identify Async Operations

- API calls (fetch, httpService)
- Timers (setTimeout, setInterval)
- Animations
- Navigation transitions
- Storage operations (MMKV reads/writes)

---

## Phase 3: Generate Tests

### Unit Tests

Create `__tests__/TargetFile.test.tsx` alongside the target file.

#### Test File Structure

```typescript
import React from 'react';
import { render, screen, fireEvent, waitFor, act } from '@testing-library/react-native';

// Import target
import { TargetComponent } from '../TargetComponent';

// Mock dependencies
jest.mock('@react-navigation/native', () => ({
  useNavigation: () => ({ navigate: jest.fn() }),
}));

jest.mock('@/services/httpService', () => ({
  get: jest.fn(),
  post: jest.fn(),
}));

// Helper: wrap with providers if Redux/Query needed
const renderWithProviders = (ui: React.ReactElement) => {
  return render(
    <QueryClientProvider client={new QueryClient()}>
      <Provider store={mockStore}>{ui}</Provider>
    </QueryClientProvider>,
  );
};

describe('TargetComponent', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  // Rendering
  it('renders correctly with default props', () => {});
  it('renders correctly with provided props', () => {});

  // User Interactions
  it('handles button press', () => {});
  it('handles text input change', () => {});

  // State Changes
  it('updates state on user action', () => {});
  it('shows loading state during async operation', () => {});

  // Error States
  it('displays error message on API failure', () => {});
  it('handles empty data gracefully', () => {});

  // Edge Cases
  it('handles rapid consecutive presses', () => {});
  it('unmounts cleanly without warnings', () => {});
});
```

#### Test Patterns by Component Type

**Presentational Component:**
- Renders with required props
- Renders with optional props
- Snapshot test
- Conditional rendering based on props

**Form Component:**
- Renders all form fields
- Validates required fields
- Shows validation errors
- Submits with valid data
- Handles submission errors

**Screen Component:**
- Renders loading state
- Renders data state
- Renders error state
- Navigation on user action
- Pull-to-refresh behavior

**Hook:**
- Returns initial state
- Updates state on action
- Handles async operations
- Cleans up on unmount

### E2E Tests

#### Maestro (Recommended)

Create `e2e/flows/{target-name}.yaml`:

```yaml
appId: ${APP_ID}
---
- launchApp:
    clearState: true

# Happy Path
- tapOn: "Element Label"
- inputText: "test input"
- assertVisible: "Expected Result"

# Error Path
- tapOn: "Submit"
- assertVisible: "Validation error message"
```

#### Detox

Create `e2e/tests/{target-name}.test.ts`:

```typescript
describe('TargetScreen', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  // Happy path
  it('should complete flow successfully', async () => {
    await element(by.id('input-field')).typeText('test');
    await element(by.id('submit-button')).tap();
    await expect(element(by.text('Success'))).toBeVisible();
  });

  // Error path
  it('should show error on invalid input', async () => {
    await element(by.id('submit-button')).tap();
    await expect(element(by.text('Required'))).toBeVisible();
  });
});
```

---

## Phase 4: Verify Generated Tests

### Run Unit Tests

```bash
npx jest path/to/__tests__/TargetFile.test.tsx --verbose
```

### Check Results

1. All tests pass
2. No console warnings about act() or unmount
3. No unhandled promise rejections
4. Coverage of the target file >80%

### If Tests Fail

1. Read the error message
2. Fix mock setup or assertions
3. Re-run until all tests pass
4. Report any issues that indicate bugs in the source file

---

## Related Resources

- [unit-test-generator.md](../skills/e2e-testing/unit-test-generator.md)
- [e2e-test-generator.md](../skills/e2e-testing/e2e-test-generator.md)
- [test-mocking-patterns.md](../skills/e2e-testing/test-mocking-patterns.md)
- [mobile-testing.md](../guides/mobile-testing.md)
