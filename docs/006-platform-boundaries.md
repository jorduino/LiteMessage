# 006-Platform-Boundaries.md

This document defines the responsibility boundaries between the backend, shared core library, and platform-specific clients. Clear boundaries ensure code reuse, maintainability, and consistent behavior across Web, Desktop, and Mobile.

---

## Architecture Overview

```text
┌──────────────────────────────────────────────────────────┐
│                         Platforms                        │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐  │
│  │     Web     │    │   Desktop   │    │   Mobile     │  │
│  │   (React)   │    │ (Electron)  │    │(React Native)│  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬───────┘  │
│         │                  │                  │          │
│         └──────────────────┼──────────────────┘          │
│                            │                             │
│                   ┌────────▼────────┐                    │
│                   │   Shared Core   │                    │
│                   │  (TypeScript)   │                    │
│                   └────────┬────────┘                    │
└────────────────────────────┼─────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │     Backend     │
                    │  (REST + WS)    │
                    └─────────────────┘
```

---

## Layer Responsibilities

### Backend

The backend is the source of truth for all persistent data.

| Responsibility              | Description                                        |
| --------------------------- | -------------------------------------------------- |
| User authentication         | Validate credentials, issue/verify JWTs            |
| Data persistence            | Store users, conversations, messages in database   |
| Message routing             | Deliver messages to recipients via WebSocket       |
| Presence tracking           | Track online/offline state, broadcast updates      |
| API endpoints               | Expose REST API per `002-api-contracts.md`         |
| WebSocket server            | Manage connections, subscriptions, event broadcast |
| Message ordering            | Assign authoritative timestamps to messages        |
| Rate limiting               | Protect against abuse                              |
| Data validation             | Enforce constraints (non-empty messages, etc.)     |

**Backend does NOT:**

- Store UI state
- Make decisions about how data is displayed
- Handle retry logic (client responsibility)
- Cache data for offline use

---

### Shared Core Library

The shared core contains all business logic that is platform-agnostic.

| Responsibility             | Description                                         |
| -------------------------- | --------------------------------------------------- |
| API client                 | HTTP client for REST endpoints                      |
| WebSocket client           | Connection management, reconnection logic           |
| State management           | Centralized store for conversations, messages, users|
| Message lifecycle          | Track message states per `003-message-lifecycle.md` |
| Optimistic updates         | Apply local changes before server confirmation      |
| Deduplication              | Generate and track `clientMessageId` for retries    |
| Event handling             | Process WebSocket events, update local state        |
| Presence management        | Send heartbeats, track other users' presence        |
| Typing indicators          | Debounce and send typing events                     |
| Pagination                 | Cursor management for message history               |
| Error handling             | Classify errors, determine retry eligibility        |
| Data normalization         | Transform API responses into consistent format      |

**Shared Core does NOT:**

- Render UI components
- Access platform-specific APIs (notifications, file system)
- Handle platform-specific navigation
- Store data persistently (delegates to platform)

---

### Platform Layer

Each platform implements UI and platform-specific integrations.

| Responsibility           | Description                                       |
| ------------------------ | ------------------------------------------------- |
| UI rendering             | Display conversations, messages, forms            |
| Navigation               | Route between screens/views                       |
| User input               | Capture text input, button clicks                 |
| Platform notifications   | Show native notifications                         |
| Persistent storage       | Store JWT, cache data for offline (if supported)  |
| Platform lifecycle       | Handle app background/foreground, startup         |
| Accessibility            | Screen readers, keyboard navigation               |
| Theming                  | Light/dark mode, platform-specific styling        |

---

## Platform-Specific Responsibilities

### Web (React)

| Feature                  | Implementation                                   |
| ------------------------ | ------------------------------------------------ |
| State binding            | React hooks connected to shared core store       |
| Notifications            | Browser Notification API                         |
| Storage                  | `localStorage` for JWT, `IndexedDB` for cache    |
| Offline detection        | `navigator.onLine`, `online`/`offline` events    |
| Tab visibility           | Page Visibility API for read receipts            |
| Responsive layout        | CSS media queries, flexbox/grid                  |

