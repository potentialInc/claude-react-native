# E2E Test Fixtures Guide

Test data and fixture management for React Native E2E tests with Detox and Maestro.

---

## Test User Data

### User Fixtures

```typescript
// e2e/fixtures/users.ts

export interface TestUser {
  id: string;
  email: string;
  password: string;
  name: string;
  role: 'patient' | 'coach' | 'admin';
}

export const testUsers: Record<string, TestUser> = {
  patient: {
    id: '1',
    email: 'patient@test.com',
    password: 'Patient@123',
    name: 'Test Patient',
    role: 'patient',
  },
  coach: {
    id: '2',
    email: 'coach@test.com',
    password: 'Coach@123',
    name: 'Test Coach',
    role: 'coach',
  },
  admin: {
    id: '3',
    email: 'admin@test.com',
    password: 'Admin@123',
    name: 'Test Admin',
    role: 'admin',
  },
};
```

---

## Authentication Fixtures

### Login Helper (Detox)

```typescript
// e2e/fixtures/auth.ts
import { by, element, expect, waitFor } from 'detox';
import { testUsers, TestUser } from './users';

export async function login(user: TestUser): Promise<void> {
  await waitFor(element(by.id('login-screen')))
    .toBeVisible()
    .withTimeout(5000);

  await element(by.id('email-input')).clearText();
  await element(by.id('email-input')).typeText(user.email);

  await element(by.id('password-input')).clearText();
  await element(by.id('password-input')).typeText(user.password);

  await element(by.id('login-button')).tap();

  // Wait for home screen
  await waitFor(element(by.id('home-screen')))
    .toBeVisible()
    .withTimeout(10000);
}

export async function loginAsPatient(): Promise<void> {
  await login(testUsers.patient);
}

export async function loginAsCoach(): Promise<void> {
  await login(testUsers.coach);
}

export async function logout(): Promise<void> {
  await element(by.id('tab-profile')).tap();
  await element(by.id('logout-button')).tap();
  await waitFor(element(by.id('login-screen')))
    .toBeVisible()
    .withTimeout(5000);
}
```

### Login Flow (Maestro)

```yaml
# e2e/flows/common/login.yaml
appId: ${APP_ID}

- assertVisible:
    id: "login-screen"

- tapOn:
    id: "email-input"
- inputText: ${EMAIL}

- tapOn:
    id: "password-input"
- inputText: ${PASSWORD}

- tapOn:
    id: "login-button"

- assertVisible:
    id: "home-screen"
    timeout: 10000
```

### Using Login in Maestro Flows

```yaml
# e2e/flows/profile/edit-profile.yaml
appId: com.yourapp.mobile

- launchApp:
    clearState: true

# Login first
- runFlow:
    file: ../common/login.yaml
    env:
      EMAIL: "patient@test.com"
      PASSWORD: "Patient@123"

# Now test profile editing
- tapOn:
    id: "tab-profile"

- tapOn:
    id: "edit-profile-button"
```

---

## Mock Data Fixtures

### Generic Test Data

```typescript
// e2e/fixtures/testData.ts

export interface TestItem {
  id: string;
  title: string;
  description: string;
}

export const mockItems: TestItem[] = [
  {
    id: '1',
    title: 'Test Item 1',
    description: 'Description for item 1',
  },
  {
    id: '2',
    title: 'Test Item 2',
    description: 'Description for item 2',
  },
];

// Data generators
export function generateUniqueEmail(): string {
  return `test-${Date.now()}@test.com`;
}

export function generateUniquePhone(): string {
  const random = Math.floor(Math.random() * 90000000) + 10000000;
  return `010${random}`;
}

export function generateUniqueId(): string {
  return `test-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

---

## AsyncStorage Mocking

### Pre-populate AsyncStorage

```typescript
// e2e/fixtures/storage.ts
import { device } from 'detox';

export async function setStorageItem(key: string, value: unknown): Promise<void> {
  // Use device.launchApp with launchArgs to inject data
  // Or use a test-only API endpoint to set storage
}

export async function clearStorage(): Promise<void> {
  await device.launchApp({ delete: true });
}

export const mockStorageData = {
  onboardingComplete: true,
  userPreferences: {
    theme: 'light',
    notifications: true,
  },
};
```

### Launch with Pre-set Data

```typescript
// In test file
beforeAll(async () => {
  await device.launchApp({
    newInstance: true,
    launchArgs: {
      detoxEnableSynchronization: 0,
    },
  });
});
```

---

## API Mocking

### Mock Server Setup

For E2E tests, you have several options:

#### Option 1: Test Environment API

