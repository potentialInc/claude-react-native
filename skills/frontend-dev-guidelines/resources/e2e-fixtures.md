# E2E Test Fixtures Guide

Test data and fixture management for Playwright E2E tests.

## Table of Contents

- [Authentication Fixtures](#authentication-fixtures)
- [User Data Fixtures](#user-data-fixtures)
- [API Mocking](#api-mocking)
- [Best Practices](#best-practices)

---

## Authentication Fixtures

### Test Users

```typescript
// frontend/test/fixtures/auth.fixture.ts

export const testUsers = {
  coach: {
    email: 'coach@test.com',
    password: 'CoachTest@123',
    role: 'COACH',
    name: '김코치',
  },
  patient: {
    email: 'patient@test.com',
    password: 'PatientTest@123',
    role: 'PATIENT',
    name: '이환자',
  },
  admin: {
    email: 'admin@test.com',
    password: 'AdminTest@123',
    role: 'ADMIN',
    name: '관리자',
  },
};
```

### Authentication Helpers

```typescript
// Authenticate via UI (realistic but slower)
export async function authenticateAsCoach(page: Page): Promise<void> {
  await page.goto('/login');
  await page.getByPlaceholder('Enter your email').fill(testUsers.coach.email);
  await page.getByPlaceholder('Enter your password').fill(testUsers.coach.password);
  await page.getByRole('button', { name: /sign in/i }).click();
  await page.waitForURL('/coach');
}

export async function authenticateAsPatient(page: Page): Promise<void> {
  await authenticate(page, testUsers.patient);
}
```

### Fast Authentication (Storage Injection)

```typescript
// For tests where login flow is not the focus
export async function authenticateViaStorage(
  page: Page,
  user: TestUser,
  tokens: { accessToken: string; refreshToken: string }
): Promise<void> {
  await page.goto('/');
  await page.evaluate(
    ({ user, tokens }) => {
      localStorage.setItem('token', tokens.accessToken);
      localStorage.setItem('refreshToken', tokens.refreshToken);
      localStorage.setItem('user', JSON.stringify(user));
    },
    { user, tokens }
  );
  await page.reload();
}
```

### Usage in Tests

```typescript
import { test } from '@playwright/test';
import { authenticateAsCoach, logout } from '../../fixtures/auth.fixture';

test.describe('Coach Dashboard', () => {
  test.beforeEach(async ({ page }) => {
    await authenticateAsCoach(page);
  });

  test.afterEach(async ({ page }) => {
    await logout(page);
  });

  test('should show dashboard', async ({ page }) => {
    await page.goto('/coach');
    // Test authenticated page
  });
});
```

---

## User Data Fixtures

### Mock Data Types

```typescript
// frontend/test/fixtures/user.fixture.ts

export interface TestPatientData {
  id: string;
  name: string;
  lastExercise: string;
  lastSurvey: string;
  progress: number;
}

export interface TestExerciseData {
  id: string;
  title: string;
  description: string;
  sets: number;
  reps: number;
  duration: string;
}
```

### Mock Data Sets

```typescript
export const mockPatients: TestPatientData[] = [
  {
    id: '1',
    name: '이재활',
    lastExercise: '3시간 전',
    lastSurvey: '어제',
    progress: 75,
  },
  {
    id: '2',
    name: '최휴식',
    lastExercise: '7일 전',
    lastSurvey: '3일 전',
    progress: 45,
  },
];

export const mockExercises: TestExerciseData[] = [
  {
    id: '1',
    title: '어깨 스트레칭',
    description: '어깨 관절 가동 범위 증가를 위한 스트레칭',
    sets: 3,
    reps: 10,
    duration: '15초',
  },
];
```

### Test Data Generators

```typescript
// Generate unique test data
export function generateTestEmail(): string {
  return `test-${Date.now()}@test.com`;
}

export function generateTestPhone(): string {
  const random = Math.floor(Math.random() * 90000000) + 10000000;
  return `010${random}`;
}

export function getTodayDate(): string {
  return new Date().toISOString().split('T')[0];
}

export function getRelativeDate(daysOffset: number): string {
  const date = new Date();
  date.setDate(date.getDate() + daysOffset);
  return date.toISOString().split('T')[0];
}
```

---

## API Mocking

### Response Wrapper

```typescript
// frontend/test/utils/api-mocking.ts

interface ApiResponse<T> {
  success: boolean;
  statusCode: number;
  message: string;
  data: T;
}

function successResponse<T>(data: T, statusCode = 200): ApiResponse<T> {
  return { success: true, statusCode, message: 'Success', data };
}

function errorResponse(message: string, statusCode = 400): ApiResponse<null> {
  return { success: false, statusCode, message, data: null };
}
```

### Mock API Endpoints

```typescript
// Mock successful login
export async function mockLoginApi(page: Page): Promise<void> {
  await page.route('**/api/auth/login', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(successResponse({
        accessToken: 'mock-token',
        refreshToken: 'mock-refresh',
        user: { id: '1', email: 'test@test.com', role: 'PATIENT' },
      })),
    });
  });
}

// Mock patients list
export async function mockPatientsApi(page: Page): Promise<void> {
  await page.route('**/api/patients*', async (route) => {
    await route.fulfill({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify(successResponse(mockPatients)),
    });
  });
}
```

### Error Mocking

```typescript
// Mock any API error
export async function mockApiError(
  page: Page,
  urlPattern: string,
  statusCode: number,
  message: string
): Promise<void> {
  await page.route(urlPattern, async (route) => {
    await route.fulfill({
      status: statusCode,
      contentType: 'application/json',
      body: JSON.stringify(errorResponse(message, statusCode)),
    });
  });
}

// Convenience methods
export async function mockNotFound(page: Page, url: string): Promise<void> {
  await mockApiError(page, url, 404, 'Resource not found');
}

export async function mockServerError(page: Page, url: string): Promise<void> {
  await mockApiError(page, url, 500, 'Internal server error');
}
```

### Usage in Tests

```typescript
import { test, expect } from '@playwright/test';
import { mockPatientsApi, mockApiError } from '../../utils/api-mocking';
import { authenticateAsCoach } from '../../fixtures/auth.fixture';

test.describe('Patients List', () => {
  test('should display patients from API', async ({ page }) => {
    await mockPatientsApi(page);
    await authenticateAsCoach(page);
    await page.goto('/coach/patients');

    await expect(page.getByText('이재활')).toBeVisible();
  });

  test('should handle API error gracefully', async ({ page }) => {
    await mockApiError(page, '**/api/patients*', 500, 'Server error');
    await authenticateAsCoach(page);
    await page.goto('/coach/patients');

    await expect(page.getByText(/error|오류/i)).toBeVisible();
  });
});
```

---

## Best Practices

### 1. Isolate Test Data

```typescript
// Each test should have independent data
test('should create new patient', async ({ page }) => {
  const uniqueEmail = generateTestEmail();
  // Use uniqueEmail for this test only
});
```

### 2. Clean Up After Tests

```typescript
test.afterEach(async ({ page }) => {
  await page.evaluate(() => {
    localStorage.clear();
    sessionStorage.clear();
  });
});
```

### 3. Use Realistic Mock Data

```typescript
// Good - realistic data
const mockPatient = {
  id: 'uuid-here',
  name: '이재활',
  createdAt: '2024-01-15T10:00:00Z',
};

// Bad - unrealistic data
const mockPatient = {
  id: '1',
  name: 'Test',
  createdAt: 'today',
};
```

### 4. Clear Mocks Between Tests

```typescript
import { clearMocks } from '../../utils/api-mocking';

test.afterEach(async ({ page }) => {
  await clearMocks(page);
});
```

---

## Related Files

- [e2e-test-generator.md](e2e-test-generator.md) - Test patterns
- [e2e-page-objects.md](e2e-page-objects.md) - Page object patterns
- [auth.fixture.ts](../../../frontend/test/fixtures/auth.fixture.ts) - Auth implementation
- [api-mocking.ts](../../../frontend/test/utils/api-mocking.ts) - Mocking implementation
