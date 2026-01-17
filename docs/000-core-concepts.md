# 000-Core-Concepts.md

This document defines the core domain concepts of LiteMessage. Each section follows a structured format to clearly communicate definitions, attributes, and architectural decisions.

---

## Message

**Definition:**  
A single text message sent between users within a conversation.

**Attributes:**

* `id`: string - unique identifier for the message
* `senderId`: string - user who sent the message
* `conversationId`: string - conversation this message belongs to
* `text`: string - message content
* `timestamp`: ISO8601 - server-assigned creation time
* `status`: enum(`pending`, `sent`, `delivered`, `read`, `failed`) - message lifecycle state

**Decisions / Notes:**

* Messages are immutable after sending
* Messages sent while offline will fail locally and require user action

---

## Conversation

**Definition:** A 1:1 chat between two users.

**Attributes:**

* `id`: string - unique identifier for the conversation
* `participants`: array of `User.id` - users in the conversation
* `lastMessageId`: string - most recent message in the conversation
* `lastReadMessageId`: string - most recent message read by the current user

---

## User

**Definition:**  
An individual using the chat application.

**Attributes:**

* `id`: string - unique user identifier
* `displayName`: string - visible display name
* `email`: string - used for authentication
* `presence`: `Presence` - current presence information

---

## Presence

**Definition:**  
Online/offline status of a user.

**Attributes:**

* `state`: enum(`online`, `offline`) - current presence state
* `lastActive`: timestamp - last known activity time

**Decisions / Notes:**

* Presence updates are broadcast every 10 seconds
* Presence is best-effort and not guaranteed to be accurate in real time

---

## Offline vs Online Behavior

**Definition:**  
Rules governing client behavior when network connectivity is lost or unstable.

**Decisions / Notes:**

* Messages do not leave the input composer while the client is offline
* If a client disconnects while a message is in `pending` state, the message transitions to `failed`
* Upon reconnection, the client syncs with the server to retrieve any messages that were successfully delivered
* Messages that failed locally but succeeded on the server are reconciled and updated
* Messages that failed both locally and on the server present retry or cancel options to the user

---

## Real-time vs Eventual Delivery

**Definition:**  
Delivery and ordering guarantees for real-time communication.

**Decisions / Notes:**

* Message delivery guarantee: **at-least-once**
* Presence and typing indicator delivery guarantee: **best-effort**
* Messages are ordered using server-assigned timestamps with message ID as a tie-breaker