### Desktop (Electron)

| Feature                  | Implementation                                   |
| ------------------------ | ------------------------------------------------ |
| State binding            | Same React app wrapped in Electron               |
| Notifications            | Electron `Notification` API (native OS)          |
| Storage                  | `electron-store` or file system                  |
| System tray              | Electron `Tray` API                              |
| Global shortcuts         | Electron `globalShortcut` API                    |
| Auto-launch              | Electron `app.setLoginItemSettings`              |
| IPC                      | Main/renderer process communication              |

### Mobile (React Native)

| Feature                  | Implementation                                   |
| ------------------------ | ------------------------------------------------ |
| State binding            | React Native hooks connected to shared core      |
| Push notifications       | Firebase Cloud Messaging (FCM) / APNs            |
| Storage                  | `AsyncStorage` or `react-native-mmkv`            |
| Background refresh       | Background fetch APIs (platform-specific)        |
| Offline detection        | `NetInfo` library                                |
| Keyboard handling        | `KeyboardAvoidingView`, input accessory          |
| Haptics                  | `react-native-haptic-feedback`                   |

---

## Interface Boundaries

### Shared Core → Platform

The shared core exposes these interfaces for platforms to consume:

```typescript
// State subscriptions
interface CoreStore {
  conversations: Map<string, Conversation>;
  messages: Map<string, Message[]>;
  users: Map<string, User>;
  currentUser: User | null;
  connectionStatus: "connected" | "connecting" | "disconnected";
}

// Actions
interface CoreActions {
  login(email: string, password: string): Promise<User>;
  logout(): Promise<void>;
  register(email: string, password: string, displayName: string): Promise<User>;

  loadConversations(): Promise<Conversation[]>;
  createConversation(participantId: string): Promise<Conversation>;
  findUserByEmail(email: string): Promise<User | null>;

  sendMessage(conversationId: string, text: string): Promise<Message>;
  loadMessages(conversationId: string, cursor?: string): Promise<Message[]>;
  markAsRead(conversationId: string, messageId: string): Promise<void>;
  retryMessage(clientMessageId: string): Promise<Message>;
  deleteFailedMessage(clientMessageId: string): void;

  setTyping(conversationId: string, isTyping: boolean): void;

  connect(): Promise<void>;
  disconnect(): void;
}

// Events for UI updates
interface CoreEvents {
  onStateChange(callback: (state: CoreStore) => void): Unsubscribe;
  onNewMessage(callback: (message: Message) => void): Unsubscribe;
  onTypingUpdate(callback: (update: TypingUpdate) => void): Unsubscribe;
  onPresenceUpdate(callback: (update: PresenceUpdate) => void): Unsubscribe;
  onConnectionChange(callback: (status: ConnectionStatus) => void): Unsubscribe;
}
```

### Platform → Shared Core

Platforms provide these adapters to the shared core:

```typescript
// Storage adapter (platform implements)
interface StorageAdapter {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  remove(key: string): Promise<void>;
}

// Network adapter (platform implements)
interface NetworkAdapter {
  isOnline(): boolean;
  onOnlineChange(callback: (online: boolean) => void): Unsubscribe;
}

// Optional: Notification adapter
interface NotificationAdapter {
  requestPermission(): Promise<boolean>;
  show(title: string, body: string, data?: object): Promise<void>;
}
```

---

## Data Flow Examples

### Sending a Message

