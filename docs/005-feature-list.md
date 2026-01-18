# 005-Feature-List.md

This document defines the feature set for LiteMessage, organized by priority and platform. Features are derived from the core concepts in `000-core-concepts.md` and aligned with the roadmap phases.

---

## Feature Categories

| Category | Description                         |
| -------- | ----------------------------------- |
| Core     | Required for MVP, all platforms     |
| Enhanced | Improves UX, implemented after core |
| Optional | Nice-to-have, time permitting       |

---

## Core Features (MVP)

### Authentication

| ID    | Feature             | Description                                       |
| ----- | ------------------- | ------------------------------------------------- |
| F-001 | User registration   | Create account with email, password, display name |
| F-002 | User login          | Authenticate with email and password              |
| F-003 | User logout         | End session and clear local credentials           |
| F-004 | Session persistence | Stay logged in across app restarts (stored JWT)   |

### Conversations

| ID    | Feature            | Description                                       |
| ----- | ------------------ | ------------------------------------------------- |
| F-010 | Start conversation | Create a new 1:1 conversation with another user   |
| F-011 | List conversations | View all conversations, sorted by recent activity |
| F-012 | View conversation  | Open a conversation to see messages               |
| F-013 | Find user by email | Search for users to start conversations           |

### Messaging

| ID    | Feature               | Description                                           |
| ----- | --------------------- | ----------------------------------------------------- |
| F-020 | Send message          | Send a text message in a conversation                 |
| F-021 | Receive message       | Receive messages in real-time via WebSocket           |
| F-022 | Message history       | Load previous messages with pagination                |
| F-023 | Message ordering      | Display messages in chronological order               |
| F-024 | Send status           | Show pending/sent/failed status for outgoing messages |
| F-025 | Retry failed message  | Retry sending a message that failed                   |
| F-026 | Delete failed message | Remove a failed message from the composer             |

### Presence

| ID    | Feature            | Description                                   |
| ----- | ------------------ | --------------------------------------------- |
| F-030 | Online indicator   | Show when a user is currently online          |
| F-031 | Last active time   | Show when a user was last active (if offline) |
| F-032 | Presence broadcast | Update own presence state via heartbeat       |

### Real-time Updates

| ID    | Feature              | Description                                          |
| ----- | -------------------- | ---------------------------------------------------- |
| F-040 | WebSocket connection | Maintain persistent connection for real-time updates |
| F-041 | Auto-reconnect       | Automatically reconnect on connection loss           |
| F-042 | Connection status    | Show online/offline/reconnecting state to user       |

---

## Enhanced Features

### Message Status

| ID    | Feature              | Description                                  |
| ----- | -------------------- | -------------------------------------------- |
| F-050 | Delivery receipts    | Show when message was delivered to recipient |
| F-051 | Read receipts        | Show when message was read by recipient      |
| F-052 | Read receipt sending | Mark messages as read when viewed            |

### Typing Indicators

| ID    | Feature                  | Description                        |
| ----- | ------------------------ | ---------------------------------- |
| F-060 | Typing indicator send    | Broadcast when user is typing      |
| F-061 | Typing indicator display | Show when other user is typing     |
| F-062 | Typing timeout           | Clear indicator after inactivity   |

### Offline Support

| ID    | Feature               | Description                                    |
| ----- | --------------------- | ---------------------------------------------- |
| F-070 | Offline detection     | Detect and display offline state               |
| F-071 | Offline message queue | Prevent sending while offline, queue for later |
| F-072 | Reconnection sync     | Sync missed messages on reconnect              |
| F-073 | Optimistic UI         | Show pending messages immediately in UI        |

---

## Platform-Specific Features

### Web

| ID    | Feature               | Description                               |
| ----- | --------------------- | ----------------------------------------- |
| F-080 | Responsive layout     | Adapt UI to different screen sizes        |
| F-081 | Browser notifications | Show notifications when tab is not active |
| F-082 | Unread badge          | Show unread count in browser tab title    |

### Desktop (Electron)

| ID    | Feature              | Description                            |
| ----- | -------------------- | -------------------------------------- |
| F-090 | System tray          | Minimize to system tray                |
| F-091 | Native notifications | Use OS notification system             |
| F-092 | Startup launch       | Option to launch on system startup     |
| F-093 | Keyboard shortcuts   | Global shortcuts for common actions    |

### Mobile (React Native)

| ID    | Feature              | Description                                     |
| ----- | -------------------- | ----------------------------------------------- |
| F-100 | Push notifications   | Receive notifications when app is closed        |
| F-101 | Background refresh   | Sync messages in background                     |
| F-102 | Offline caching      | Cache messages for offline viewing              |
| F-103 | Battery optimization | Minimize battery drain from background activity |

---

## Optional Features

| ID    | Feature             | Description                        | Notes                            |
| ----- | ------------------- | ---------------------------------- | -------------------------------- |
| F-110 | Message search      | Search through message history     | Requires full-text indexing      |
| F-111 | Message reactions   | React to messages with emoji       | Schema change required           |
| F-112 | Link previews       | Show previews for URLs in messages | Requires URL fetching service    |
| F-113 | File attachments    | Send images and files              | Requires file storage            |
| F-114 | Group conversations | Chat with multiple users           | Significant schema change        |
| F-115 | User avatars        | Profile pictures for users         | Requires image storage           |
| F-116 | Message editing     | Edit sent messages                 | Conflicts with immutability rule |
| F-117 | Message deletion    | Delete sent messages               | Conflicts with immutability rule |

---

## Out of Scope

The following features are explicitly **not** planned for LiteMessage:

| Feature               | Reason                                                   |
| --------------------- | -------------------------------------------------------- |
| End-to-end encryption | Adds significant complexity; focus on core functionality |
| Voice/video calls     | Different technology stack; out of scope for text chat   |
| Multi-device sync     | Single device per platform for MVP                       |
| User blocking         | Social features deferred                                 |
| Message forwarding    | Complexity vs. value tradeoff                            |
| Channels/broadcasts   | Focus is on 1:1 conversations                            |
| Bots/integrations     | Enterprise feature; out of scope                         |
| Admin dashboard       | Operational tooling deferred                             |

---

## Feature-to-Document Mapping

| Feature Area     | Relevant Documentation                             |
| ---------------- | -------------------------------------------------- |
| Authentication   | `002-api-contracts.md` (Auth Endpoints)            |
| Conversations    | `002-api-contracts.md` (Conversations)             |
| Messaging        | `002-api-contracts.md`, `003-message-lifecycle.md` |
| Presence         | `000-core-concepts.md`, `002-api-contracts.md`     |
| Real-time        | `002-api-contracts.md` (WebSocket Events)          |
| Data persistence | `004-database-schema.md`                           |
| State management | `003-message-lifecycle.md`                         |

---

## MVP Checklist

Minimum features required for a functional 1:1 chat application:

- [ ] F-001: User registration
- [ ] F-002: User login
- [ ] F-010: Start conversation
- [ ] F-011: List conversations
- [ ] F-012: View conversation
- [ ] F-020: Send message
- [ ] F-021: Receive message
- [ ] F-022: Message history
- [ ] F-023: Message ordering
- [ ] F-040: WebSocket connection
- [ ] F-041: Auto-reconnect
