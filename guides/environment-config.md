# Environment Configuration Guide

Managing environment variables, build profiles, and configuration in Expo/React Native apps.

## Table of Contents

- [app.config.ts Dynamic Configuration](#appconfigts-dynamic-configuration)
- [Environment Variables](#environment-variables)
- [Multi-Environment Setup](#multi-environment-setup)
- [Runtime Configuration](#runtime-configuration)
- [Secrets Management](#secrets-management)
- [Build Variants](#build-variants)

---

## app.config.ts Dynamic Configuration

### Converting from app.json

Replace static `app.json` with dynamic `app.config.ts` to support environment-based configuration:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config';

const IS_DEV = process.env.APP_VARIANT === 'development';
const IS_STAGING = process.env.APP_VARIANT === 'staging';

function getAppName(): string {
  if (IS_DEV) return 'MyApp (Dev)';
  if (IS_STAGING) return 'MyApp (Staging)';
  return 'MyApp';
}

function getBundleId(): string {
  if (IS_DEV) return 'com.mycompany.myapp.dev';
  if (IS_STAGING) return 'com.mycompany.myapp.staging';
  return 'com.mycompany.myapp';
}

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: getAppName(),
  slug: 'myapp',
  version: '1.0.0',
  ios: {
    bundleIdentifier: getBundleId(),
    supportsTablet: true,
  },
  android: {
    package: getBundleId(),
    adaptiveIcon: {
      foregroundImage: './assets/adaptive-icon.png',
      backgroundColor: '#ffffff',
    },
  },
  extra: {
    apiUrl: process.env.EXPO_PUBLIC_API_URL,
    sentryDsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
    appVariant: process.env.APP_VARIANT ?? 'production',
    eas: {
      projectId: 'your-project-id',
    },
  },
});
```

---

## Environment Variables

### EXPO_PUBLIC_ Prefix

Variables prefixed with `EXPO_PUBLIC_` are inlined at build time and accessible in client code:

```typescript
// Accessible in your app code
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
const analyticsKey = process.env.EXPO_PUBLIC_ANALYTICS_KEY;
```

```bash
# .env.development
EXPO_PUBLIC_API_URL=https://api-dev.myapp.com
EXPO_PUBLIC_ANALYTICS_KEY=dev-key-123

# .env.staging
EXPO_PUBLIC_API_URL=https://api-staging.myapp.com
EXPO_PUBLIC_ANALYTICS_KEY=staging-key-456

# .env.production
EXPO_PUBLIC_API_URL=https://api.myapp.com
EXPO_PUBLIC_ANALYTICS_KEY=prod-key-789
```

### Private Variables

Variables without the `EXPO_PUBLIC_` prefix are only available during the build process (in `app.config.ts`, EAS Build, etc.) and are NOT bundled into the app:

```bash
# Private — only available at build time
SENTRY_AUTH_TOKEN=secret-token-here
APP_STORE_CONNECT_API_KEY=another-secret
```

### .gitignore Setup

```gitignore
# NEVER commit env files with secrets
.env
.env.local
.env.development
.env.staging
.env.production

# DO commit the example file
!.env.example
```

Create a `.env.example` for documentation:

```bash
# .env.example — copy to .env and fill in values
EXPO_PUBLIC_API_URL=https://api.example.com
EXPO_PUBLIC_ANALYTICS_KEY=your-key-here
APP_VARIANT=development
```

---

## Multi-Environment Setup

### EAS Build Profiles

```json
// eas.json
{
  "cli": {
    "version": ">= 5.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_VARIANT": "development",
        "EXPO_PUBLIC_API_URL": "https://api-dev.myapp.com"
      },
      "ios": {
        "simulator": true
      }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "APP_VARIANT": "staging",
        "EXPO_PUBLIC_API_URL": "https://api-staging.myapp.com"
      },
      "channel": "staging"
    },
    "production": {
      "env": {
        "APP_VARIANT": "production",
        "EXPO_PUBLIC_API_URL": "https://api.myapp.com"
      },
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@email.com",
        "ascAppId": "1234567890"
      },
      "android": {
        "serviceAccountKeyPath": "./google-services.json"
      }
    }
  }
}
```

### Building Per Environment

```bash
# Development build (simulators)
eas build --profile development --platform ios

# Staging build (internal testing)
eas build --profile staging --platform all

