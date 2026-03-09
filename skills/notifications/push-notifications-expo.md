---
name: push-notifications-expo
description: Comprehensive guide for implementing FCM push notifications + expo-notifications local notifications in React Native Expo (bare workflow / custom dev client). Covers hybrid architecture, iOS-specific setup, Android channels, foreground display, background handling, and all common pitfalls.
---

# React Native Expo — Push & Local Notifications Skill Guide

> Reusable implementation reference for FCM push notifications + expo-notifications local notifications in React Native Expo (bare workflow / custom dev client).

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites & Installation](#2-prerequisites--installation)
3. [iOS-Specific Setup](#3-ios-specific-setup)
4. [Type Definitions](#4-type-definitions)
5. [FCM Service](#5-fcm-service)
6. [Local Notification Service](#6-local-notification-service)
7. [React Hooks](#7-react-hooks)
8. [Provider & UI Components](#8-provider--ui-components)
9. [Entry Point (Background Handler)](#9-entry-point-background-handler)
10. [Backend Requirements](#10-backend-requirements)
11. [Common Pitfalls & Solutions](#11-common-pitfalls--solutions)
12. [Testing Checklist](#12-testing-checklist)

---

## 1. Architecture Overview

### Hybrid Approach

| Library | Role |
|---------|------|
| `@react-native-firebase/messaging` | Remote push notification delivery, FCM token management, background/quit-state handling |
| `expo-notifications` | Local notifications, foreground system display, scheduling, Android channels |

### Why Both?

- **Firebase** handles the push delivery pipeline (FCM → APNs on iOS, FCM on Android). It manages tokens, background messages, and notification-opened events.
- **expo-notifications** handles everything local: posting notifications to the system tray when the app is foregrounded, scheduling future notifications, and managing Android notification channels.
- They **do not conflict** — they serve different purposes and share the same OS permission system.

### Notification Flow by App State

```
┌─────────────────────────────────────────────────────────────┐
│ BACKGROUND / KILLED                                         │
│ FCM delivers → OS displays notification automatically       │
│ User taps → messaging().onNotificationOpenedApp() fires     │
│             or getInitialNotification() on cold start       │
│ → Navigate to correct screen                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ FOREGROUND                                                  │
│ FCM delivers → messaging().onMessage() fires                │
│ → Show in-app banner (custom component)                     │
│ → Post local notification via expo-notifications (system    │
│   tray)                                                     │
│ User taps system notification → expo-notifications response │
│   listener → Navigate via same routing logic                │
│ User taps in-app banner → Navigate directly                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ LOCAL NOTIFICATIONS                                         │
│ App schedules via expo-notifications                        │
│ OS fires at scheduled time                                  │
│ User taps → expo-notifications response listener            │
│ → Navigate via same routing logic                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Prerequisites & Installation

### Required Packages

```bash
# Firebase
npx expo install @react-native-firebase/app @react-native-firebase/messaging

# Local notifications
npx expo install expo-notifications expo-device

# Already included in most Expo projects
npx expo install expo-constants
```

### Firebase Project Setup

1. Create Firebase project at [console.firebase.google.com](https://console.firebase.google.com)
2. Add iOS app → download `GoogleService-Info.plist` → place in `ios/{AppName}/`
3. Add Android app → download `google-services.json` → place in `android/app/`
4. Enable Cloud Messaging in Firebase console

### app.json Plugin Configuration

```json
{
  "expo": {
    "plugins": [
      "@react-native-firebase/app",
      "@react-native-firebase/messaging",
      [
        "expo-notifications",
        {
          "icon": "./assets/images/icon.png",
          "color": "{YOUR_BRAND_COLOR}"
        }
      ]
    ],
    "ios": {
      "infoPlist": {
        "UIBackgroundModes": ["remote-notification"]
      }
    },
    "android": {
      "permissions": ["POST_NOTIFICATIONS"]
    }
  }
}
```

### Expo Config Plugin for Firebase (iOS)

`GoogleService-Info.plist` must be added to the Xcode project's build resources. Create a config plugin:

```js
// plugins/withFirebaseConfig.js
const { withXcodeProject } = require('@expo/config-plugins');

module.exports = function withFirebaseConfig(config) {
  return withXcodeProject(config, async (config) => {
    const xcodeProject = config.modResults;
    const appName = config.modRequest.projectName;
    const plistPath = `${appName}/GoogleService-Info.plist`;

    // Add GoogleService-Info.plist to Xcode project resources
    // Use manual PBX manipulation (addResourceFile API is broken in some versions)
    const groupKey = xcodeProject.findPBXGroupKey({ name: appName });
    if (groupKey) {
      // Create file reference
      const fileRef = xcodeProject.generateUuid();
      xcodeProject.hash.project.objects.PBXFileReference[fileRef] = {
        isa: 'PBXFileReference',
        lastKnownFileType: 'text.plist.xml',
        path: plistPath,
        name: 'GoogleService-Info.plist',
        sourceTree: '"<group>"',
      };
      // Add to resources build phase
      // (full implementation depends on project structure)
    }

    return config;
  });
};
```

Add to `app.json` plugins:
```json
["./plugins/withFirebaseConfig"]
```

> **Pitfall**: The `path` in `PBXFileReference` must include the app name prefix (e.g., `YourApp/GoogleService-Info.plist`), not just `GoogleService-Info.plist`. This matches the pattern of other files in the Xcode project group.

---

## 3. iOS-Specific Setup

### Critical: Register Device for Remote Messages

On iOS, you **MUST** call `messaging().registerDeviceForRemoteMessages()` before `messaging().getToken()`. Without this, you'll get:

```
[messaging/unregistered] You must be registered for remote messages before calling getToken
```

```typescript
// In your FCM service, AFTER requestPermission() and BEFORE getToken():
if (Platform.OS === 'ios') {
  if (!messaging().isDeviceRegisteredForRemoteMessages) {
    await messaging().registerDeviceForRemoteMessages();
  }
}
```

### APNs Token Handling

FCM acts as a proxy to APNs on iOS. The APNs token must be available before the FCM token works in production:

```typescript
private async ensureAPNSToken(): Promise<boolean> {
  if (Platform.OS !== 'ios') return true;

  let apnsToken = await messaging().getAPNSToken();
  if (apnsToken) return true;

  // Wait with retry
  const maxAttempts = 5;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    await new Promise(resolve => setTimeout(resolve, 1000));
    apnsToken = await messaging().getAPNSToken();
    if (apnsToken) return true;
  }

  return false;
}
```

### iOS Provisional Permissions

Request provisional permission for a better UX — notifications are delivered quietly until the user explicitly interacts:

```typescript
const authStatus = await messaging().requestPermission({
  alert: true,
  badge: true,
  sound: true,
  provisional: Platform.OS === 'ios',
});
```

---

## 4. Type Definitions

### fcm.d.ts

```typescript
// CUSTOMIZE: Replace NotificationType with your app's notification types
import type { NotificationType } from './notification';

/** FCM notification payload data from backend */
export interface FCMNotificationData {
  type: NotificationType;
  title?: string;
  body?: string;
  notificationId?: string;
  // CUSTOMIZE: Add your app-specific data fields
  // Example fields:
  // userId?: string;
  // conversationId?: string;
  // orderId?: string;
  // actorName?: string;
  // actorImage?: string;
  [key: string]: string | undefined; // Allow additional custom fields
}

/** Parsed remote message */
export interface FCMRemoteMessage {
  messageId?: string;
  notification?: {
    title?: string;
    body?: string;
    android?: { channelId?: string; imageUrl?: string; sound?: string };
    ios?: { badge?: string; sound?: string };
  };
  data?: FCMNotificationData;
  from?: string;
  sentTime?: number;
  ttl?: number;
}

export type FCMAuthorizationStatus =
  | 'authorized'
  | 'provisional'
  | 'denied'
  | 'not_determined';

export interface FCMNavigationRoute {
  pathname: string;
  params?: Record<string, string>;
}

export interface FCMTokenRegistrationResponse {
  success: boolean;
  statusCode: number;
  message: string;
  data?: { registered: boolean };
}

export interface FCMEventListeners {
  onForegroundMessage?: (message: FCMRemoteMessage) => void;
  onNotificationOpened?: (message: FCMRemoteMessage) => void;
  onTokenRefresh?: (token: string) => void;
}
```

### notification.d.ts

```typescript
// CUSTOMIZE: Define your app's notification types
export type NotificationType =
  | 'MESSAGE'
  | 'SOCIAL'
  | 'SYSTEM';
  // Add more types as needed, e.g.:
  // | 'ORDER_UPDATE'
  // | 'PROMOTION'
  // | 'REMINDER'
```

---

## 5. FCM Service

Singleton service handling all Firebase Cloud Messaging operations.

```typescript
// services/fcmService.ts

import messaging, { FirebaseMessagingTypes } from '@react-native-firebase/messaging';
import { Platform } from 'react-native';
import { router } from 'expo-router';
import Constants from 'expo-constants';
import type {
  FCMNotificationData,
  FCMRemoteMessage,
  FCMAuthorizationStatus,
  FCMNavigationRoute,
  FCMTokenRegistrationResponse,
  FCMEventListeners,
} from '../types/fcm';

class FCMNotificationService {
  private isInitialized = false;
  private currentToken: string | null = null;
  private listeners: FCMEventListeners = {};
  private unsubscribeForeground: (() => void) | null = null;
  private unsubscribeTokenRefresh: (() => void) | null = null;

  // ── Initialization ────────────────────────────────────────

  async initialize(): Promise<void> {
    if (this.isInitialized) return;

    // 1. Request permissions
    const hasPermission = await this.requestPermission();
    if (!hasPermission) return;

    // 2. iOS: Register device for remote messages (MUST be before getToken)
    if (Platform.OS === 'ios') {
      if (!messaging().isDeviceRegisteredForRemoteMessages) {
        await messaging().registerDeviceForRemoteMessages();
      }
    }

    // 3. Get and register token with retry
    await this.getAndRegisterTokenWithRetry();

    // 4. Set up message handlers
    this.setupMessageHandlers();

    // 5. Handle notification that opened app from quit state
    await this.handleInitialNotification();

    this.isInitialized = true;
  }

  // ── Permissions ───────────────────────────────────────────

  async requestPermission(): Promise<boolean> {
    const authStatus = await messaging().requestPermission({
      alert: true,
      badge: true,
      sound: true,
      provisional: Platform.OS === 'ios',
    });

    return (
      authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
      authStatus === messaging.AuthorizationStatus.PROVISIONAL
    );
  }

  async getAuthorizationStatus(): Promise<FCMAuthorizationStatus> {
    const status = await messaging().hasPermission();
    switch (status) {
      case messaging.AuthorizationStatus.AUTHORIZED:
        return 'authorized';
      case messaging.AuthorizationStatus.PROVISIONAL:
        return 'provisional';
      case messaging.AuthorizationStatus.DENIED:
        return 'denied';
      default:
        return 'not_determined';
    }
  }

  // ── Token Management ──────────────────────────────────────

  private async ensureAPNSToken(): Promise<boolean> {
    if (Platform.OS !== 'ios') return true;

    let apnsToken = await messaging().getAPNSToken();
    if (apnsToken) return true;

    for (let attempt = 1; attempt <= 5; attempt++) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      apnsToken = await messaging().getAPNSToken();
      if (apnsToken) return true;
    }

    return false;
  }

  async getToken(): Promise<string | null> {
    try {
      if (Platform.OS === 'ios') {
        await this.ensureAPNSToken();
      }
      const token = await messaging().getToken();
      this.currentToken = token;
      return token;
    } catch (error) {
      console.error('FCM: Failed to get token', error);
      return null;
    }
  }

  private async getAndRegisterTokenWithRetry(maxRetries = 3): Promise<boolean> {
    const token = await this.getToken();
    if (!token) return false;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      const success = await this.registerTokenWithBackend(token);
      if (success) return true;

      if (attempt < maxRetries) {
        // Exponential backoff: 1s, 2s, 4s
        await new Promise(resolve =>
          setTimeout(resolve, Math.pow(2, attempt - 1) * 1000)
        );
      }
    }
    return false;
  }

  async registerTokenWithBackend(token: string): Promise<boolean> {
    try {
      const os = Platform.OS as 'ios' | 'android';
      const deviceName = Constants.deviceName || `${Platform.OS} Device`;

      // CUSTOMIZE: Replace with your API call
      const response = await httpService.post<FCMTokenRegistrationResponse>(
        '/auth/register-device-info',
        { fcmToken: token, os, deviceName }
      );
      return response.success;
    } catch (error) {
      console.error('FCM: Failed to register token', error);
      return false;
    }
  }

  /**
   * Force register token — call after login to ensure token is registered
   */
  async forceRegisterToken(): Promise<boolean> {
    const hasPermission = await this.requestPermission();
    if (!hasPermission) return false;

    // iOS: register device for remote messages
    if (Platform.OS === 'ios') {
      if (!messaging().isDeviceRegisteredForRemoteMessages) {
        await messaging().registerDeviceForRemoteMessages();
      }
    }

    const token = await this.getToken();
    if (!token) return false;

    return this.registerTokenWithBackend(token);
  }

  async unregisterTokenFromBackend(): Promise<boolean> {
    if (!this.currentToken) return true;
    try {
      // CUSTOMIZE: Replace with your logout API call
      await httpService.post('/auth/logout', { fcmToken: this.currentToken });
      return true;
    } catch (error) {
      return false;
    }
  }

  async deleteToken(): Promise<void> {
    await this.unregisterTokenFromBackend();
    await messaging().deleteToken();
    this.currentToken = null;
  }

  // ── Message Handlers ──────────────────────────────────────

  private setupMessageHandlers(): void {
    // Foreground messages
    this.unsubscribeForeground = messaging().onMessage(
      async (remoteMessage: FirebaseMessagingTypes.RemoteMessage) => {
        const message = this.parseRemoteMessage(remoteMessage);
        this.listeners.onForegroundMessage?.(message);
      }
    );

    // Background/quit notification tap
    messaging().onNotificationOpenedApp(
      (remoteMessage: FirebaseMessagingTypes.RemoteMessage) => {
        const message = this.parseRemoteMessage(remoteMessage);
        this.listeners.onNotificationOpened?.(message);
        this.handleNotificationNavigation(message.data);
      }
    );

    // Token refresh
    this.unsubscribeTokenRefresh = messaging().onTokenRefresh(async (token) => {
      this.currentToken = token;
      await this.registerTokenWithBackend(token);
      this.listeners.onTokenRefresh?.(token);
    });
  }

  private async handleInitialNotification(): Promise<void> {
    const remoteMessage = await messaging().getInitialNotification();
    if (remoteMessage) {
      const message = this.parseRemoteMessage(remoteMessage);
      // Delay to ensure app is fully loaded
      setTimeout(() => {
        this.handleNotificationNavigation(message.data);
      }, 1000);
    }
  }

  // ── Navigation ────────────────────────────────────────────

  handleNotificationNavigation(data?: FCMNotificationData): void {
    if (!data?.type) return;

    const route = this.getNavigationRoute(data);
    if (route) {
      router.push({
        pathname: route.pathname as any,
        params: route.params,
      });
    }
  }

  /**
   * CUSTOMIZE: Map your notification types to app routes
   */
  getNavigationRoute(data: FCMNotificationData): FCMNavigationRoute | null {
    const { type } = data;

    switch (type) {
      case 'MESSAGE':
        // Example: navigate to a chat/message screen
        // return { pathname: '/chat/[id]', params: { id: data.conversationId } };
        return { pathname: '/notifications', params: {} };
      case 'SOCIAL':
        // Example: navigate to a user profile
        // return { pathname: '/profile/[id]', params: { id: data.userId } };
        return { pathname: '/notifications', params: {} };
      case 'SYSTEM':
      default:
        return { pathname: '/notifications', params: {} };
    }

    return { pathname: '/notifications', params: {} };
  }

  // ── Event Listeners ───────────────────────────────────────

  setEventListeners(listeners: FCMEventListeners): void {
    this.listeners = { ...this.listeners, ...listeners };
  }

  removeEventListeners(): void {
    this.listeners = {};
  }

  // ── Cleanup ───────────────────────────────────────────────

  cleanup(): void {
    this.unsubscribeForeground?.();
    this.unsubscribeTokenRefresh?.();
    this.listeners = {};
    this.isInitialized = false;
  }

  // ── Helpers ───────────────────────────────────────────────

  private parseRemoteMessage(
    remoteMessage: FirebaseMessagingTypes.RemoteMessage
  ): FCMRemoteMessage {
    return {
      messageId: remoteMessage.messageId,
      notification: remoteMessage.notification
        ? {
            title: remoteMessage.notification.title,
            body: remoteMessage.notification.body,
            android: remoteMessage.notification.android,
            ios: remoteMessage.notification.ios
              ? {
                  badge: remoteMessage.notification.ios.badge,
                  sound:
                    typeof remoteMessage.notification.ios.sound === 'string'
                      ? remoteMessage.notification.ios.sound
                      : undefined,
                }
              : undefined,
          }
        : undefined,
      data: remoteMessage.data as FCMNotificationData | undefined,
      from: remoteMessage.from,
      sentTime: remoteMessage.sentTime,
      ttl: remoteMessage.ttl,
    };
  }

  get initialized(): boolean { return this.isInitialized; }
  get token(): string | null { return this.currentToken; }
}

export const fcmService = new FCMNotificationService();

// Background handler — export for index.js
export const fcmBackgroundHandler = async (
  remoteMessage: FirebaseMessagingTypes.RemoteMessage
): Promise<void> => {
  console.log('FCM: Background message received', remoteMessage.messageId);
};
```

---

## 6. Local Notification Service

Singleton wrapping `expo-notifications` for local notifications and foreground FCM display.

```typescript
// services/localNotificationService.ts

import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import { Platform } from 'react-native';
import { fcmService } from './fcmService';
import type { FCMRemoteMessage, FCMNotificationData } from '../types/fcm';
import type { NotificationType } from '../types/notification';

// ── Android Notification Channels ──────────────────────────
// CUSTOMIZE: Define channels appropriate for your app

export const NOTIFICATION_CHANNELS = {
  DEFAULT: 'default',
  MESSAGES: 'messages',
  SOCIAL: 'social',
  // Add more channels as needed
} as const;

export type NotificationChannelId =
  (typeof NOTIFICATION_CHANNELS)[keyof typeof NOTIFICATION_CHANNELS];

// ── Types ──────────────────────────────────────────────────

export interface ScheduleLocalNotificationOptions {
  title: string;
  body: string;
  data?: Record<string, any>;
  channelId?: NotificationChannelId;
  trigger?: Notifications.NotificationTriggerInput;
  sound?: boolean;
  badge?: number;
}

// ── Channel Mapping ────────────────────────────────────────

/** CUSTOMIZE: Map your notification types to Android channels */
function getChannelForType(type?: NotificationType): NotificationChannelId {
  switch (type) {
    case 'MESSAGE':
      return NOTIFICATION_CHANNELS.MESSAGES;
    case 'SOCIAL':
      return NOTIFICATION_CHANNELS.SOCIAL;
    case 'SYSTEM':
    default:
      return NOTIFICATION_CHANNELS.DEFAULT;
  }
}

// ── Service ────────────────────────────────────────────────

class LocalNotificationService {
  private isInitialized = false;
  private responseSubscription: Notifications.EventSubscription | null = null;

  /**
   * Initialize — call ONCE during app startup (inside your notification provider).
   */
  async initialize(): Promise<void> {
    if (this.isInitialized) return;

    // How to present notifications when app is in foreground
    Notifications.setNotificationHandler({
      handleNotification: async () => ({
        shouldShowAlert: true,
        shouldPlaySound: true,
        shouldSetBadge: false,
        shouldShowBanner: true,
        shouldShowList: true,
      }),
    });

    // Create Android channels
    if (Platform.OS === 'android') {
      await this.createAndroidChannels();
    }

    // Listen for notification taps
    this.responseSubscription =
      Notifications.addNotificationResponseReceivedListener(
        this.handleNotificationResponse
      );

    this.isInitialized = true;
  }

  async requestPermissions(): Promise<boolean> {
    if (!Device.isDevice) return false;

    const { status: existingStatus } = await Notifications.getPermissionsAsync();
    let finalStatus = existingStatus;

    if (existingStatus !== 'granted') {
      const { status } = await Notifications.requestPermissionsAsync();
      finalStatus = status;
    }

    return finalStatus === 'granted';
  }

  /**
   * CUSTOMIZE: Define your Android notification channels.
   * Importance levels: DEFAULT (no sound), HIGH (sound + heads-up), MAX
   */
  private async createAndroidChannels(): Promise<void> {
    const channels = [
      {
        id: NOTIFICATION_CHANNELS.DEFAULT,
        name: 'General',
        description: 'General notifications',
        importance: Notifications.AndroidImportance.DEFAULT,
      },
      {
        id: NOTIFICATION_CHANNELS.MESSAGES,
        name: 'Messages',
        description: 'Chat message notifications',
        importance: Notifications.AndroidImportance.HIGH,
      },
      {
        id: NOTIFICATION_CHANNELS.SOCIAL,
        name: 'Social',
        description: 'Social notifications',
        importance: Notifications.AndroidImportance.DEFAULT,
      },
    ];

    for (const channel of channels) {
      await Notifications.setNotificationChannelAsync(channel.id, {
        name: channel.name,
        description: channel.description,
        importance: channel.importance,
        vibrationPattern: [0, 250, 250, 250],
        lightColor: '{YOUR_BRAND_COLOR}60',
        sound: 'default',
      });
    }
  }

  // ── Foreground FCM → System Notification ─────────────────

  /**
   * Post a system notification for a foreground FCM message.
   * Call this from your onForegroundMessage handler.
   */
  async showNotificationForFCM(message: FCMRemoteMessage): Promise<string | null> {
    try {
      const data = message.data;
      const title = message.notification?.title || data?.title || 'App';
      const body = message.notification?.body || data?.body || '';
      const channelId = getChannelForType(data?.type);

      const identifier = await Notifications.scheduleNotificationAsync({
        content: {
          title,
          body,
          data: data
            ? { ...data, _source: 'fcm_foreground' }
            : { _source: 'fcm_foreground' },
          sound: 'default',
          ...(Platform.OS === 'android' ? { channelId } : {}),
        },
        trigger: null, // null = fire immediately
      });

      return identifier;
    } catch (error) {
      console.error('Failed to show FCM foreground notification', error);
      return null;
    }
  }

  // ── Local Notification Scheduling ────────────────────────

  /**
   * Schedule a local notification.
   *
   * @example
   * // Fire immediately
   * await localNotificationService.schedule({
   *   title: 'Reminder',
   *   body: 'Check back for new content!',
   * });
   *
   * @example
   * // Schedule for 5 minutes from now
   * await localNotificationService.schedule({
   *   title: 'Come back!',
   *   body: 'Your friends are waiting.',
   *   trigger: {
   *     type: SchedulableTriggerInputTypes.TIME_INTERVAL,
   *     seconds: 300,
   *     repeats: false,
   *   },
   * });
   *
   * @example
   * // Daily at 10:00 AM
   * await localNotificationService.schedule({
   *   title: 'Daily Reminder',
   *   body: 'New content available!',
   *   trigger: {
   *     type: SchedulableTriggerInputTypes.DAILY,
   *     hour: 10,
   *     minute: 0,
   *   },
   * });
   */
  async schedule(options: ScheduleLocalNotificationOptions): Promise<string | null> {
    try {
      const {
        title, body, data,
        channelId = NOTIFICATION_CHANNELS.DEFAULT,
        trigger = null,
        sound = true,
        badge,
      } = options;

      return await Notifications.scheduleNotificationAsync({
        content: {
          title,
          body,
          data: { ...data, _source: 'local' },
          sound: sound ? 'default' : undefined,
          badge,
          ...(Platform.OS === 'android' ? { channelId } : {}),
        },
        trigger,
      });
    } catch (error) {
      console.error('Failed to schedule notification', error);
      return null;
    }
  }

  async cancel(identifier: string): Promise<void> {
    await Notifications.cancelScheduledNotificationAsync(identifier);
  }

  async cancelAll(): Promise<void> {
    await Notifications.cancelAllScheduledNotificationsAsync();
  }

  async getScheduled(): Promise<Notifications.NotificationRequest[]> {
    return Notifications.getAllScheduledNotificationsAsync();
  }

  async dismissAll(): Promise<void> {
    await Notifications.dismissAllNotificationsAsync();
  }

  async setBadgeCount(count: number): Promise<void> {
    await Notifications.setBadgeCountAsync(count);
  }

  // ── Notification Tap Handler ─────────────────────────────

  private handleNotificationResponse = (
    response: Notifications.NotificationResponseReceivedEvent
  ): void => {
    if (response.actionIdentifier !== Notifications.DEFAULT_ACTION_IDENTIFIER) return;

    const data = response.notification.request.content.data as
      | (FCMNotificationData & { _source?: string })
      | undefined;

    if (data) {
      // Reuse FCM navigation logic for consistent routing
      fcmService.handleNotificationNavigation(data as FCMNotificationData);
    }
  };

  // ── Cleanup ──────────────────────────────────────────────

  cleanup(): void {
    this.responseSubscription?.remove();
    this.responseSubscription = null;
    this.isInitialized = false;
  }

  get initialized(): boolean { return this.isInitialized; }
}

export const localNotificationService = new LocalNotificationService();

// Re-export trigger types for convenience
export { SchedulableTriggerInputTypes } from 'expo-notifications';
```

---

## 7. React Hooks

### useFCM Hook

```typescript
// hooks/useFCM.ts

import { useEffect, useState, useCallback, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';
import { fcmService } from '../services/fcmService';
import type { FCMRemoteMessage, FCMAuthorizationStatus } from '../types/fcm';

interface UseFCMOptions {
  autoInitialize?: boolean;
  onForegroundMessage?: (message: FCMRemoteMessage) => void;
  onNotificationOpened?: (message: FCMRemoteMessage) => void;
  onTokenRefresh?: (token: string) => void;
}

interface UseFCMReturn {
  isInitialized: boolean;
  isLoading: boolean;
  token: string | null;
  authStatus: FCMAuthorizationStatus | null;
  error: Error | null;
  initialize: () => Promise<void>;
  requestPermission: () => Promise<boolean>;
  handleNotificationPress: (message: FCMRemoteMessage) => void;
  forceRegisterToken: () => Promise<boolean>;
}

export function useFCM(options: UseFCMOptions = {}): UseFCMReturn {
  const {
    autoInitialize = true,
    onForegroundMessage,
    onNotificationOpened,
    onTokenRefresh,
  } = options;

  // CUSTOMIZE: Replace with your auth state selector
  // const isAuthenticated = useSelector((state: any) => state.auth.isAuthenticated);
  const isAuthenticated = true; // Replace with actual auth check

  const [isInitialized, setIsInitialized] = useState(fcmService.initialized);
  const [isLoading, setIsLoading] = useState(false);
  const [token, setToken] = useState<string | null>(fcmService.token);
  const [authStatus, setAuthStatus] = useState<FCMAuthorizationStatus | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const initializationAttempted = useRef(false);

  const initialize = useCallback(async () => {
    if (fcmService.initialized || isLoading) return;

    setIsLoading(true);
    setError(null);
    try {
      await fcmService.initialize();
      setIsInitialized(true);
      setToken(fcmService.token);
      const status = await fcmService.getAuthorizationStatus();
      setAuthStatus(status);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('FCM init failed'));
    } finally {
      setIsLoading(false);
    }
  }, [isLoading]);

  const requestPermission = useCallback(async () => {
    const granted = await fcmService.requestPermission();
    const status = await fcmService.getAuthorizationStatus();
    setAuthStatus(status);
    return granted;
  }, []);

  const handleNotificationPress = useCallback((message: FCMRemoteMessage) => {
    fcmService.handleNotificationNavigation(message.data);
  }, []);

  const forceRegisterToken = useCallback(async () => {
    const success = await fcmService.forceRegisterToken();
    if (success) {
      setToken(fcmService.token);
      const status = await fcmService.getAuthorizationStatus();
      setAuthStatus(status);
    }
    return success;
  }, []);

  // CRITICAL: Set listeners and initialize in a single effect
  // to prevent race conditions
  useEffect(() => {
    fcmService.setEventListeners({
      onForegroundMessage: (message) => onForegroundMessage?.(message),
      onNotificationOpened: (message) => onNotificationOpened?.(message),
      onTokenRefresh: (newToken) => {
        setToken(newToken);
        onTokenRefresh?.(newToken);
      },
    });

    if (autoInitialize && isAuthenticated && !fcmService.initialized && !initializationAttempted.current) {
      initializationAttempted.current = true;
      initialize();
    }

    return () => fcmService.removeEventListeners();
  }, [autoInitialize, isAuthenticated, initialize, onForegroundMessage, onNotificationOpened, onTokenRefresh]);

  // Re-check auth status when app returns to foreground
  useEffect(() => {
    const sub = AppState.addEventListener('change', async (state: AppStateStatus) => {
      if (state === 'active' && fcmService.initialized && isAuthenticated) {
        setAuthStatus(await fcmService.getAuthorizationStatus());
      }
    });
    return () => sub.remove();
  }, [isAuthenticated]);

  return {
    isInitialized, isLoading, token, authStatus, error,
    initialize, requestPermission, handleNotificationPress, forceRegisterToken,
  };
}

/** Hook for managing foreground notification banner visibility */
export function useFCMForegroundNotification() {
  const [notification, setNotification] = useState<FCMRemoteMessage | null>(null);
  const [isVisible, setIsVisible] = useState(false);
  const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  const showNotification = useCallback((message: FCMRemoteMessage) => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    setNotification(message);
    setIsVisible(true);
    timeoutRef.current = setTimeout(() => setIsVisible(false), 4000);
  }, []);

  const hideNotification = useCallback(() => {
    if (timeoutRef.current) clearTimeout(timeoutRef.current);
    setIsVisible(false);
  }, []);

  const handlePress = useCallback(() => {
    if (notification) {
      fcmService.handleNotificationNavigation(notification.data);
      hideNotification();
    }
  }, [notification, hideNotification]);

  useEffect(() => {
    return () => { if (timeoutRef.current) clearTimeout(timeoutRef.current); };
  }, []);

  return { notification, isVisible, showNotification, hideNotification, handlePress };
}
```

### useLocalNotifications Hook

```typescript
// hooks/useLocalNotifications.ts

import { useCallback } from 'react';
import {
  localNotificationService,
  type ScheduleLocalNotificationOptions,
} from '../services/localNotificationService';

export function useLocalNotifications() {
  const schedule = useCallback(
    (options: ScheduleLocalNotificationOptions) =>
      localNotificationService.schedule(options),
    []
  );

  const cancel = useCallback(
    (identifier: string) => localNotificationService.cancel(identifier),
    []
  );

  const cancelAll = useCallback(
    () => localNotificationService.cancelAll(),
    []
  );

  const getScheduled = useCallback(
    () => localNotificationService.getScheduled(),
    []
  );

  const dismissAll = useCallback(
    () => localNotificationService.dismissAll(),
    []
  );

  const setBadgeCount = useCallback(
    (count: number) => localNotificationService.setBadgeCount(count),
    []
  );

  return { schedule, cancel, cancelAll, getScheduled, dismissAll, setBadgeCount };
}
```

---

## 8. Provider & UI Components

### FCMProvider

```tsx
// components/notifications/FCMProvider.tsx

import React, { createContext, useContext, useEffect, ReactNode } from 'react';
import { useFCM, useFCMForegroundNotification } from '@/hooks/useFCM';
import { ForegroundNotificationBanner } from './ForegroundNotificationBanner';
import { localNotificationService } from '@/services/localNotificationService';
import type { FCMRemoteMessage, FCMAuthorizationStatus } from '@/types/fcm';

interface FCMContextValue {
  isInitialized: boolean;
  isLoading: boolean;
  token: string | null;
  authStatus: FCMAuthorizationStatus | null;
  error: Error | null;
  initialize: () => Promise<void>;
  requestPermission: () => Promise<boolean>;
  forceRegisterToken: () => Promise<boolean>;
}

const FCMContext = createContext<FCMContextValue | null>(null);

export function FCMProvider({ children }: { children: ReactNode }) {
  const {
    notification, isVisible, showNotification, hideNotification, handlePress,
  } = useFCMForegroundNotification();

  // Initialize local notification service (channels, handler, tap listener)
  useEffect(() => {
    localNotificationService.initialize();
    return () => localNotificationService.cleanup();
  }, []);

  const fcm = useFCM({
    autoInitialize: true,
    onForegroundMessage: (message: FCMRemoteMessage) => {
      showNotification(message);                                // In-app banner
      localNotificationService.showNotificationForFCM(message); // System tray
    },
    onNotificationOpened: (message: FCMRemoteMessage) => {
      console.log('Notification opened:', message.messageId);
    },
    onTokenRefresh: (_token: string) => {
      console.log('FCM token refreshed');
    },
  });

  const contextValue: FCMContextValue = {
    isInitialized: fcm.isInitialized,
    isLoading: fcm.isLoading,
    token: fcm.token,
    authStatus: fcm.authStatus,
    error: fcm.error,
    initialize: fcm.initialize,
    requestPermission: fcm.requestPermission,
    forceRegisterToken: fcm.forceRegisterToken,
  };

  return (
    <FCMContext.Provider value={contextValue}>
      {children}
      <ForegroundNotificationBanner
        notification={notification}
        isVisible={isVisible}
        onPress={handlePress}
        onDismiss={hideNotification}
      />
    </FCMContext.Provider>
  );
}

export function useFCMContext(): FCMContextValue {
  const context = useContext(FCMContext);
  if (!context) throw new Error('useFCMContext must be used within FCMProvider');
  return context;
}
```

### ForegroundNotificationBanner (Example)

```tsx
// components/notifications/ForegroundNotificationBanner.tsx

import React from 'react';
import { View, Text, TouchableOpacity, Animated } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import type { FCMRemoteMessage, FCMNotificationData } from '@/types/fcm';

interface Props {
  notification: FCMRemoteMessage | null;
  isVisible: boolean;
  onPress: () => void;
  onDismiss: () => void;
}

export function ForegroundNotificationBanner({
  notification, isVisible, onPress, onDismiss,
}: Props) {
  const insets = useSafeAreaInsets();
  const translateY = React.useRef(new Animated.Value(-100)).current;

  React.useEffect(() => {
    Animated.spring(translateY, {
      toValue: isVisible ? 0 : -100,
      useNativeDriver: true,
      tension: 100,
      friction: 10,
    }).start();
  }, [isVisible]);

  if (!notification) return null;

  const data = notification.data as FCMNotificationData | undefined;
  const title = notification.notification?.title || data?.title || 'Notification';
  const body = notification.notification?.body || data?.body || '';

  return (
    <Animated.View
      style={{
        position: 'absolute',
        top: insets.top,
        left: 0, right: 0,
        zIndex: 9999,
        transform: [{ translateY }],
      }}
    >
      <TouchableOpacity onPress={onPress} activeOpacity={0.9}>
        {/* CUSTOMIZE: Style to match your app's design system */}
        <View style={{
          margin: 16,
          backgroundColor: '#1A1A1A',
          borderRadius: 16,
          padding: 16,
          shadowColor: '#000',
          shadowOffset: { width: 0, height: 4 },
          shadowOpacity: 0.3,
          shadowRadius: 8,
          elevation: 8,
        }}>
          <Text style={{ color: '#fff', fontWeight: '600', fontSize: 16 }} numberOfLines={1}>
            {title}
          </Text>
          {body ? (
            <Text style={{ color: '#999', fontSize: 14, marginTop: 4 }} numberOfLines={2}>
              {body}
            </Text>
          ) : null}
        </View>
      </TouchableOpacity>
    </Animated.View>
  );
}
```

---

## 9. Entry Point (Background Handler)

Register the background message handler in `index.js` **before** any React components mount:

```javascript
// index.js

import '@react-native-firebase/app';
import messaging from '@react-native-firebase/messaging';
import { Platform } from 'react-native';
import 'expo-router/entry';

try {
  const messagingInstance = messaging();

  if (Platform.OS === 'ios' || Platform.OS === 'android') {
    messagingInstance.setBackgroundMessageHandler(async (remoteMessage) => {
      console.log('FCM Background Message:', {
        messageId: remoteMessage.messageId,
        data: remoteMessage.data,
      });
      // Background messages are automatically displayed by the OS.
      // Navigation is handled when user taps the notification
      // (onNotificationOpenedApp or getInitialNotification).
    });
  }
} catch (error) {
  // Firebase not configured for this runtime (e.g., Expo Go)
  console.warn('[FCM] Failed to initialize background handler', error);
}
```

### App Layout — Wrap with FCMProvider

```tsx
// app/_layout.tsx (Expo Router example)

import { FCMProvider } from '@/components/notifications/FCMProvider';

export default function RootLayout() {
  return (
    <FCMProvider>
      {/* Your app content */}
      <Stack />
    </FCMProvider>
  );
}
```

### After Login — Force Register Token

```typescript
// In your login success handler:
const { forceRegisterToken } = useFCMContext();

const handleLoginSuccess = async () => {
  // ... login logic ...
  await forceRegisterToken(); // Register FCM token with backend
};
```

---

## 10. Backend Requirements

### Token Registration Endpoint

```typescript
// POST /auth/register-device-info
// Body: { fcmToken: string, os: 'ios' | 'android', deviceName: string }

// Store FCM tokens per user (one user can have multiple devices)
// On logout: remove the specific device token
```

### Sending Notifications (firebase-admin)

```typescript
import * as admin from 'firebase-admin';

// Initialize once
admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
});

// Send notification (V1 API)
const message: admin.messaging.MulticastMessage = {
  tokens: userFcmTokens, // array of device tokens
  notification: {
    title: 'New Message',
    body: 'You have a new message from Alice',
  },
  data: {
    type: 'MESSAGE',
    // CUSTOMIZE: Add your notification-specific data fields
    // conversationId: 'conv_123',
    // userId: 'user_456',
  },
  android: {
    priority: 'high',
    notification: { channelId: 'messages' },
  },
  apns: {
    payload: {
      aps: {
        badge: unreadCount,
        sound: 'default',
      },
    },
  },
};

const response = await admin.messaging().sendEachForMulticast(message);
// Handle failures: remove invalid tokens from database
```

### Payload Structure

| Field | Purpose |
|-------|---------|
| `notification.title` / `notification.body` | Displayed by OS in background (data-only messages won't show system notifications automatically on iOS) |
| `data.*` | Custom data available in foreground `onMessage()` and tap handlers |

> **Important**: Always send both `notification` and `data` fields. The `notification` field ensures the OS displays the notification in background/killed state. The `data` field provides navigation context.

---

## 11. Common Pitfalls & Solutions

### iOS: `[messaging/unregistered]` Error

**Problem**: `messaging().getToken()` throws `You must be registered for remote messages`

**Solution**: Call `messaging().registerDeviceForRemoteMessages()` after `requestPermission()` and before `getToken()`:
```typescript
if (Platform.OS === 'ios' && !messaging().isDeviceRegisteredForRemoteMessages) {
  await messaging().registerDeviceForRemoteMessages();
}
```

### iOS: `GoogleService-Info.plist` Not in Build

**Problem**: App crashes at `[FIRApp configure]` with SIGABRT

**Solution**: Use an Expo config plugin to add the file to Xcode project resources. The `path` in `PBXFileReference` must include the app name prefix (e.g., `AppName/GoogleService-Info.plist`).

### Race Condition: Listeners vs Initialization

**Problem**: Messages arrive after `initialize()` calls `setupMessageHandlers()` but before listeners are set, so `onForegroundMessage` is undefined.

**Solution**: Set event listeners and initialize in a **single combined `useEffect`** — listeners first, then initialize:
```typescript
useEffect(() => {
  fcmService.setEventListeners({ onForegroundMessage: ... }); // First
  if (shouldInit) fcmService.initialize();                      // Then
  return () => fcmService.removeEventListeners();
}, [...]);
```

### No Token Registration Retry

**Problem**: Network instability causes silent FCM token registration failures.

**Solution**: Exponential backoff retry (3 attempts: 1s, 2s, 4s delays).

### Foreground Messages Not Visible

**Problem**: On both iOS and Android, foreground FCM messages are NOT displayed by the OS.

**Solution**: In `onMessage()`, post a local notification via `expo-notifications`:
```typescript
localNotificationService.showNotificationForFCM(message);
```

### Double Navigation on Tap

**Problem**: Both Firebase and expo-notifications handle the same notification tap.

**Solution**: They handle **different** scenarios and don't overlap:
- **Background tap**: Firebase's `onNotificationOpenedApp()` handles it. expo-notifications is NOT involved.
- **Foreground local notification tap**: expo-notifications' `addNotificationResponseReceivedListener` handles it. Firebase's listener does NOT fire for local notifications.

### Missing User Data in Navigation

**Problem**: Tapping a notification opens a screen without required context data.

**Solution**: Pass all necessary data fields in the FCM `data` payload (e.g., user name, avatar URL, entity IDs), and use them as route params. Add a fallback that fetches missing data from the API.

---

## 12. Testing Checklist

- [ ] **iOS Build**: `npx expo prebuild --clean && npx expo run:ios` — no crashes
- [ ] **Android Build**: `npx expo prebuild --clean && npx expo run:android` — no crashes
- [ ] **FCM Token**: Check logs for successful token registration with backend
- [ ] **iOS FCM Token**: No `[messaging/unregistered]` error
- [ ] **Background Push**: Send push while app is backgrounded → notification appears in tray
- [ ] **Background Tap**: Tap background notification → app opens, navigates to correct screen
- [ ] **Killed-State Push**: Kill app, send push → notification appears, tap opens app and navigates
- [ ] **Foreground Push**: Send push while app is open → in-app banner appears AND system notification in tray
- [ ] **Foreground Tap (Banner)**: Tap in-app banner → navigates to correct screen
- [ ] **Foreground Tap (Tray)**: Pull down notification tray, tap notification → navigates to correct screen
- [ ] **Local Notification**: Schedule a local notification → fires at correct time
- [ ] **Android Channels**: Settings > Apps > [App] > Notifications → channels visible
- [ ] **Token Refresh**: Force token refresh → new token registered with backend
- [ ] **Logout**: Logout → token unregistered, no more notifications
- [ ] **Re-login**: Login again → `forceRegisterToken()` succeeds
- [ ] **Permission Denied**: Deny notification permission → app handles gracefully, no crash

---

## Quick Start Summary

1. Install packages (`@react-native-firebase/app`, `@react-native-firebase/messaging`, `expo-notifications`, `expo-device`)
2. Configure Firebase (download config files, add Expo config plugin)
3. Add plugins to `app.json`
4. Create type definitions (`fcm.d.ts`, `notification.d.ts`)
5. Create `fcmService.ts` (singleton, handles push lifecycle)
6. Create `localNotificationService.ts` (singleton, handles local + foreground display)
7. Create hooks (`useFCM.ts`, `useLocalNotifications.ts`)
8. Create `FCMProvider.tsx` and wrap app
9. Register background handler in `index.js`
10. Call `forceRegisterToken()` after login
11. Implement backend token storage and `firebase-admin` sending
12. Run through testing checklist
