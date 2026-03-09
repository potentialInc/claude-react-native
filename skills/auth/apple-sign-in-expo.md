---
name: apple-sign-in-expo
description: Comprehensive guide for implementing Apple Sign-In in React Native Expo apps, covering iOS (native) and Android (web-based OAuth) flows, deep link handling, cold-start race conditions, and all platform-specific gotchas.
---

# Apple Sign-In in React Native Expo (iOS + Android)

A comprehensive guide for implementing Apple Sign-In in React Native Expo apps, covering both iOS (native) and Android (web-based) flows, deep link handling, and all the race conditions you'll encounter.

---

## Architecture Overview

Apple Sign-In works fundamentally differently on each platform:

| Aspect | iOS | Android |
|--------|-----|---------|
| Auth mechanism | Native Apple Sign-In sheet (`expo-apple-authentication`) | Web-based OAuth via Chrome Custom Tab (`expo-web-browser`) |
| Token delivery | Returned synchronously from native API | Delivered via deep link after backend 302 redirect |
| Backend callback | Not needed | Required — Apple POSTs form data to your backend |
| Deep link handling | Not needed | Required — `{YOUR_SCHEME}://` custom scheme |
| Cold-start risk | None | High — app may be killed while Chrome Custom Tab is open |

### Flow Diagrams

**iOS (Simple)**:
```
User taps "Sign in with Apple"
  → Native Apple Sign-In sheet appears
  → User authenticates with Face ID / password
  → expo-apple-authentication returns { identityToken, nonce, email, fullName }
  → Frontend calls backend /auth/apple-login with token
  → Backend validates token with Apple, returns session
  → Done
```

**Android (Complex)**:
```
User taps "Sign in with Apple"
  → openAuthSessionAsync opens Chrome Custom Tab
  → User authenticates on appleid.apple.com
  → Apple POSTs form_post to YOUR backend /auth/apple-callback
  → Backend extracts id_token, sends HTTP 302 redirect to {YOUR_SCHEME}://auth/apple-callback?id_token=...
  → Android intercepts {YOUR_SCHEME}:// scheme
  → Deep link arrives at app via one of two paths:
    a) WARM START: Linking.addEventListener captures URL → socialAuthService resolves Promise
    b) COLD START: App relaunches → +native-intent.tsx rewrites path → apple-callback.tsx processes login
```

---

## Dependencies

```json
{
  "expo-apple-authentication": "~7.2.2",
  "expo-web-browser": "~14.0.2",
  "expo-crypto": "~14.1.0",
  "react-native-mmkv": "^3.2.0"
}
```

**app.json plugins**:
```json
["expo-apple-authentication"]
```

**app.json iOS config**:
```json
{
  "ios": {
    "usesAppleSignIn": true,
    "bundleIdentifier": "{YOUR_BUNDLE_ID}"
  }
}
```

**app.json Android config** (deep link support):
```json
{
  "android": {
    "intentFilters": [
      {
        "action": "VIEW",
        "data": [{ "scheme": "{YOUR_SCHEME}" }],
        "category": ["DEFAULT", "BROWSABLE"]
      }
    ]
  }
}
```

---

## Apple Developer Console Setup

### 1. App ID (iOS)
- Enable "Sign in with Apple" capability on your App ID
- Bundle ID must match `ios.bundleIdentifier` in app.json

### 2. Services ID (Android web flow)
- Create a **Services ID** (NOT the App ID) — e.g., `{YOUR_BUNDLE_ID}.service`
- Enable "Sign in with Apple"
- Configure **Domains and Subdomains**: your API domain (e.g., `{YOUR_API_DOMAIN}`)
- Configure **Return URLs**: your backend callback URL (e.g., `https://{YOUR_API_DOMAIN}/auth/apple-callback`)

> **Critical**: The `client_id` in the OAuth request must be the **Services ID**, not the App/Bundle ID.

---

## Backend Implementation

### Apple Callback Endpoint (POST)

Apple sends a `form_post` with `id_token`, `state`, `user`, and `code` to your callback URL. Your backend must:

1. Extract the parameters from the POST body
2. Redirect to your app's custom URL scheme with the tokens as query params

