# 003-Message-Lifecycle.md

This document defines the complete lifecycle of a message in LiteMessage, including state transitions, client/server responsibilities, and error handling. It consolidates behavioral requirements from `000-core-concepts.md` and maps them to the API contracts in `002-api-contracts.md`.

---

## Message States

Messages progress through the following states (defined in `001-data-models.md`):

```text
pending → sent → delivered → read
                    ↘
              failed (terminal)
```

| State       | Description                                      | Set By  |
| ----------- | ------------------------------------------------ | ------- |
| `pending`   | Message created locally, not yet sent to server  | Client  |
| `sent`      | Server received and persisted the message        | Server  |
| `delivered` | Message delivered to recipient's device          | Server  |
| `read`      | Recipient has viewed the message                 | Server  |
| `failed`    | Message could not be sent                        | Client  |

---

## State Transitions

### Happy Path

```text
┌──────────┐    POST /messages    ┌──────────┐
│ pending  │ ──────────────────── │   sent   │
└──────────┘     (success)        └──────────┘
                                       │
                                       │ WebSocket: message:status
                                       ▼
                                  ┌──────────┐
                                  │delivered │
                                  └──────────┘
                                       │
                                       │ WebSocket: message:status
                                       ▼
                                  ┌──────────┐
                                  │   read   │
                                  └──────────┘
```

### Error Path

```text
┌──────────┐    POST /messages    ┌──────────┐
│ pending  │ ──────────────────── │  failed  │
└──────────┘     (error/timeout)  └──────────┘
                                       │
                                       │ User action: retry
                                       ▼
                                  ┌──────────┐
                                  │ pending  │ (retry attempt)
                                  └──────────┘
```

---

## Detailed State Descriptions

### Pending

**Trigger:** User submits a message in the UI.

**Client responsibilities:**

- Generate a `clientMessageId` for deduplication
- Display the message in the conversation with a pending indicator
- Attempt to send via `POST /conversations/:id/messages`

**Constraints:**

- Client must be online to transition out of this state
- If client goes offline while pending, transition to `failed`

---

### Sent

**Trigger:** Server responds with `2xx` to the POST request.

**Server responsibilities:**

- Assign `id` and `timestamp` to the message
- Persist to database
- Broadcast `message:new` event to recipient via WebSocket

**Client responsibilities:**

- Replace `clientMessageId` with server-assigned `id`
- Update UI to show sent indicator
- Store server-assigned `timestamp` for ordering

---

### Delivered

**Trigger:** Recipient's client receives the `message:new` WebSocket event.

**Server responsibilities:**

- Track that recipient's WebSocket connection received the message
- Broadcast `message:status` event with `status: "delivered"` to sender

**Client responsibilities (sender):**

- Update UI to show delivered indicator (e.g., single checkmark)

**Notes:**

- Delivery confirmation requires recipient to have an active WebSocket connection
- If recipient is offline, message remains in `sent` state until they reconnect

---

### Read

**Trigger:** Recipient calls `PATCH /conversations/:id/read` with the message ID.

**Server responsibilities:**

- Update `conversation_read_receipts` table
- Broadcast `message:status` event with `status: "read"` to sender

**Client responsibilities (recipient):**

- Call the read endpoint when messages become visible in viewport
- Debounce/batch read receipts to avoid excessive API calls

**Client responsibilities (sender):**

- Update UI to show read indicator (e.g., double checkmark)

---

### Failed

**Trigger:** Any of the following:

- Network error during POST request
- Server returns error response
- Request timeout
- Client goes offline while message is `pending`

**Client responsibilities:**

- Display error indicator on the message
- Provide retry and delete options to user
- Preserve message text for retry attempts

**User actions available:**

- **Retry:** Transition back to `pending`, attempt to send again
- **Delete:** Remove message from local state (never reached server)

---

## Offline Behavior

Per `000-core-concepts.md`, offline handling follows these rules:

### Sending While Offline

1. User types message but client detects offline state
2. Message is **not** created in `pending` state
3. Send button is disabled or shows offline indicator
4. Message remains in composer until connectivity is restored

### Disconnect During Pending

1. Message is in `pending` state, awaiting server response
2. Client detects connection loss
3. Message transitions to `failed`
4. User can retry when back online

### Reconnection Sync

