# 004-Database-Schema.md

This document defines the PostgreSQL database schema for LiteMessage. The schema implements the data models from `001-data-models.md` and supports the API operations defined in `002-api-contracts.md`.

---

## Design Decisions

- **Database**: PostgreSQL (for UUID support, JSONB if needed, robust indexing)
- **IDs**: UUID v4 for all primary keys
- **Timestamps**: `TIMESTAMPTZ` for all time fields (stored in UTC)
- **Soft deletes**: Not implemented (messages are immutable and not deletable per core concepts)
- **Presence**: Stored in users table (single source of truth, updated via WebSocket heartbeats)

---

## Schema

### Enums

```sql
CREATE TYPE presence_state AS ENUM ('online', 'offline');

CREATE TYPE message_status AS ENUM ('pending', 'sent', 'delivered', 'read', 'failed');
```

---

### Users

Stores user accounts and presence information.

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    presence_state  presence_state NOT NULL DEFAULT 'offline',
    last_active     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users (email);
```

**Notes:**

- `email` has a unique constraint for authentication lookups
- `password_hash` stores bcrypt or argon2 hashed passwords
- `presence_state` and `last_active` together form the `Presence` type from data models

---

### Conversations

Stores 1:1 conversations between two users.

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_1   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    participant_2   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT conversations_participants_ordered
        CHECK (participant_1 < participant_2),
    CONSTRAINT conversations_unique_participants
        UNIQUE (participant_1, participant_2)
);

CREATE INDEX idx_conversations_participant_1 ON conversations (participant_1);
CREATE INDEX idx_conversations_participant_2 ON conversations (participant_2);
```

**Notes:**

- `participant_1 < participant_2` constraint ensures canonical ordering, preventing duplicate conversations
- To find all conversations for a user, query both participant columns
- The unique constraint on `(participant_1, participant_2)` enforces one conversation per user pair

---

### Messages

Stores all messages in conversations.

```sql
CREATE TABLE messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id     UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    text                TEXT NOT NULL,
    timestamp           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    status              message_status NOT NULL DEFAULT 'sent',
    client_message_id   VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT messages_text_not_empty CHECK (LENGTH(TRIM(text)) > 0)
);

CREATE INDEX idx_messages_conversation_timestamp
    ON messages (conversation_id, timestamp DESC);

CREATE INDEX idx_messages_conversation_id
    ON messages (conversation_id);

CREATE UNIQUE INDEX idx_messages_client_dedup
    ON messages (conversation_id, sender_id, client_message_id)
    WHERE client_message_id IS NOT NULL;
```

**Notes:**

- `timestamp` is server-assigned at creation time (not client-provided)
- `client_message_id` enables client-side deduplication for retry scenarios
- Composite index on `(conversation_id, timestamp DESC)` supports paginated message queries
- The partial unique index on `client_message_id` prevents duplicate messages from retries

---

### Conversation Read Receipts

Tracks the last read message per user per conversation.

```sql
CREATE TABLE conversation_read_receipts (
    conversation_id     UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    last_read_message_id UUID REFERENCES messages(id) ON DELETE SET NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (conversation_id, user_id)
);
```

**Notes:**

- Maps to `Conversation.lastReadMessageId` from data models (per-user)
- `ON DELETE SET NULL` handles edge case where a message is somehow removed
- One row per participant per conversation

---

## Derived Fields

Some fields from the data models are computed rather than stored:

| Data Model Field                 | Source                                                                                                   |
| -------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `Conversation.lastMessageId`     | `SELECT id FROM messages WHERE conversation_id = ? ORDER BY timestamp DESC LIMIT 1`                      |
| `Conversation.lastReadMessageId` | `SELECT last_read_message_id FROM conversation_read_receipts WHERE conversation_id = ? AND user_id = ?`  |
| `User.presence`                  | Composed from `users.presence_state` and `users.last_active`                                             |

---

## Common Queries

### Get conversations for a user

```sql
SELECT c.*,
       m.id AS last_message_id,
       m.text AS last_message_text,
       m.timestamp AS last_message_timestamp,
       crr.last_read_message_id
FROM conversations c
LEFT JOIN LATERAL (
    SELECT id, text, timestamp
    FROM messages
    WHERE conversation_id = c.id
    ORDER BY timestamp DESC
    LIMIT 1
) m ON true
LEFT JOIN conversation_read_receipts crr
    ON crr.conversation_id = c.id AND crr.user_id = $1
WHERE c.participant_1 = $1 OR c.participant_2 = $1
ORDER BY COALESCE(m.timestamp, c.created_at) DESC;
```

### Get paginated messages (cursor-based)

```sql
-- First page (no cursor)
SELECT * FROM messages
WHERE conversation_id = $1
ORDER BY timestamp DESC
LIMIT $2;

-- Subsequent pages (with cursor)
SELECT * FROM messages
WHERE conversation_id = $1
  AND (timestamp, id) < (
      SELECT timestamp, id FROM messages WHERE id = $3
  )
ORDER BY timestamp DESC
LIMIT $2;
```

### Find existing conversation between two users

```sql
SELECT * FROM conversations
WHERE participant_1 = LEAST($1, $2)
  AND participant_2 = GREATEST($1, $2);
```

### Mark messages as read

```sql
INSERT INTO conversation_read_receipts (conversation_id, user_id, last_read_message_id, updated_at)
VALUES ($1, $2, $3, NOW())
ON CONFLICT (conversation_id, user_id)
DO UPDATE SET
    last_read_message_id = EXCLUDED.last_read_message_id,
    updated_at = NOW();
```

### Update user presence

```sql
UPDATE users
SET presence_state = $2,
    last_active = NOW(),
    updated_at = NOW()
WHERE id = $1;
```

---

## Indexes Summary

| Table                        | Index                                           | Purpose                          |
| ---------------------------- | ----------------------------------------------- | -------------------------------- |
| `users`                      | `idx_users_email`                               | Login lookup                     |
| `conversations`              | `idx_conversations_participant_1`               | Find user's conversations        |
| `conversations`              | `idx_conversations_participant_2`               | Find user's conversations        |
| `messages`                   | `idx_messages_conversation_timestamp`           | Paginated message retrieval      |
| `messages`                   | `idx_messages_conversation_id`                  | Message count, existence checks  |
| `messages`                   | `idx_messages_client_dedup` (partial, unique)   | Client-side deduplication        |

---

## Migration Notes

- Run enum creation before table creation
- Foreign key constraints require referenced tables to exist first
- Order: enums → users → conversations → messages → conversation_read_receipts
