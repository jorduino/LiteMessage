# 000-Core-Concepts.md

This document defines the core concepts of LiteMessage. Each section follows a structured format to clearly communicate definitions, attributes, and decisions.

---

## Message

**Definition:** A single text message sent between users.

**Attributes:**

* `id`: string - unique identifier for the message
* `senderId`: string - user who sent the message
* `conversationId`: string - the conversation this message belongs to
* `text`: string - message content
* `timestamp`: ISO8601 - time message was created
* `status`: enum(pending, sent, delivered, read, failed) - message lifecycle state

**Decisions / Notes:**

* Messages are immutable after sending
* Messages sent while offline will soft fail

---

## Conversation

**Definition:** A 1:1 chat between two users.

**Attributes:**

* `id`: string - unique identifier for the conversation
* `participants`: array of `User.id` - users in the conversation
* `lastMessageId`: string - id of the most recent message
* `lastReadMessageId`: string - id of the most recent read message

---

## User

**Definition:** An individual using the chat application.

**Attributes:**

* `id`: string - unique user identifier
* `displayName`: string - visible name
* `email`: string - for login
* `status`: presence object - presence state

---

## Presence

**Definition:** Online/offline status of a user.

**Attributes:**

* `state`: enum(online, offline) - current presence
* `lastActive`: timestamp - last known activity

**Decisions / Notes:**

* Frequency of presence updates: 10s

---

## Offline vs Online Behavior

**Definition:** How the system behaves when clients lose connectivity.

**Decisions / Notes:**

* Messages will not leave the input composer while offline
* If agent disconnects while message in "pending" status, message will fail with option to rectify upon reconnection
* Upon reconnection, conversation will sync any messages that may have successfully sent
* Messages that failed locally but succeeded on the server will be updated
* Messages that failed locally and on the server will have option to retry or cancel

---

## Real-time vs Eventual Delivery

**Definition:** Guarantees around sending and receiving messages.

**Decisions / Notes:**

* Message delivery guarantee: at-least-once
* Presence and typing indicator delivery guarantee: best effort
* Order messages by server assigned timestamps with message id tiebreaker
