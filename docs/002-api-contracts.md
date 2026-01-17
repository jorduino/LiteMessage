# 002-API-Contracts.md

This document defines the REST endpoints and WebSocket events for LiteMessage. All types reference definitions from `001-data-models.md`, and behavioral requirements are derived from `000-core-concepts.md`.

---

## Authentication

LiteMessage uses stateless JWT authentication. Tokens are passed via the `Authorization` header for REST requests and as a query parameter for WebSocket connections.

### Token Format

```typescript
interface JWTPayload {
  userId: string;  // User.id
  exp: number;     // Expiration timestamp (Unix)
}
```

### REST Authentication

All authenticated endpoints require:

```text
Authorization: Bearer <token>
```

### WebSocket Authentication

Connect with token as query parameter:

```text
wss://api.example.com/ws?token=<token>
```

---

## Response Format

All REST responses use a consistent envelope structure.

### Success Response

```typescript
interface SuccessResponse<T> {
  data: T;
  error: null;
}
```

### Error Response

```typescript
interface ErrorResponse {
  data: null;
  error: {
    code: string;    // e.g., "NOT_FOUND", "UNAUTHORIZED", "VALIDATION_ERROR"
    message: string; // Human-readable description
  };
}
```

### Common Error Codes

| Code               | HTTP Status | Description                              |
| ------------------ | ----------- | ---------------------------------------- |
| `UNAUTHORIZED`     | 401         | Missing or invalid token                 |
| `FORBIDDEN`        | 403         | Valid token but insufficient permissions |
| `NOT_FOUND`        | 404         | Resource does not exist                  |
| `VALIDATION_ERROR` | 400         | Invalid request body or parameters       |
| `CONFLICT`         | 409         | Resource already exists                  |
| `INTERNAL_ERROR`   | 500         | Server error                             |

---

## Pagination

Message history uses cursor-based pagination for efficient traversal.

### Request Parameters

| Parameter | Type   | Default | Description                           |
| --------- | ------ | ------- | ------------------------------------- |
| `cursor`  | string | (none)  | Message ID to start after (exclusive) |
| `limit`   | number | 50      | Max messages to return (1-100)        |

### Paginated Response

```typescript
interface PaginatedResponse<T> {
  data: {
    items: T[];
    nextCursor: string | null; // null when no more results
  };
  error: null;
}
```

---

## REST Endpoints

### Auth Endpoints

#### POST /auth/register

Create a new user account.

**Request Body:**

```typescript
{
  email: string;
  password: string;
  displayName: string;
}
```

**Response:** `SuccessResponse<{ user: User; token: string }>`

**Errors:**

- `CONFLICT` - Email already registered
- `VALIDATION_ERROR` - Invalid email format or weak password

---

#### POST /auth/login

Authenticate and receive a token.

**Request Body:**

```typescript
{
  email: string;
  password: string;
}
```

**Response:** `SuccessResponse<{ user: User; token: string }>`

**Errors:**

- `UNAUTHORIZED` - Invalid credentials

---

#### POST /auth/logout

Invalidate the current session. While JWTs are stateless, this endpoint can be used for server-side token blocklisting if implemented.

**Headers:** `Authorization: Bearer <token>`

**Response:** `SuccessResponse<{}>`

---

### Users

#### GET /users/me

Get the current authenticated user's profile.

**Headers:** `Authorization: Bearer <token>`

**Response:** `SuccessResponse<User>`

---

#### GET /users/:id

Get a user by ID. Returns limited public information.

**Headers:** `Authorization: Bearer <token>`

**Response:**

```typescript
SuccessResponse<{
  id: string;
  displayName: string;
  presence: Presence;
}>
```

**Errors:**

- `NOT_FOUND` - User does not exist

---

#### GET /users?email=

Search for a user by email address. Used for starting new conversations.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**

- `email` (required) - Exact email to search for

**Response:**

```typescript
SuccessResponse<{
  id: string;
  displayName: string;
  presence: Presence;
} | null>
```

Returns `null` in data field if no user found (not an error).

---

### Conversations

#### GET /conversations

List all conversations for the authenticated user.

**Headers:** `Authorization: Bearer <token>`

**Response:**

```typescript
SuccessResponse<{
  id: string;
  participants: [string, string];
  lastMessageId: string | null;
  lastReadMessageId: string | null;
  lastMessage: Message | null;      // Hydrated for display
  otherParticipant: {               // Convenience field
    id: string;
    displayName: string;
    presence: Presence;
  };
}[]>
```

---

#### POST /conversations

Create a new conversation with another user.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**

```typescript
{
  participantId: string;  // User.id of the other participant
}
```

**Response:** `SuccessResponse<Conversation>`

**Errors:**

- `NOT_FOUND` - Participant user does not exist
- `CONFLICT` - Conversation already exists with this participant
- `VALIDATION_ERROR` - Cannot create conversation with self

---

#### GET /conversations/:id

Get a single conversation by ID.

**Headers:** `Authorization: Bearer <token>`

**Response:** `SuccessResponse<Conversation>`

**Errors:**

- `NOT_FOUND` - Conversation does not exist
- `FORBIDDEN` - User is not a participant

---

#### GET /conversations/:id/messages

