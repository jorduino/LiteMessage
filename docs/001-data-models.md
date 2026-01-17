# 001-Data-Models.md

This document defines the TypeScript type definitions for LiteMessage. These types formalize the core concepts from `000-core-concepts.md` and serve as the source of truth for the backend and all clients.

---

## Enums

```typescript
type MessageStatus = "pending" | "sent" | "delivered" | "read" | "failed";

type PresenceState = "online" | "offline";
```

---

## Core Types

### Presence

```typescript
interface Presence {
  state: PresenceState;
  lastActive: string; // ISO8601
}
```

### User

```typescript
interface User {
  id: string;         // UUID
  displayName: string;
  email: string;
  presence: Presence;
}
```

### Message

```typescript
interface Message {
  id: string;            // UUID
  senderId: string;      // User.id
  conversationId: string; // Conversation.id
  text: string;
  timestamp: string;     // ISO8601, server-assigned
  status: MessageStatus;
}
```

### Conversation

```typescript
interface Conversation {
  id: string;                     // UUID
  participants: [string, string]; // Exactly 2 User.ids (1:1 only)
  lastMessageId: string | null;
  lastReadMessageId: string | null;
}
```

---

## Notes

### Nullability

- `Conversation.lastMessageId` is `null` when a conversation has no messages yet
- `Conversation.lastReadMessageId` is `null` when no messages have been read

### ID Format

- All `id` fields use UUID format

### Timestamps

- All timestamps are ISO8601 strings
- `Message.timestamp` is server-assigned at creation time
- `Presence.lastActive` reflects the last known activity time

### Behavioral Guarantees

See `000-core-concepts.md` for:

- Message immutability rules
- Offline/online behavior and message state transitions
- Delivery guarantees (at-least-once for messages, best-effort for presence)
- Message ordering (server timestamp with message ID tie-breaker)
