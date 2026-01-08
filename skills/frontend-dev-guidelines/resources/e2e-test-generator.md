# E2E Test Generator Guide

Complete guide for generating Playwright-based end-to-end tests for React frontend pages.

## Table of Contents

- [When to Generate Tests](#when-to-generate-tests)
- [Test Infrastructure](#test-infrastructure)
- [Quick Start - Generate Tests](#quick-start---generate-tests)
- [Test Patterns by Page Type](#test-patterns-by-page-type)
- [Route Analysis Logic](#route-analysis-logic)
- [Complete Examples](#complete-examples)
- [Test Checklist](#test-checklist)

---

## When to Generate Tests

This guide activates automatically when:

- Creating a new page component
- Modifying existing page functionality
- Adding form validation or API integration
- You ask to "generate e2e tests" or "create frontend tests"
- After finishing a page implementation

**Manual invocation:** Ask Claude to "generate e2e tests for [page]"

---

## Test Infrastructure

### Directory Structure

```
frontend/
├── playwright.config.ts              # Playwright configuration (root level)
└── test/
    ├── tests/                        # Test files by page
    │   ├── auth/
    │   │   ├── login.spec.ts
    │   │   ├── patient-signup.spec.ts
    │   │   └── forgot-password.spec.ts
    │   ├── coach/
    │   │   ├── home.spec.ts
    │   │   ├── patients.spec.ts
    │   │   └── patient-detail.spec.ts
    │   └── patient/
    │       ├── home.spec.ts
    │       ├── exercise.spec.ts
    │       └── exercise-detail.spec.ts
    ├── pages/                        # Page Object Models
    │   ├── base.page.ts              # Base page class
    │   ├── auth/
    │   ├── coach/
    │   └── patient/
    ├── fixtures/                     # Test fixtures
    │   ├── auth.fixture.ts           # Authentication helpers
    │   └── user.fixture.ts           # Test data
    └── utils/                        # Test utilities
        ├── test-helpers.ts           # Common helpers
        └── api-mocking.ts            # API mocking
```

### Prerequisites for Running E2E Tests

**IMPORTANT:** E2E tests run against the **real backend**, not mocked APIs. You must have both servers running before executing tests.

#### 1. Start the Backend Server

```bash
# Terminal 1: Start the backend (from project root)
cd backend
npm run start:dev
# Backend runs on http://localhost:3000
```

Verify backend is running:
```bash
curl http://localhost:3000/api/auth/login -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"patient","password":"12341234"}'
# Should return: {"success":true,...}
```

#### 2. Start the Frontend Dev Server

```bash
# Terminal 2: Start the frontend
cd frontend
npm run dev
# Frontend runs on http://localhost:5173
```

#### 3. Ensure Test Users Exist in Database

The following test accounts must exist in the backend database:

| Role    | Username | Password   | Email                    |
|---------|----------|------------|--------------------------|
| Patient | patient  | 12341234   | patient@potentialai.com  |
| Coach   | coach    | 12341234   | coach@potentialai.com    |
| Admin   | admin    | 12341234   | admin@potentialai.com    |

### Running Tests

```bash
# Terminal 3: Run tests (from frontend directory)
cd frontend

# Install Playwright browsers (first time only)
npx playwright install

# Run all E2E tests
npm run test:e2e

# Run patient tests only
npm run test:e2e -- test/tests/patient/

# Run coach tests only
npm run test:e2e -- test/tests/coach/

# Run tests with UI mode (debugging)
npm run test:e2e:ui

# Run tests in headed browser
npm run test:e2e:headed

# Generate tests by recording actions
npm run test:e2e:codegen

# View test report
npm run test:e2e:report

# Run specific test file
npm run test:e2e -- --grep "login"

# Run tests for a specific spec file
npx playwright test test/tests/auth/login.spec.ts

# Run tests for specific project (browser)
npx playwright test --project=chromium
```

### ⚠️ Troubleshooting: "Cannot navigate to invalid URL" Error

This error occurs when Playwright can't find the `baseURL` configuration. Common causes:

#### 1. Running from Wrong Directory (Most Common)

Make sure you're running tests from the `frontend` directory where `playwright.config.ts` is located.

```bash
# ❌ WRONG - Running from project root
cd /path/to/ActivityCoaching
npx playwright test

# ✅ CORRECT - Run from frontend directory
cd /path/to/ActivityCoaching/frontend
npm run test:e2e
```

#### 2. Port Conflict - Dev Server Using Different Port

When the default port (5173) is already in use, Vite picks the next available port (5174, 5175, etc.), but Playwright's `baseURL` is hardcoded to `http://localhost:5173`.

**Solutions:**

```bash
# Option A: Kill the process using port 5173
lsof -ti :5173 | xargs kill -9

# Option B: Use environment variable for dynamic port
# In playwright.config.ts:
baseURL: process.env.BASE_URL || 'http://localhost:5173',

# Then run with:
BASE_URL=http://localhost:5174 npm run test:e2e
```

#### 3. Dev Server Not Running

If `webServer.reuseExistingServer` is true and no server is running:

```bash
# Start dev server first in a separate terminal
npm run dev

# Then run tests
npm run test:e2e
```

#### 4. Custom Fixtures Not Inheriting Configuration

If you create custom fixtures with `browser.newPage()`, the `baseURL` is not automatically applied:

```typescript
// ❌ WRONG - New page doesn't have baseURL
const page = await browser.newPage();
await page.goto('/'); // Error: Cannot navigate to invalid URL

// ✅ CORRECT - Use the page from test context
test('example', async ({ page }) => {
  await page.goto('/'); // Works - uses baseURL from config
});
```

---

## Quick Start - Generate Tests

### Step 1: Create Page Object

```typescript
// frontend/test/pages/auth/login.page.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from '../base.page';

export class LoginPage extends BasePage {
  readonly url = '/login';

  // Locators - define all page elements
  readonly emailInput = this.page.getByPlaceholder('Enter your email');
  readonly passwordInput = this.page.getByPlaceholder('Enter your password');
  readonly signInButton = this.page.getByRole('button', { name: /sign in/i });
  readonly registerLink = this.page.getByRole('link', { name: /create.*account/i });
  readonly forgotPasswordLink = this.page.getByRole('link', { name: /forgot/i });

  constructor(page: Page) {
    super(page);
  }

  // Actions - user interactions
  async login(email: string, password: string): Promise<void> {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.signInButton.click();
  }

  // Assertions - page-specific validations
  async expectValidationError(field: 'email' | 'password'): Promise<void> {
    const input = field === 'email' ? this.emailInput : this.passwordInput;
    await expect(input.locator('..').locator('.text-destructive')).toBeVisible();
  }
}
```

### Step 2: Create Test File

```typescript
// frontend/test/tests/auth/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../../pages/auth/login.page';

test.describe('Login Page', () => {
  let loginPage: LoginPage;

  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.navigate();
  });

  test.describe('Page Load', () => {
    test('should display login form', async () => {
      await expect(loginPage.emailInput).toBeVisible();
      await expect(loginPage.passwordInput).toBeVisible();
      await expect(loginPage.signInButton).toBeVisible();
    });
  });

  test.describe('Form Validation', () => {
    test('should show error for empty email', async () => {
      await loginPage.passwordInput.fill('password123');
      await loginPage.signInButton.click();
      await loginPage.expectValidationError('email');
    });
  });

  test.describe('Successful Login', () => {
    test('should redirect to dashboard on valid credentials', async ({ page }) => {
      await loginPage.login('coach@test.com', 'CoachTest@123');
      await page.waitForURL('/coach');
    });
  });
});
```

---

## Test Patterns by Page Type

### 1. Form Pages (Login, Signup, Create Forms)

```typescript
test.describe('[FormPage] Tests', () => {
  test.describe('Page Load', () => {
    test('should display form elements', async () => {
      // Assert all form inputs are visible
      // Assert submit button is visible
      // Assert form title/heading is correct
    });
  });

  test.describe('Form Validation', () => {
    test('should show error for empty required fields', async () => {
      // Submit without filling required fields
      // Assert validation errors appear
    });

    test('should show error for invalid format', async () => {
      // Fill with invalid format (email, phone, etc.)
      // Assert format validation errors
    });

    test('should show error for password mismatch', async () => {
      // For signup forms with confirm password
    });
  });

  test.describe('Successful Submission', () => {
    test('should submit form with valid data', async () => {
      // Fill all fields with valid data
      // Submit form
      // Assert redirect or success message
    });
  });

  test.describe('Error Handling', () => {
    test('should display API error messages', async () => {
      // Mock API to return error
      // Submit form
      // Assert error message displayed
    });
  });

  test.describe('Navigation', () => {
    test('should navigate to related pages', async () => {
      // Click links (register, forgot password, etc.)
      // Assert navigation works
    });
  });
});
```

### 2. List Pages (Patients, Exercises, Chat)

```typescript
test.describe('[ListPage] Tests', () => {
  test.beforeEach(async ({ page }) => {
    // Authenticate before accessing protected pages
    await authenticateAsCoach(page);
  });

  test.describe('Page Load', () => {
    test('should display page header', async () => {
      // Assert title, count, breadcrumbs
    });

    test('should display list items', async ({ page }) => {
      // Assert list is populated
      const count = await page.locator('[data-testid="list-item"]').count();
      expect(count).toBeGreaterThan(0);
    });
  });

  test.describe('Search/Filter', () => {
    test('should filter items by search query', async () => {
      // Enter search term
      // Assert filtered results
    });

    test('should show empty state for no results', async () => {
      // Search for non-existent term
      // Assert empty state message
    });

    test('should clear search and show all items', async () => {
      // Search, then clear
      // Assert all items visible again
    });
  });

  test.describe('Navigation', () => {
    test('should navigate to detail page on item click', async ({ page }) => {
      // Click on list item
      await page.waitForURL(/\/detail\/\d+/);
    });
  });

  test.describe('Actions', () => {
    test('should handle delete action', async () => {
      // If list has delete functionality
    });

    test('should handle bulk actions', async () => {
      // If list has bulk selection
    });
  });
});
```

### 3. Detail Pages (Patient Detail, Exercise Detail)

```typescript
test.describe('[DetailPage] Tests', () => {
  test.beforeEach(async ({ page }) => {
    await authenticateAsPatient(page);
    // Navigate to specific detail page
    await page.goto('/patient/exercise/1');
  });

  test.describe('Page Load', () => {
    test('should display item details', async () => {
      // Assert title, description, metadata
    });

    test('should display related sections', async () => {
      // Assert related data sections (history, progress, etc.)
    });
  });

  test.describe('Actions', () => {
    test('should handle primary action', async () => {
      // Click main action button
      // Assert result
    });

    test('should handle edit action', async () => {
      // If editable
    });
  });

  test.describe('Navigation', () => {
    test('should navigate back to list', async ({ page }) => {
      await page.getByRole('button', { name: /back/i }).click();
      await page.waitForURL('/patient/exercise');
    });
  });
});
```

### 4. Multi-Step Flows (Patient Signup with OTP)

```typescript
test.describe('[MultiStepFlow] Tests', () => {
  test.describe('Step 1: Form Input', () => {
    test('should validate and proceed to step 2', async () => {
      // Fill step 1 fields
      // Submit
      // Assert step 2 is visible
    });
  });

  test.describe('Step 2: OTP Verification', () => {
    test('should send OTP and show input', async () => {
      // Complete step 1
      // Assert OTP input appears
      // Assert countdown timer visible
    });

    test('should verify valid OTP', async () => {
      // Enter valid OTP
      // Submit
      // Assert proceed to step 3
    });

    test('should allow OTP resend after countdown', async () => {
      // Wait for countdown
      // Assert resend button enabled
    });
  });

  test.describe('Step 3: Completion', () => {
    test('should complete registration and redirect', async () => {
      // Complete all steps
      // Assert redirect to dashboard
    });
  });

  test.describe('Error Handling', () => {
    test('should handle invalid OTP', async () => {
      // Enter invalid OTP
      // Assert error message
    });

    test('should handle expired OTP', async () => {
      // Test expired OTP scenario
    });
  });
});
```

---

## Route Analysis Logic

When generating tests, analyze the page component to determine:

### Page Type Detection

| Indicator | Page Type | Test Pattern |
|-----------|-----------|--------------|
| `useForm`, `zodResolver` | Form | Form validation tests |
| `.map()`, list rendering | List | Search, filter, item tests |
| `useParams`, `:id` route | Detail | Data display, actions |
| `useState` with steps | Multi-step | Step progression tests |
| Dashboard layout | Dashboard | Widget, navigation tests |

### Route-to-Test Mapping

```
Route                          → Test File
/login                         → tests/auth/login.spec.ts
/signup/patient                → tests/auth/signup-patient.spec.ts
/coach/patients                → tests/coach/patients.spec.ts
/coach/patient/:id             → tests/coach/patient-detail.spec.ts
/patient/exercise              → tests/patient/exercise.spec.ts
/patient/exercise/:id          → tests/patient/exercise-detail.spec.ts
```

---

## Complete Examples

### Form Page Test (Login)

See `frontend/test/tests/auth/login.spec.ts` for complete example.

### List Page Test (Patients)

```typescript
// frontend/test/tests/coach/patients.spec.ts
import { test, expect } from '@playwright/test';
import { PatientsPage } from '../../pages/coach/patients.page';
import { authenticateAsCoach } from '../../fixtures/auth.fixture';

test.describe('Patients List Page', () => {
  let patientsPage: PatientsPage;

  test.beforeEach(async ({ page }) => {
    await authenticateAsCoach(page);
    patientsPage = new PatientsPage(page);
    await patientsPage.navigate();
  });

  test('should display patient list', async () => {
    await expect(patientsPage.title).toBeVisible();
    const count = await patientsPage.getPatientCount();
    expect(count).toBeGreaterThan(0);
  });

  test('should filter patients by name', async () => {
    await patientsPage.searchPatient('이재활');
    await patientsPage.expectPatientVisible('이재활');
  });

  test('should navigate to patient detail', async ({ page }) => {
    await patientsPage.clickPatient('이재활');
    await page.waitForURL(/\/coach\/patient\/\d+/);
  });
});
```

---

## Test Checklist

### For Every New Page

- [ ] **Page Load Tests**
    - [ ] All main elements visible
    - [ ] Correct title/heading
    - [ ] Loading state handled

- [ ] **Authentication Tests** (if protected)
    - [ ] Redirects to login when not authenticated
    - [ ] Shows content when authenticated
    - [ ] Respects role-based access

- [ ] **Form Validation Tests** (if form page)
    - [ ] Required field validation
    - [ ] Format validation (email, phone, etc.)
    - [ ] Submit button states

- [ ] **API Integration Tests**
    - [ ] Data loads correctly
    - [ ] Loading states shown
    - [ ] Error states handled

- [ ] **Navigation Tests**
    - [ ] Links work correctly
    - [ ] Back button works
    - [ ] Breadcrumbs navigate correctly

- [ ] **Mobile Tests**
    - [ ] Layout works on mobile viewport
    - [ ] Touch interactions work
    - [ ] Bottom navigation works

---

## Related Files

- [e2e-page-objects.md](e2e-page-objects.md) - Page Object pattern details
- [e2e-fixtures.md](e2e-fixtures.md) - Test data management
- [browser-testing.md](browser-testing.md) - Manual browser testing
