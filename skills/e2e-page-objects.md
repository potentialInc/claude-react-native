# E2E Page Objects for React Native

Page Object Model patterns for mobile E2E testing with Detox and Maestro.

---

## Overview

Page Objects encapsulate screen interactions, making tests more maintainable and readable.

---

## Detox Page Objects

### Base Screen Class

```typescript
// e2e/screens/BaseScreen.ts
import { by, element, expect, waitFor } from 'detox';

export abstract class BaseScreen {
  protected abstract screenId: string;

  async isVisible(): Promise<void> {
    await expect(element(by.id(this.screenId))).toBeVisible();
  }

  async waitForScreen(timeout = 5000): Promise<void> {
    await waitFor(element(by.id(this.screenId)))
      .toBeVisible()
      .withTimeout(timeout);
  }

  protected async tapElement(testId: string): Promise<void> {
    await element(by.id(testId)).tap();
  }

  protected async typeText(testId: string, text: string): Promise<void> {
    await element(by.id(testId)).typeText(text);
  }

  protected async clearAndType(testId: string, text: string): Promise<void> {
    await element(by.id(testId)).clearText();
    await element(by.id(testId)).typeText(text);
  }

  protected async scrollTo(testId: string, direction: 'up' | 'down' = 'down'): Promise<void> {
    await waitFor(element(by.id(testId)))
      .toBeVisible()
      .whileElement(by.id(this.screenId))
      .scroll(200, direction);
  }
}
```

### Login Screen Example

```typescript
// e2e/screens/LoginScreen.ts
import { by, element, expect } from 'detox';
import { BaseScreen } from './BaseScreen';

export class LoginScreen extends BaseScreen {
  protected screenId = 'login-screen';

  // Element locators
  private emailInput = by.id('email-input');
  private passwordInput = by.id('password-input');
  private loginButton = by.id('login-button');
  private errorMessage = by.id('error-message');
  private forgotPasswordLink = by.id('forgot-password');

  async enterEmail(email: string): Promise<void> {
    await element(this.emailInput).clearText();
    await element(this.emailInput).typeText(email);
  }

  async enterPassword(password: string): Promise<void> {
    await element(this.passwordInput).clearText();
    await element(this.passwordInput).typeText(password);
  }

  async tapLoginButton(): Promise<void> {
    await element(this.loginButton).tap();
  }

  async login(email: string, password: string): Promise<void> {
    await this.enterEmail(email);
    await this.enterPassword(password);
    await this.tapLoginButton();
  }

  async expectErrorMessage(message: string): Promise<void> {
    await expect(element(this.errorMessage)).toHaveText(message);
  }

  async tapForgotPassword(): Promise<void> {
    await element(this.forgotPasswordLink).tap();
  }
}
```

### Home Screen Example

```typescript
// e2e/screens/HomeScreen.ts
import { by, element, expect, waitFor } from 'detox';
import { BaseScreen } from './BaseScreen';

export class HomeScreen extends BaseScreen {
  protected screenId = 'home-screen';

  private welcomeText = by.id('welcome-text');
  private profileButton = by.id('profile-button');
  private settingsButton = by.id('settings-button');
  private itemList = by.id('item-list');

  async expectWelcomeMessage(name: string): Promise<void> {
    await expect(element(this.welcomeText)).toHaveText(`Welcome, ${name}`);
  }

  async tapProfile(): Promise<void> {
    await element(this.profileButton).tap();
  }

  async tapSettings(): Promise<void> {
    await element(this.settingsButton).tap();
  }

  async scrollToItem(itemId: string): Promise<void> {
    await waitFor(element(by.id(itemId)))
      .toBeVisible()
      .whileElement(this.itemList)
      .scroll(100, 'down');
  }

  async tapItem(itemId: string): Promise<void> {
    await element(by.id(itemId)).tap();
  }
}
```

---

## Detox Element Matchers

### By ID (Recommended)

```typescript
// Set testID in component
<View testID="my-component" />

// Match in test
element(by.id('my-component'))
```

### By Text

```typescript
// Exact match
element(by.text('Submit'))

// Partial match
element(by.text('Submit').withAncestor(by.id('form')))
```

### By Label (Accessibility)

```typescript
// Set accessibilityLabel
<Pressable accessibilityLabel="Close button" />

// Match in test
element(by.label('Close button'))
```

### Combining Matchers

```typescript
// Element with ID inside ancestor
element(by.id('submit-btn').withAncestor(by.id('login-form')))

// Element with ID containing descendant
element(by.id('list').withDescendant(by.text('Item 1')))

// Multiple conditions
element(by.id('button').and(by.label('Submit')))
```

---

## Detox Actions

### Tap Actions

```typescript
await element(by.id('button')).tap();
await element(by.id('button')).longPress();
await element(by.id('button')).longPress(2000); // 2 seconds
await element(by.id('button')).multiTap(2); // Double tap
```

### Text Input

```typescript
await element(by.id('input')).typeText('Hello');
await element(by.id('input')).clearText();
await element(by.id('input')).replaceText('New text');
await element(by.id('input')).tapReturnKey();
```

### Scroll Actions