```typescript
// NestJS example
@Post('apple-callback')
@HttpCode(302)
appleCallback(
    @Body() body: Record<string, string>,
    @Res() res: Response,
): void {
    const { id_token, state, user, code } = body;

    const params = new URLSearchParams();
    if (id_token) params.set('id_token', id_token);
    if (state) params.set('nonce', state);
    if (user) params.set('user', user);
    if (code) params.set('code', code);

    const redirectUrl = `{YOUR_SCHEME}://auth/apple-callback?${params.toString()}`;
    res.redirect(302, redirectUrl);
}
```

> **Why HTTP 302?** Chrome Custom Tab on Android follows HTTP 302 redirects. When the redirect target is a custom URL scheme (`{YOUR_SCHEME}://`), Android's intent system intercepts it and delivers it to your app as a deep link. JavaScript-based redirects (`window.location.href`) do NOT work reliably.

### Apple Login Endpoint (POST)

Separate endpoint that validates the Apple `id_token` with Apple's JWKS and creates/returns a session:

```typescript
@Post('apple-login')
async appleLogin(@Body() dto: AppleLoginDto) {
    // 1. Verify id_token signature against Apple's JWKS
    // 2. Extract email/sub from token claims
    // 3. Find or create user
    // 4. Generate session token
    // 5. Return { user, token }
}
```

---

## Frontend Implementation

### File Structure

```
src/
├── services/
│   └── socialAuthService.ts      # Platform-specific Apple Sign-In logic
├── hooks/
│   └── useSocialAuth.ts          # React hook wrapping socialAuthService
├── app/
│   ├── +native-intent.tsx        # Deep link routing (cold-start path rewrite + MMKV flag)
│   ├── _layout.tsx               # Root layout (cold-start routing guard)
│   └── auth/
│       └── apple-callback.tsx    # Cold-start fallback screen
└── services/
    └── httpService.ts            # Axios with 401 interceptor (MMKV guard)
```

### socialAuthService.ts — Core Auth Logic

