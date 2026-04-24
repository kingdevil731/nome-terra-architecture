# Nome Terra

Realtime multiplayer word game built with event-driven architecture.

> Public architecture showcase for a private production project.
> This repository focuses on system design, gameplay architecture and engineering decisions rather than full source distribution.

## Live Application

[Nome Terra(beta)](https://nometerra.kingdevil731.dev)

## Overview

Nome Terra is a multiplayer word game where players join a shared room, vote on game columns, receive a random letter, submit answers under time pressure, review responses, vote invalid entries, and accumulate scores across rounds.

Explores:

- Realtime multiplayer systems
- Event-driven architecture
- Shared state synchronization
- Gameplay and scoring logic
- Interactive product design

## Core Features

- Multiplayer rooms and lobby flow
- Realtime synchronized gameplay rounds
- Column proposal and voting
- Submission and scoring engine
- Review phase with invalid-answer voting
- Shared leaderboards
- Reconnection handling
- Presence synchronization

## High-Level Architecture

Players
↓
Socket.IO Gateway
↓
Room Service
↓
Game Engine
↓
Scoring Logic

## Architectural Principles

### Server-authoritative state

Game progression and scoring coordinated server-side to preserve consistency.

### Event-driven communication

Example events:

- room:create
- room:join
- game:start
- round:submit
- round:voteInvalid
- round:finalize

### Shared game-core separation

Layers:

- Socket transport
- Room orchestration
- Game core / scoring logic

## Gameplay Flow

Create Room
↓
Players Join
↓
Columns Voting
↓
Game Starts
↓
Letter Generated
↓
Timed Submission
↓
Review / Voting
↓
Scoring
↓
Next Round

## Technical Challenges

- Multiplayer state synchronization
- Responsiveness vs correctness
- Duplicate events
- Reconnect recovery
- Race-condition edge cases

## Stack

Backend:

- Node.js
- TypeScript
- Socket.IO
- PostgreSQL
- Redis
- Docker

Frontend:

- Next.js
- React

## Architecture Metrics

- Realtime Architecture: Socket.IO
- State Model: Server-Authoritative
- Multiplayer: Room-Based
- Game Logic: Shared Game Core

## Future Scaling

- Redis adapter for horizontal scaling
- Gameplay analytics
- Persistent room-state strategies
- Matchmaking evolution