# Production build (app store)
eas build --profile production --platform all
```

### Different App Names and Icons Per Environment

```typescript
// app.config.ts
function getAppIcon(): string {
  if (IS_DEV) return './assets/icon-dev.png';
  if (IS_STAGING) return './assets/icon-staging.png';
  return './assets/icon.png';
}

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: getAppName(),
  icon: getAppIcon(),
  // Different bundle IDs allow installing all variants simultaneously
  ios: { bundleIdentifier: getBundleId() },
  android: { package: getBundleId() },
});
```

---

## Runtime Configuration

### Using expo-constants

```typescript
import Constants from 'expo-constants';

interface AppConfig {
  apiUrl: string;
  sentryDsn: string;
  appVariant: string;
}

export function getAppConfig(): AppConfig {
  const extra = Constants.expoConfig?.extra;

  return {
    apiUrl: extra?.apiUrl ?? 'https://api.myapp.com',
    sentryDsn: extra?.sentryDsn ?? '',
    appVariant: extra?.appVariant ?? 'production',
  };
}
```

### Environment-Based API Client

```typescript
// services/httpService.ts
import axios from 'axios';
import { getAppConfig } from '@/config';

const { apiUrl } = getAppConfig();

export const httpService = axios.create({
  baseURL: apiUrl,
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
  },
});
```

### Feature Flags

```typescript
// config/featureFlags.ts
import { getAppConfig } from '@/config';

interface FeatureFlags {
  enableNewCheckout: boolean;
  enableBiometricAuth: boolean;
  enablePushNotifications: boolean;
}

export function getFeatureFlags(): FeatureFlags {
  const { appVariant } = getAppConfig();

  return {
    // Enable experimental features in dev/staging only
    enableNewCheckout: appVariant !== 'production',
    enableBiometricAuth: true,
    enablePushNotifications: appVariant === 'production',
  };
}
```

```tsx
// Usage in components
import { getFeatureFlags } from '@/config/featureFlags';

function CheckoutScreen() {
  const { enableNewCheckout } = getFeatureFlags();

  if (enableNewCheckout) {
    return <NewCheckoutFlow />;
  }

  return <LegacyCheckoutFlow />;
}
```

---

## Secrets Management

### EAS Secrets for CI/CD

Store sensitive values as EAS Secrets — they are injected during EAS Build and never stored in your repository:

```bash
# Set secrets via CLI
eas secret:create --scope project --name SENTRY_AUTH_TOKEN --value "your-token"
eas secret:create --scope project --name GOOGLE_SERVICES_JSON --value "$(cat google-services.json)"

# List current secrets
eas secret:list

# Delete a secret
eas secret:delete --name SENTRY_AUTH_TOKEN
```

### Never Hardcode API Keys

```typescript
// GOOD: Read from environment
const config = {
  apiKey: process.env.EXPO_PUBLIC_API_KEY,
  mapsKey: process.env.EXPO_PUBLIC_MAPS_KEY,
};

// BAD: Hardcoded secrets — will be in the JS bundle
const config = {
  apiKey: 'sk-1234567890abcdef',
  mapsKey: 'AIzaSyB...',
};
```

### Secret Rotation Workflow

1. Generate new secret value
2. Update EAS Secret: `eas secret:create --scope project --name API_KEY --value "new-value" --force`
3. Trigger new build: `eas build --profile production`
4. For OTA-updateable secrets (via `extra`), push an update: `eas update --channel production`
5. Revoke old secret value on the provider side

---

## Build Variants

### iOS Schemes

Different Xcode schemes allow you to install dev, staging, and production builds side-by-side on the same device:

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  ios: {
    bundleIdentifier: getBundleId(), // Different per variant
    infoPlist: {
      CFBundleDisplayName: getAppName(),
    },
  },
});
```

### Android Build Flavors

```typescript
// app.config.ts
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  android: {
    package: getBundleId(), // Different per variant
    // Gradle properties for build variants
    ...(IS_DEV && {
      gradle: {
        buildType: 'debug',
      },
    }),
  },
});
```

### Per-Environment Splash Screen

```typescript
// app.config.ts
function getSplashConfig() {
  const backgroundColor = IS_DEV ? '#FF6B6B' : IS_STAGING ? '#FFA726' : '#FFFFFF';

  return {
    image: './assets/splash.png',
    resizeMode: 'contain' as const,
    backgroundColor,
  };
}

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  splash: getSplashConfig(),
});
```

This makes it visually obvious which environment you are running when the app launches — red for development, orange for staging, white for production.

---

## Related Resources

- [File Organization Guide](./file-organization.md) — project structure conventions
- [Data Fetching Guide](./data-fetching.md) — httpService and API layer setup