```typescript
import * as AppleAuthentication from 'expo-apple-authentication';
import * as Crypto from 'expo-crypto';
import * as WebBrowser from 'expo-web-browser';
import { Linking, Platform } from 'react-native';

// CUSTOMIZE: Replace with your Apple Services ID and callback URL
const APPLE_SERVICE_ID = '{YOUR_SERVICES_ID}'; // Services ID, NOT App ID
const APPLE_CALLBACK_URL = 'https://{YOUR_API_DOMAIN}/auth/apple-callback';

interface AppleAuthResult {
    identityToken: string;
    nonce: string;
    email: string | null;
    fullName: string | null;
    userPayload: string | undefined;
    authorizationCode: string | null;
}

class SocialAuthService {
    /**
     * Dispatches to native (iOS) or web (Android) flow
     */
    async signInWithApple(options?: {
        termsAccepted?: boolean;
        forceLogin?: boolean;
    }): Promise<AppleAuthResult> {
        if (Platform.OS === 'ios') {
            return this.signInWithAppleNative();
        }
        return this.signInWithAppleWeb(options);
    }

    /**
     * iOS: Native Apple Sign-In via expo-apple-authentication
     */
    private async signInWithAppleNative(): Promise<AppleAuthResult> {
        const isAvailable = await AppleAuthentication.isAvailableAsync();
        if (!isAvailable) throw new Error('Apple Sign-In not available');

        const rawNonce = this.generateNonce();
        const hashedNonce = await Crypto.digestStringAsync(
            Crypto.CryptoDigestAlgorithm.SHA256,
            rawNonce
        );

        const credential = await AppleAuthentication.signInAsync({
            requestedScopes: [
                AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
                AppleAuthentication.AppleAuthenticationScope.EMAIL,
            ],
            nonce: hashedNonce,
        });

        if (!credential.identityToken) {
            throw new Error('Failed to get Apple identity token');
        }

        // User info (name, email) is only provided on FIRST authorization.
        // Subsequent sign-ins return null for these fields.
        let userPayload: string | undefined;
        if (credential.fullName?.givenName || credential.email) {
            userPayload = JSON.stringify({
                email: credential.email,
                name: {
                    firstName: credential.fullName?.givenName,
                    lastName: credential.fullName?.familyName,
                },
            });
        }

        const fullName = [credential.fullName?.givenName, credential.fullName?.familyName]
            .filter(Boolean)
            .join(' ') || null;

        return {
            identityToken: credential.identityToken,
            nonce: hashedNonce, // Send HASHED nonce — it must match the JWT's nonce claim
            email: credential.email,
            fullName,
            userPayload,
            authorizationCode: credential.authorizationCode || null,
        };
    }

    /**
     * Android: Web-based Apple Sign-In via Chrome Custom Tab
     *
     * DUAL-MECHANISM CAPTURE:
     * 1. Linking.addEventListener — captures deep link via Android intent system (PRIMARY)
     * 2. openAuthSessionAsync — captures HTTP 302 redirect directly (SECONDARY, some devices)
     * Whichever fires first wins via `settled` guard.
     */
    private async signInWithAppleWeb(
        _options?: { termsAccepted?: boolean; forceLogin?: boolean }
    ): Promise<AppleAuthResult> {
        const rawNonce = this.generateNonce();
        const hashedNonce = await Crypto.digestStringAsync(
            Crypto.CryptoDigestAlgorithm.SHA256,
            rawNonce
        );

        const params = new URLSearchParams({
            client_id: APPLE_SERVICE_ID,
            redirect_uri: APPLE_CALLBACK_URL,
            response_type: 'code id_token',
            response_mode: 'form_post',
            scope: 'name email',
            nonce: hashedNonce,
            state: hashedNonce,
        });

        const authUrl = `https://appleid.apple.com/auth/authorize?${params.toString()}`;
        // CUSTOMIZE: Replace with your app's custom URL scheme
        const callbackSchemeUrl = '{YOUR_SCHEME}://auth/apple-callback';

        return new Promise<AppleAuthResult>((resolve, reject) => {
            let settled = false;

            const settle = (fn: () => void) => {
                if (settled) return;
                settled = true;
                subscription.remove();
                fn();
            };

            // PRIMARY: Linking.addEventListener captures deep link
            const subscription = Linking.addEventListener('url', ({ url }) => {
                if (!url.includes('apple-callback')) return;

                settle(() => {
                    // dismissAuthSession is iOS-only; on Android the Chrome Custom Tab
                    // closes automatically when the deep link intent fires
                    try { WebBrowser.dismissAuthSession(); } catch {}
                    const callbackParams = this.parseAppleCallbackParams(url);
                    const authResult = this.buildAppleAuthResult(callbackParams, hashedNonce);
                    if (authResult) resolve(authResult);
                    else reject(new Error('No id_token in Apple callback'));
                });
            });

            // SECONDARY: openAuthSessionAsync captures HTTP 302 redirect
            WebBrowser.openAuthSessionAsync(authUrl, callbackSchemeUrl)
                .then((result) => {
                    if (result.type === 'success' && result.url) {
                        settle(() => {
                            const callbackParams = this.parseAppleCallbackParams(result.url);
                            const authResult = this.buildAppleAuthResult(callbackParams, hashedNonce);
                            if (authResult) resolve(authResult);
                            else reject(new Error('No id_token in Apple callback'));
                        });
                    } else if (!settled) {
                        settle(() => reject(new Error('Sign in cancelled')));
                    }
                })
                .catch((err) => settle(() => reject(err)));
        });
    }

    private parseAppleCallbackParams(url: string) {
        const queryString = url.split('?')[1] || '';
        const urlParams = new URLSearchParams(queryString);
        return {
            id_token: urlParams.get('id_token'),
            nonce: urlParams.get('nonce'),
            code: urlParams.get('code'),
            user: urlParams.get('user'),
        };
    }

    private buildAppleAuthResult(
        params: { id_token: string | null; nonce: string | null; code: string | null; user: string | null },
        fallbackNonce: string,
    ): AppleAuthResult | null {
        if (!params.id_token) return null;

        let email: string | null = null;
        let fullName: string | null = null;
        if (params.user) {
            try {
                const userData = JSON.parse(params.user);
                email = userData.email || null;
                const firstName = userData.name?.firstName || '';
                const lastName = userData.name?.lastName || '';
                fullName = [firstName, lastName].filter(Boolean).join(' ') || null;
            } catch {}
        }

        return {
            identityToken: params.id_token,
            nonce: params.nonce || fallbackNonce,
            email,
            fullName,
            userPayload: params.user || undefined,
            authorizationCode: params.code || null,
        };
    }

    private generateNonce(length = 32): string {
        const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
        let result = '';
        const randomValues = new Uint8Array(length);
        if (typeof crypto !== 'undefined' && crypto.getRandomValues) {
            crypto.getRandomValues(randomValues);
        } else {
            for (let i = 0; i < length; i++) randomValues[i] = Math.floor(Math.random() * 256);
        }
        for (let i = 0; i < length; i++) result += charset[randomValues[i] % charset.length];
        return result;
    }
}
```

---

## Deep Link Handling — The Hard Part

### The Problem

When Chrome Custom Tab redirects to `{YOUR_SCHEME}://auth/apple-callback?id_token=...`:

