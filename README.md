# LiteMessage Learning Repo

## Project Overview

This is a chat application that will have direct message capabilities. The purpose of making the project is to learn full stack development from all aspects by taking on every role required to make such an application. I am using the guidance of ChatGPT for role responsibilities and roadmap, but all of the code shall be human generated.

## Responsibilities

The following outlines what responsibilities should be completed and which type of position would typically fulfill them. My goal is that I would be able to do all of them by myself

- **Product**
  - Product Manager (PM): requirements, scope control
  - Designer (UX/UI): flows, interaction models

- **Frontend**
  - Web Engineer (React)
  - Mobile Engineer (React Native)
  - Desktop Engineer (Electron)

- **Backend**
  - Backend Engineer: APIs, auth, messaging
  - Infrastructure Engineer: scaling, deployments

- **Platform**
  - Shared Core Engineer: state, data models, networking
  - QA Engineer: testing, edge cases

## Platforms

The goal is that there will be a web version first, followed by a desktop version, and finally a mobile version. Each should look the same or similar enough, and each of them should be able to communicate with each other seamlessly.

## Architecture

This project will be built on NodeJS with React used between the 3 platforms. NextJS will be used for the web app, Electron for desktop, and ReactNative for mobile.

## Data Models

See [Data Models](./docs/001-data-models.md)

## API Contracts

See [Api Contracts](./docs/002-api-contracts.md)

## Roadmap

See [Roadmap](./docs/roadmap.md)

## Core Features

The app will have the following features:

- 1:1 chats only
- Text messages only
- Login via email
- Presence (online/offline)
- Message history

## Non-Goals / Exclusions

This app will intentionally exclude the following features:

- Groups
- Voice
- Media uploads
- Search

## Installation Instructions

TODO

## Contributing

Feel free to fork the project, make changes, or publish your own versions, but this project is more about learning than making a viable chat application.
