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
  - [ ] Define core concepts:
    - What is a "message"?
    - What is a "conversation"?
    - Offline vs online behavior
    - Real-time vs eventual delivery
  - [ ] Finalize all documentation
  - [ ] Define feature list
  - [ ] Define data models
  - [ ] Define platform responsibility boundaries

### Phase 1 - Architecture & Scope Definition

- **Objective:** Define scope, message lifecycle, API contracts, and data models
- **Milestone:** Data types and contracts defined
- **Tasks:**
  - [ ] Backend
    - [ ] Create REST endpoints
    - [ ] Identify WebSocket events
    - [ ] Define auth model
    - [ ] Draft API contracts
  - [ ] Platforms
    - [ ] Create shared interfaces
    - [ ] Define state management
    - [ ] Define message lifecycle
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