1. **Warm start** (app alive in background): `Linking.addEventListener` fires → Promise resolves → done
2. **Cold start** (app killed by Android): App relaunches from scratch → original Promise is dead → need a fallback screen

### +native-intent.tsx — Synchronous Route Rewriting

Expo Router's `+native-intent.tsx` runs **synchronously before any React components mount**. This is the earliest possible interception point.

```typescript
import { MMKV } from 'react-native-mmkv';

const mmkv = new MMKV();

export function redirectSystemPath({ path, initial }: { path: string; initial: boolean }): string {
    // WARM START: suppress Expo Router navigation — Linking.addEventListener handles it
    if (!initial && path.includes('apple-callback')) {
        return '/';
    }

    // COLD START: set MMKV flag + fix path
    if (initial && path.includes('apple-callback')) {
        // Set flag BEFORE any components mount — 401 interceptor checks this
        mmkv.set('apple_callback_in_progress', true);

        // Fix URL parsing: {YOUR_SCHEME}://auth/apple-callback parses "auth" as hostname
        // → path arrives as "/apple-callback" instead of "/auth/apple-callback"
        if (!path.includes('auth/apple-callback')) {
            const queryIndex = path.indexOf('?');
            const queryString = queryIndex >= 0 ? path.substring(queryIndex) : '';
            return `/auth/apple-callback${queryString}`;
        }
    }

    return path;
}
```

> **Why the path rewrite?** Custom URL schemes (`{YOUR_SCHEME}://auth/apple-callback`) are parsed per URL spec where `auth` is the **hostname**, not a path segment. So the path arrives as `/apple-callback`, not `/auth/apple-callback`. Expo Router can't match the file route without the rewrite.

### apple-callback.tsx — Cold-Start Fallback Screen

This screen only runs on cold start. It:
1. Extracts tokens from URL params (with `Linking.getInitialURL()` fallback)
2. Clears stale auth state
3. Calls the backend login endpoint
4. Dispatches login on success

```typescript
export default function AppleCallbackScreen() {
    const params = useLocalSearchParams<{ id_token?: string; nonce?: string; code?: string; user?: string }>();
    const dispatch = useDispatch();
    const hasProcessed = useRef(false);

    // Safety cleanup: clear MMKV flag on unmount
    useEffect(() => {
        return () => { mmkv.delete('apple_callback_in_progress'); };
    }, []);

    useEffect(() => {
        if (hasProcessed.current) return;
        hasProcessed.current = true;
        processAppleCallback();
    }, []);

    const processAppleCallback = async () => {
        let { id_token, nonce, code, user } = params;

        // Fallback: Expo Router may drop query params on custom schemes
        if (!id_token) {
            const initialUrl = await Linking.getInitialURL();
            if (initialUrl?.includes('apple-callback')) {
                const urlParams = new URLSearchParams(initialUrl.split('?')[1] || '');
                id_token = urlParams.get('id_token') || undefined;
                // ... parse other params
            }
        }

        if (!id_token) {
            mmkv.delete('apple_callback_in_progress');
            router.replace('/(auth)/signin');
            return;
        }

        // CRITICAL: Clear stale Redux Persist state before new login
        dispatch(logout());

        // CUSTOMIZE: Replace with your auth service call
        const response = await authService.appleLogin({ id_token, nonce, code, user });
        mmkv.delete('apple_callback_in_progress');
        dispatch(login({ user: response.user, token: response.token }));
        // _layout.tsx routing picks up isAuthenticated=true → navigates to home
    };

    return (
        <View style={{ flex: 1, backgroundColor: '#000', justifyContent: 'center', alignItems: 'center' }}>
            <ActivityIndicator size="large" color="{YOUR_BRAND_COLOR}" />
            <Text style={{ color: '#fff', marginTop: 16 }}>Signing in with Apple...</Text>
        </View>
    );
}
```