1. Client reconnects to WebSocket
2. Client fetches recent messages via `GET /conversations/:id/messages`
3. Reconcile local state with server state:
   - Messages that failed locally but exist on server → update to `sent`
   - Messages marked `sent` locally that are `delivered`/`read` on server → update status
   - New messages from other participant → add to local state

---

## Deduplication

To handle retry scenarios and network issues, clients use `clientMessageId`:

```typescript
// Client generates before sending
const clientMessageId = crypto.randomUUID();

// Included in POST request
POST /conversations/:id/messages
{
  "text": "Hello",
  "clientMessageId": "abc-123-def"
}
```

**Server behavior:**

- If a message with the same `(conversation_id, sender_id, client_message_id)` exists, return the existing message instead of creating a duplicate
- This ensures at-least-once delivery without duplicates

---

## WebSocket Events

The following events drive status transitions (from `002-api-contracts.md`):

### message:new (Server → Client)

Received by the recipient when a new message arrives:

```typescript
{
  event: "message:new",
  data: Message  // Full message object with status: "sent"
}
```

### message:status (Server → Client)

Received by the sender when message status changes:

```typescript
{
  event: "message:status",
  data: {
    messageId: string,
    conversationId: string,
    status: "delivered" | "read"
  }
}
```

---

## Client State Management

### Recommended Local State Structure

```typescript
interface LocalMessage {
  // Server fields (from Message type)
  id: string | null;           // null while pending
  senderId: string;
  conversationId: string;
  text: string;
  timestamp: string | null;    // null while pending, use local time for sorting
  status: MessageStatus;

  // Client-only fields
  clientMessageId: string;     // For deduplication
  localTimestamp: string;      // For optimistic ordering
  retryCount: number;          // Track retry attempts
}
```

### Optimistic Updates

1. **On send:** Immediately add message to UI with `pending` status
2. **On success:** Update with server-assigned `id` and `timestamp`
3. **On failure:** Show error state, preserve for retry

### Ordering

Messages are ordered by:

1. `timestamp` (server-assigned) - primary
2. `id` (server-assigned) - tie-breaker

For pending messages without server timestamps, use `localTimestamp` for display ordering until server response arrives.

---

## Error Handling

### Transient Errors (Retry Appropriate)

| Error | Handling |
| ----- | -------- |
| Network timeout | Auto-retry with exponential backoff (max 3 attempts) |
| 5xx server error | Auto-retry with exponential backoff |
| WebSocket disconnect | Queue status updates, process on reconnect |

### Permanent Errors (No Retry)

| Error | Handling |
| ----- | -------- |
| 400 Validation error | Show error to user, allow edit and retry |
| 403 Forbidden | Show error, user not in conversation |
| 404 Not found | Show error, conversation deleted |

### Retry Backoff

```typescript
const backoffMs = Math.min(1000 * Math.pow(2, retryCount), 30000);
// Attempt 1: 1s, Attempt 2: 2s, Attempt 3: 4s, max: 30s
```

---

## Sequence Diagrams

### Successful Message Send

```text
Client A          Server           Client B
    │                │                │
    │ POST /messages │                │
    │ ─────────────► │                │
    │                │                │
    │ 200 {message}  │                │
    │ ◄───────────── │                │
    │                │                │
    │ status: sent   │ WS: message:new│
    │                │ ─────────────► │
    │                │                │
    │                │                │ (message displayed)
    │                │ ◄───────────── │
    │                │   (delivered)  │
    │                │                │
    │ WS: status     │                │
    │ ◄───────────── │                │
    │ (delivered)    │                │
    │                │                │
    │                │ PATCH /read    │
    │                │ ◄───────────── │
    │                │                │
    │ WS: status     │                │
    │ ◄───────────── │                │
    │ (read)         │                │
```

### Failed Message with Retry

```text
Client A          Server
    │                │
    │ POST /messages │
    │ ─────────────► │
    │                │
    │ (timeout)      │
    │ ◄ ─ ─ ─ ─ ─ ─  │
    │                │
    │ status: failed │
    │                │
    │ (user clicks   │
    │  retry)        │
    │                │
    │ POST /messages │
    │ (same clientId)│
    │ ─────────────► │
    │                │
    │ 200 {message}  │
    │ ◄───────────── │
    │                │
    │ status: sent   │
```