```text
┌──────────┐     ┌─────────────┐     ┌─────────┐
│ Platform │     │ Shared Core │     │ Backend │
└────┬─────┘     └──────┬──────┘     └────┬────┘
     │                  │                 │
     │ sendMessage()    │                 │
     │─────────────────>│                 │
     │                  │                 │
     │ onStateChange()  │                 │
     │<─────────────────│ (optimistic)    │
     │ [pending msg]    │                 │
     │                  │                 │
     │                  │ POST /messages  │
     │                  │────────────────>│
     │                  │                 │
     │                  │ 200 OK          │
     │                  │<────────────────│
     │                  │                 │
     │ onStateChange()  │                 │
     │<─────────────────│ (confirmed)     │
     │ [sent msg]       │                 │
```

### Receiving a Message

```text
┌──────────┐     ┌─────────────┐     ┌─────────┐
│ Platform │     │ Shared Core │     │ Backend │
└────┬─────┘     └──────┬──────┘     └────┬────┘
     │                  │                 │
     │                  │ WS: message:new │
     │                  │<────────────────│
     │                  │                 │
     │ onNewMessage()   │                 │
     │<─────────────────│                 │
     │                  │                 │
     │ onStateChange()  │                 │
     │<─────────────────│                 │
     │ [new msg in list]│                 │
     │                  │                 │
     │ showNotification │                 │
     │ (if applicable)  │                 │
```

---

## Responsibility Matrix

| Concern                     | Backend | Shared Core | Platform |
| --------------------------- | :-----: | :---------: | :------: |
| User authentication         | ✓       |             |          |
| JWT storage                 |         |             | ✓        |
| JWT validation              | ✓       |             |          |
| API requests                |         | ✓           |          |
| WebSocket connection        |         | ✓           |          |
| Reconnection logic          |         | ✓           |          |
| Message state machine       |         | ✓           |          |
| Optimistic updates          |         | ✓           |          |
| Data persistence (DB)       | ✓       |             |          |
| Data caching (local)        |         |             | ✓        |
| Message ordering            | ✓       |             |          |
| Message display order       |         | ✓           |          |
| UI rendering                |         |             | ✓        |
| Push notifications          | ✓       |             | ✓        |
| In-app notifications        |         |             | ✓        |
| Typing indicator logic      |         | ✓           |          |
| Typing indicator display    |         |             | ✓        |
| Presence heartbeat          |         | ✓           |          |
| Presence display            |         |             | ✓        |
| Offline detection           |         |             | ✓        |
| Offline state handling      |         | ✓           |          |
| Rate limiting               | ✓       |             |          |
| Input validation (server)   | ✓       |             |          |
| Input validation (client)   |         | ✓           |          |
| Error display               |         |             | ✓        |
| Navigation                  |         |             | ✓        |
| Theming                     |         |             | ✓        |

---

## Package Structure

```text
litemessage/
├── packages/
│   ├── backend/           # Node.js server
│   │   ├── src/
│   │   │   ├── routes/    # REST endpoints
│   │   │   ├── ws/        # WebSocket handlers
│   │   │   ├── db/        # Database access
│   │   │   └── auth/      # JWT handling
│   │   └── package.json
│   │
│   ├── core/              # Shared TypeScript library
│   │   ├── src/
│   │   │   ├── api/       # REST client
│   │   │   ├── ws/        # WebSocket client
│   │   │   ├── store/     # State management
│   │   │   ├── types/     # Shared type definitions
│   │   │   └── index.ts   # Public API
│   │   └── package.json
│   │
│   ├── web/               # React web app
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── hooks/     # Core integration
│   │   │   ├── pages/
│   │   │   └── adapters/  # Storage, network
│   │   └── package.json
│   │
│   ├── desktop/           # Electron wrapper
│   │   ├── src/
│   │   │   ├── main/      # Main process
│   │   │   └── preload/   # Preload scripts
│   │   └── package.json
│   │
│   └── mobile/            # React Native app
│       ├── src/
│       │   ├── components/
│       │   ├── screens/
│       │   └── adapters/  # Storage, network, push
│       └── package.json
│
└── package.json           # Workspace root
```