---

## Race Conditions & Solutions

### Race 1: Redux Persist Rehydrates Stale Token

**Problem**: On cold start, Redux Persist rehydrates `isAuthenticated=true` with an OLD token. `_layout.tsx` routing sees this and navigates to the home screen. Home screen components fire API calls with the stale token → 401 "Session invalidated" → auto-logout.

**Solution** (`_layout.tsx`): Check `Linking.getInitialURL()` on mount. If apple-callback detected, dispatch `logout()` immediately and set `isAppleCallbackColdStart` flag to defer routing.

```typescript
const [initialUrlChecked, setInitialUrlChecked] = useState(false);
const isAppleCallbackColdStart = useRef(false);

useEffect(() => {
    Linking.getInitialURL().then((url) => {
        if (url?.includes('apple-callback')) {
            isAppleCallbackColdStart.current = true;
            dispatch(logout()); // Clear stale state
        }
        setInitialUrlChecked(true);
    });
}, []);

// In routing effect:
if (!initialUrlChecked) return; // Wait for URL check
if (!isAuthenticated && isAppleCallbackColdStart.current) {
    // Don't navigate to signin — let apple-callback.tsx handle login
    return;
}
if (isAuthenticated && isAppleCallbackColdStart.current) {
    // apple-callback.tsx completed login — resume normal routing
    isAppleCallbackColdStart.current = false;
}
```

### Race 2: 401 Interceptor Kills apple-callback.tsx

**Problem**: The 401 response interceptor in `httpService.ts` calls `router.replace('/(auth)/signin')` directly, bypassing the `_layout.tsx` guard. Background components make API calls with the stale token during the ~50ms window before `Linking.getInitialURL()` resolves → 401 → interceptor navigates away → `apple-callback.tsx` unmounted.

**Timeline**:
```
t=0  Cold start → Redux Persist: isAuthenticated=true, token=<stale>
t=1  Background components mount → see isAuthenticated=true → initialize
t=2  _layout.tsx useEffect starts Linking.getInitialURL() (ASYNC)
t=3  Background component makes HTTP call with stale token → 401
t=4  httpService.ts interceptor: logout() + router.replace('/(auth)/signin') ← KILLS apple-callback
t=5  Linking.getInitialURL() resolves — TOO LATE
```

**Solution**: MMKV flag set in `+native-intent.tsx` (runs SYNCHRONOUSLY before React mounts):

```typescript
// +native-intent.tsx (synchronous — before any React component)
if (initial && path.includes('apple-callback')) {
    mmkv.set('apple_callback_in_progress', true);
}

// httpService.ts (401 interceptor)
if (error.response?.status === 401) {
    store.dispatch(logout());
    if (!mmkv.getBoolean('apple_callback_in_progress')) {
        router.replace('/(auth)/signin?sessionExpired=true');
    }
}

// apple-callback.tsx (cleanup)
// Clear on success, error, and unmount
mmkv.delete('apple_callback_in_progress');
```

### Race 3: WebBrowser.dismissAuthSession() is iOS-Only

**Problem**: `WebBrowser.dismissAuthSession()` crashes on Android with "not available on android".

**Solution**: Wrap in try-catch. On Android, Chrome Custom Tab closes automatically when the deep link intent fires.

```typescript
try { WebBrowser.dismissAuthSession(); } catch {}
```

### Race 4: openAuthSessionAsync Never Resolves on Android