```typescript
// e2e/fixtures/api.ts

// Point to test backend
export const API_BASE_URL = 'https://api-test.yourapp.com';

// Or local mock server
export const API_BASE_URL = 'http://localhost:3001';
```

#### Option 2: MSW (Mock Service Worker)

```typescript
// e2e/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('*/api/auth/login', () => {
    return HttpResponse.json({
      success: true,
      data: {
        token: 'mock-token',
        user: testUsers.patient,
      },
    });
  }),

  http.get('*/api/items', () => {
    return HttpResponse.json({
      success: true,
      data: mockItems,
    });
  }),
];
```

#### Option 3: Network Mocking in Detox

```typescript
// Block real API calls
await device.setURLBlacklist(['.*api.yourapp.com.*']);

// Allow specific endpoints
await device.setURLBlacklist([]);
```

---

## Device State Fixtures

### Device Helpers

```typescript
// e2e/fixtures/device.ts
import { device } from 'detox';

export const DeviceFixtures = {
  async resetApp(): Promise<void> {
    await device.launchApp({ delete: true, newInstance: true });
  },

  async reloadApp(): Promise<void> {
    await device.reloadReactNative();
  },

  async setPermissions(permissions: Record<string, string>): Promise<void> {
    await device.launchApp({
      newInstance: true,
      permissions,
    });
  },

  async grantCameraPermission(): Promise<void> {
    await this.setPermissions({ camera: 'YES' });
  },

  async grantLocationPermission(): Promise<void> {
    await this.setPermissions({ location: 'always' });
  },

  async setMockLocation(lat: number, lon: number): Promise<void> {
    await device.setLocation(lat, lon);
  },

  async simulateNotification(payload: object): Promise<void> {
    await device.sendUserNotification({
      trigger: { type: 'push' },
      ...payload,
    });
  },
};
```

---

## Maestro Environment Variables

### .env File for Maestro

```bash
# e2e/.env.maestro
APP_ID=com.yourapp.mobile
BASE_URL=https://api-test.yourapp.com

# Test credentials
PATIENT_EMAIL=patient@test.com
PATIENT_PASSWORD=Patient@123
COACH_EMAIL=coach@test.com
COACH_PASSWORD=Coach@123
```

### Using Variables in Flows

```yaml
# e2e/flows/login-patient.yaml
appId: ${APP_ID}

- launchApp:
    clearState: true

- runFlow:
    file: common/login.yaml
    env:
      EMAIL: ${PATIENT_EMAIL}
      PASSWORD: ${PATIENT_PASSWORD}
```

### Running with Variables

```bash
maestro test --env-file e2e/.env.maestro e2e/flows/
```

---

## Test Data Cleanup

### After Each Test

```typescript
// e2e/helpers/cleanup.ts
import { device } from 'detox';

export async function cleanupAfterTest(): Promise<void> {
  // Logout if logged in
  try {
    await element(by.id('tab-profile')).tap();
    await element(by.id('logout-button')).tap();
  } catch {
    // Ignore if not logged in
  }
}

// In test file
afterEach(async () => {
  await cleanupAfterTest();
});
```

### Before All Tests

```typescript
beforeAll(async () => {
  // Fresh app install
  await device.launchApp({ delete: true, newInstance: true });
});
```

---

## Best Practices

### 1. Isolate Test Data

```typescript
// Each test should use unique data
it('should create new item', async () => {
  const uniqueTitle = `Item-${Date.now()}`;
  // Use uniqueTitle
});
```

### 2. Don't Share State Between Tests

```typescript
// Reset state in beforeEach
beforeEach(async () => {
  await device.reloadReactNative();
});
```

### 3. Use Meaningful Test Data

```typescript
// Good - realistic data
const testUser = {
  email: 'john.doe@example.com',
  name: 'John Doe',
};

// Bad - meaningless data
const testUser = {
  email: 'test@test.com',
  name: 'Test',
};
```

### 4. Centralize Test Data

```typescript
// All test data in fixtures/
import { testUsers } from '../fixtures/users';
import { mockItems } from '../fixtures/testData';

// Don't scatter data in test files
```

### 5. Environment-Specific Fixtures

```typescript
// e2e/fixtures/config.ts
export const testConfig = {
  development: {
    apiUrl: 'http://localhost:3000',
    timeout: 10000,
  },
  staging: {
    apiUrl: 'https://api-staging.yourapp.com',
    timeout: 15000,
  },
};

const env = process.env.TEST_ENV || 'development';
export const config = testConfig[env];
```

---

## Related Files

- [e2e-test-generator.md](e2e-test-generator.md) - Test patterns
- [e2e-page-objects.md](e2e-page-objects.md) - Page object patterns
- [mobile-testing.md](../guides/mobile-testing.md) - Debugging guide