```typescript
// Scroll within element
await element(by.id('scroll-view')).scroll(200, 'down');
await element(by.id('scroll-view')).scroll(200, 'up');
await element(by.id('scroll-view')).scrollTo('bottom');
await element(by.id('scroll-view')).scrollTo('top');

// Scroll until visible
await waitFor(element(by.id('target')))
  .toBeVisible()
  .whileElement(by.id('scroll-view'))
  .scroll(100, 'down');
```

### Swipe Actions

```typescript
await element(by.id('card')).swipe('left');
await element(by.id('card')).swipe('right');
await element(by.id('card')).swipe('up', 'fast');
await element(by.id('card')).swipe('down', 'slow', 0.5); // 50% of element
```

---

## Maestro Page Objects (YAML)

### Reusable Flows

```yaml
# e2e/flows/common/login.yaml
appId: ${APP_ID}

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
```

### Using Reusable Flows

```yaml
# e2e/flows/user-journey.yaml
appId: com.yourapp

- launchApp

# Login with credentials
- runFlow:
    file: common/login.yaml
    env:
      EMAIL: "test@example.com"
      PASSWORD: "password123"

# Continue with test
- tapOn:
    id: "profile-button"
```

### Maestro Selectors

```yaml
# By testID (recommended)
- tapOn:
    id: "submit-button"

# By text
- tapOn:
    text: "Submit"

# By accessibility label
- tapOn:
    label: "Close"

# By index (when multiple matches)
- tapOn:
    text: "Item"
    index: 0

# Containing text
- tapOn:
    text: ".*Submit.*"  # regex
```

### Maestro Actions

```yaml
# Tap
- tapOn:
    id: "button"

# Long press
- longPressOn:
    id: "item"

# Input text
- inputText: "Hello World"

# Clear and input
- clearText:
    id: "input"
- inputText: "New text"

# Scroll
- scroll
- scrollUntilVisible:
    element:
      id: "target-item"

# Swipe
- swipe:
    direction: LEFT
    duration: 500

# Wait
- waitForAnimationToEnd
- extendedWaitUntil:
    visible:
      id: "loading-complete"
    timeout: 10000
```

---

## Test Helpers

### Device Helpers (Detox)

```typescript
// e2e/helpers/device.ts
import { device } from 'detox';

export const DeviceHelpers = {
  async reloadApp(): Promise<void> {
    await device.reloadReactNative();
  },

  async reinstallApp(): Promise<void> {
    await device.uninstallApp();
    await device.installApp();
    await device.launchApp({ newInstance: true });
  },

  async setLocation(lat: number, lon: number): Promise<void> {
    await device.setLocation(lat, lon);
  },

  async sendNotification(payload: object): Promise<void> {
    await device.sendUserNotification(payload);
  },

  async openURL(url: string): Promise<void> {
    await device.openURL({ url });
  },

  async shake(): Promise<void> {
    await device.shake();
  },
};
```

### Wait Helpers

```typescript
// e2e/helpers/wait.ts
import { waitFor, element, by } from 'detox';

export const WaitHelpers = {
  async waitForElement(testId: string, timeout = 5000): Promise<void> {
    await waitFor(element(by.id(testId)))
      .toBeVisible()
      .withTimeout(timeout);
  },

  async waitForText(text: string, timeout = 5000): Promise<void> {
    await waitFor(element(by.text(text)))
      .toBeVisible()
      .withTimeout(timeout);
  },

  async waitForNotVisible(testId: string, timeout = 5000): Promise<void> {
    await waitFor(element(by.id(testId)))
      .not.toBeVisible()
      .withTimeout(timeout);
  },
};
```

---

## Setting Up testID Props

### React Native Components

```tsx
// Always add testID for E2E testing
<View testID="home-screen">
  <Text testID="welcome-text">Welcome</Text>

  <TextInput
    testID="email-input"
    placeholder="Email"
  />

  <Pressable testID="submit-button">
    <Text>Submit</Text>
  </Pressable>
</View>
```

### Naming Convention

```
Pattern: {screen}-{element}-{type}

Examples:
- login-email-input
- login-password-input
- login-submit-button
- home-welcome-text
- profile-avatar-image
- settings-notifications-toggle
```

---

## Best Practices

### 1. Use testID Consistently
```tsx
// Every interactive element needs testID
<Pressable testID="action-button" />
<TextInput testID="form-input" />
<ScrollView testID="content-scroll" />
```

### 2. Keep Page Objects Focused
```typescript
// One screen = one page object
// Don't mix multiple screens in one class
```

### 3. Use Descriptive Method Names
```typescript
// Good
async submitLoginForm()
async expectErrorMessageVisible()

// Bad
async click()
async check()
```

### 4. Handle Async Operations
```typescript
// Always wait for elements
await waitFor(element(by.id('content')))
  .toBeVisible()
  .withTimeout(5000);
```

### 5. Avoid Hardcoded Waits
```typescript
// Bad
await new Promise(r => setTimeout(r, 2000));

// Good
await waitFor(element(by.id('loaded')))
  .toBeVisible()
  .withTimeout(5000);
```

---

## Related Files

- [e2e-test-generator.md](e2e-test-generator.md) - Mobile test generation
- [e2e-fixtures.md](e2e-fixtures.md) - Mobile test fixtures