**Problem**: `openAuthSessionAsync` cannot capture HTTP 302 redirects to custom URL schemes on most Android devices. It opens Chrome Custom Tab successfully but never resolves.

**Solution**: Use `Linking.addEventListener` as the PRIMARY capture mechanism. `openAuthSessionAsync` is SECONDARY (works on some devices). The `settled` flag prevents double-handling.

---

## Key Gotchas

### 1. Custom URL Scheme Parsing
`{YOUR_SCHEME}://auth/apple-callback` is parsed as:
- Scheme: `{YOUR_SCHEME}`
- Host: `auth` (NOT a path segment!)
- Path: `/apple-callback`

This means Expo Router receives path `/apple-callback`, not `/auth/apple-callback`. You MUST rewrite the path in `+native-intent.tsx`.

### 2. Apple Only Provides User Info Once
Apple only sends the user's name and email on the **first** authorization. Subsequent sign-ins return `null`. Your backend must save this info on first login.

### 3. Nonce Must Be Hashed
The nonce in the Apple Sign-In request must be SHA256 hashed. The JWT's `nonce` claim contains the hash, so you must send the **hashed** nonce to your backend for verification.

### 4. response_mode: 'form_post'
Apple's OAuth uses `form_post` response mode — it sends a POST request to your callback URL, NOT a redirect with query params. Your backend callback must handle POST, not GET.

### 5. Services ID vs App ID
The `client_id` for web-based Apple Sign-In (Android) must be a **Services ID** registered in Apple Developer Console, NOT your App/Bundle ID. This is a separate identifier.

### 6. Metro Logs Disconnect During Chrome Custom Tab
React Native Metro/Logcat may disconnect while Chrome Custom Tab is in the foreground. This is a dev tooling issue, not an app crash. The app may or may not be alive in the background — you MUST handle both warm-start and cold-start paths.

### 7. MMKV for Cross-Component Coordination
Use MMKV (synchronous key-value storage) for flags that need to be checked across different parts of the app (e.g., `+native-intent.tsx` → `httpService.ts` → `apple-callback.tsx`). Redux is async (React re-renders) and not suitable for this.

### 8. Single Session Enforcement
If your backend enforces single sessions (one `activeSessionId` per user), a new login invalidates all previous tokens immediately. This means ANY stale token usage after a new login will fail with 401. Clear stale tokens BEFORE the new login, not after.

---

## Complete File Reference

| File | Role |
|------|------|
| `services/socialAuthService.ts` | Platform dispatch (iOS native / Android web), dual-mechanism deep link capture |
| `hooks/useSocialAuth.ts` | React hook wrapping socialAuthService, handles termsAccepted/forceLogin |
| `app/+native-intent.tsx` | Synchronous route rewriting, MMKV flag for cold-start detection |
| `app/_layout.tsx` | Cold-start routing guard (Linking.getInitialURL + isAppleCallbackColdStart) |
| `app/auth/apple-callback.tsx` | Cold-start fallback screen (full login flow) |
| `services/httpService.ts` | 401 interceptor with MMKV guard |
| Backend `auth.controller.ts` | Apple callback endpoint (HTTP 302 redirect) |
| Backend `auth.service.ts` | Apple login endpoint (token verification, session creation) |
| `app.json` | `usesAppleSignIn`, `intentFilters`, `scheme` config |

---

## Testing Checklist

- [ ] **iOS**: Native Apple Sign-In → login succeeds → home screen
- [ ] **Android warm-start**: Chrome Custom Tab → Linking.addEventListener captures deep link → login succeeds
- [ ] **Android cold-start**: Kill app → Chrome Custom Tab → deep link relaunches app → apple-callback.tsx processes login → home screen
- [ ] **Android cancel**: Close Chrome Custom Tab → back on signin screen (no crash)
- [ ] **Already logged in on other device**: 409 error → confirmation dialog → force login works
- [ ] **Terms acceptance**: New user → redirect to terms page → accept → retry login
- [ ] **No 401 errors**: Backend logs show clean login without "Session invalidated" errors
- [ ] **MMKV cleanup**: `apple_callback_in_progress` flag is deleted after login (success or error)
