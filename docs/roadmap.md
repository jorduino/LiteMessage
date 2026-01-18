# LiteMessage Roadmap

## Vision & Strategy

- **Goal:** Create a fully functional chat application on Web, Desktop, and Mobile with seamless cross-platform communication.
- **Key Metric:** Achieve a fully functional 1:1 chat application with shared core logic across all platforms.

## Themes & Milestones

### Phase 0 - Product Definition

- **Objective:** Finalize project decisions and documentation
- **Milestone:** Documentation completed
- **Tasks:**
  - [x] Finalize roadmap
  - [x] Define core concepts: *(see [core concepts](./000-core-concepts.md))*
    - What is a "message"?
    - What is a "conversation"?
    - Offline vs online behavior
    - Real-time vs eventual delivery
  - [ ] Finalize all documentation
  - [x] Define feature list *(see [feature list](./005-feature-list.md))*
  - [x] Define data models *(see [data models](./001-data-models.md))*
  - [x] Define platform responsibility boundaries *(see [platform boundaries](./006-platform-boundaries.md))*

### Phase 1 - Architecture & Scope Definition

- **Objective:** Define scope, message lifecycle, API contracts, and data models
- **Milestone:** Data types and contracts defined
- **Tasks:**
  - [x] Backend *(see [api contracts](./002-api-contracts.md), [database schema](./004-database-schema.md))*
    - [x] Create REST endpoints
    - [x] Identify WebSocket events
    - [x] Define auth model
    - [x] Draft API contracts
  - [ ] Platforms
    - [ ] Create shared interfaces
    - [x] Define state management *(see [message lifecycle](./003-message-lifecycle.md))*
    - [x] Define message lifecycle *(see [message lifecycle](./003-message-lifecycle.md))*
    - [ ] Define platform boundaries

### Phase 2 - Backend

- **Objective:** Implement server backend
- **Milestone:** Backend API operational
- **Tasks:**
  - [ ] Implement user authorization
  - [ ] Implement conversation creation
  - [ ] Implement message persistence
  - [ ] Set up WebSocket server

### Phase 3 - Shared Core Library

- **Objective:** Implement shared core functionality
- **Milestone:** Core logic functional
- **Tasks:**
  - [ ] Implement sending and receiving messages
  - [ ] Implement automatic reconnection
  - [ ] Implement message pagination

### Phase 4 - Web Client

- **Objective:** Implement web client
- **Milestone:** Web client functional
- **Tasks:**
  - [ ] Implement conversation list
  - [ ] Implement message list
  - [ ] Implement input composer
  - [ ] Implement online presence indicator

### Phase 5 - Desktop Client

- **Objective:** Implement desktop client
- **Milestone:** Desktop client functional
- **Tasks:**
  - [ ] Wrap web client in Electron
  - [ ] Implement system tray minimization
  - [ ] Implement notifications
  - [ ] Set up IPC hooks

### Phase 6 - Mobile Client

- **Objective:** Implement mobile client
- **Milestone:** Mobile client functional
- **Tasks:**
  - [ ] Implement UI
  - [ ] Implement background refresh
  - [ ] Implement push notifications
  - [ ] Implement offline caching
  - [ ] Optimize for battery usage

### Phase 7 - Cross-Platform Polish

- **Objective:** Finalize UI and UX across platforms
- **Milestone:** UI consistency achieved
- **Tasks:**
  - [ ] Ensure consistent message ordering
  - [ ] Implement read receipts *(optional/advanced)*
  - [ ] Implement typing indicators *(optional/advanced)*
  - [ ] Improve reconnection UX

### Phase 8 - Final Testing

- **Objective:** Complete testing and QA
- **Milestone:** All tests passed
- **Tasks:**
  - [ ] Test network dropouts
  - [ ] Handle duplicate messages
  - [ ] Verify clock skew handling
  - [ ] Test race conditions
  - [ ] Perform load testing

## Legend/Key

- `[ ]`: To Do / Not Started
- `[x]`: Complete
- `@username`: Owner/Contact
- `#issue-ID`: Link to detailed issue/ticket