Get paginated message history for a conversation.

**Headers:** `Authorization: Bearer <token>`

**Query Parameters:**

- `cursor` (optional) - Message ID to start after
- `limit` (optional) - Number of messages (default: 50, max: 100)

**Response:**

```typescript
PaginatedResponse<Message>
```

Messages are returned in reverse chronological order (newest first). Use `nextCursor` to fetch older messages.

**Errors:**

- `NOT_FOUND` - Conversation does not exist
- `FORBIDDEN` - User is not a participant

---

### Messages

#### POST /conversations/:id/messages

Send a new message to a conversation.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**

```typescript
{
  text: string;
  clientMessageId?: string;  // Optional client-generated ID for deduplication
}
```

**Response:** `SuccessResponse<Message>`

The returned message will have:

- Server-assigned `id` and `timestamp`
- Initial `status` of `"sent"`

**Errors:**

- `NOT_FOUND` - Conversation does not exist
- `FORBIDDEN` - User is not a participant
- `VALIDATION_ERROR` - Empty message text

---

#### PATCH /conversations/:id/read

Mark messages as read up to a specific message.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**

```typescript
{
  messageId: string;  // Mark all messages up to and including this one as read
}
```

**Response:** `SuccessResponse<{ lastReadMessageId: string }>`

**Errors:**

- `NOT_FOUND` - Conversation or message does not exist
- `FORBIDDEN` - User is not a participant

---

## WebSocket Events

The WebSocket connection provides real-time updates for messages, presence, and typing indicators. Connect using `wss://api.example.com/ws?token=<jwt>`.

### Connection Lifecycle

1. Client connects with valid JWT token
2. Server authenticates and acknowledges connection
3. Client subscribes to conversations
4. Bidirectional events flow until disconnect

---

### Client → Server Events

#### subscribe

Subscribe to real-time updates for specific conversations.

```typescript
{
  event: "subscribe";
  data: {
    conversationIds: string[];
  };
}
```

**Response Event:**

```typescript
{
  event: "subscribed";
  data: {
    conversationIds: string[];
  };
}
```

---

#### typing

Indicate that the user is typing in a conversation.

```typescript
{
  event: "typing";
  data: {
    conversationId: string;
    isTyping: boolean;
  };
}
```

Delivery is best-effort. Clients should send `isTyping: false` when typing stops or after a timeout.

---

#### presence

Heartbeat to maintain online presence.

```typescript
{
  event: "presence";
  data: {
    state: "online";
  };
}
```

Clients should send this event every 10 seconds. If the server does not receive a heartbeat within 30 seconds, the user is marked as offline.

---

### Server → Client Events

#### message:new

A new message was received in a subscribed conversation.

```typescript
{
  event: "message:new";
  data: Message;
}
```

---

#### message:status

A message's status was updated (e.g., delivered or read).

```typescript
{
  event: "message:status";
  data: {
    messageId: string;
    conversationId: string;
    status: MessageStatus;
  };
}
```

---

#### presence:update

A user's presence state changed.

```typescript
{
  event: "presence:update";
  data: {
    userId: string;
    presence: Presence;
  };
}
```

Delivery is best-effort per `000-core-concepts.md`.

---

#### typing:update

A user started or stopped typing in a conversation.

```typescript
{
  event: "typing:update";
  data: {
    conversationId: string;
    userId: string;
    isTyping: boolean;
  };
}
```

Delivery is best-effort. Clients should implement a timeout to clear stale typing indicators.

---

#### error

Server-side error related to the WebSocket connection or a client event.

```typescript
{
  event: "error";
  data: {
    code: string;
    message: string;
    originalEvent?: string;  // The event that caused the error, if applicable
  };
}
```

---

## Summary

| Category      | Endpoint/Event                | Method | Description              |
| ------------- | ----------------------------- | ------ | ------------------------ |
| Auth          | `/auth/register`              | POST   | Create account           |
| Auth          | `/auth/login`                 | POST   | Login, get token         |
| Auth          | `/auth/logout`                | POST   | Invalidate session       |
| Users         | `/users/me`                   | GET    | Current user profile     |
| Users         | `/users/:id`                  | GET    | Get user by ID           |
| Users         | `/users?email=`               | GET    | Search user by email     |
| Conversations | `/conversations`              | GET    | List conversations       |
| Conversations | `/conversations`              | POST   | Create conversation      |
| Conversations | `/conversations/:id`          | GET    | Get conversation         |
| Conversations | `/conversations/:id/messages` | GET    | Get messages (paginated) |
| Messages      | `/conversations/:id/messages` | POST   | Send message             |
| Messages      | `/conversations/:id/read`     | PATCH  | Mark as read             |
| WebSocket     | `subscribe`                   | C→S    | Join conversation rooms  |
| WebSocket     | `typing`                      | C→S    | Typing indicator         |
| WebSocket     | `presence`                    | C→S    | Heartbeat ping           |
| WebSocket     | `message:new`                 | S→C    | New message              |
| WebSocket     | `message:status`              | S→C    | Status update            |
| WebSocket     | `presence:update`             | S→C    | Presence changed         |
| WebSocket     | `typing:update`               | S→C    | Typing indicator         |
