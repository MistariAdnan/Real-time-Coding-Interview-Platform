# Real-Time Collaborative Interview Platform

A live, multi-user coding interview platform. Interviewers author a problem with test cases, candidates join a room and solve it in a synchronized code editor, and submitted code is executed in an isolated Docker sandbox with per-test-case pass/fail grading, all in real time over WebSockets.

**Live Demo:** [real-time-interview-platform-beta.vercel.app](https://real-time-interview-platform-beta.vercel.app/)
**Backend:** [real-time-interview-platform.onrender.com](https://real-time-interview-platform.onrender.com)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Design Decisions](#design-decisions)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Roadmap](#roadmap)

---

## Overview

Most collaborative code editor projects stop at allowing two people to type in the same box. This project goes further. It is built around the actual structure of a technical interview: an interviewer authors a problem with visible and hidden test cases, a candidate is graded against them, and the system enforces who can see what.

The core engineering challenges are not in the user interface. They lie in making real-time state synchronization, untrusted code execution, and role-based data visibility work correctly together.

## Features

- **Real-time collaborative editor:** A Monaco-based editor synchronized across all participants in a room via WebSockets, with debounced updates to avoid flooding the network on every keystroke.
- **Secure, sandboxed code execution:** User-submitted code (JavaScript and Python) runs inside isolated Docker containers with no network access, hard memory and CPU limits, and an execution timeout. Arbitrary code can never reach the host machine or its credentials.
- **Asynchronous execution via job queue:** "Run Code" requests are placed on a Redis-backed queue (BullMQ) and processed by a separate worker process, so the main server never blocks while code executes.
- **Interviewer and candidate roles:** The session creator is the interviewer and the first participant to join is the candidate. Each role sees a purpose-built view of the same room.
- **Problem authoring with hidden test cases:** Interviewers write a problem description along with a mix of visible and hidden test cases. Candidates are graded against all of them but can see only the visible ones.
- **Server-side data sanitization by role:** Hidden test case data (input, expected output, and actual output) is stripped on the server before it reaches a candidate's browser, across both WebSocket payloads and REST responses. It is not merely hidden in the UI.
- **Live presence and cursor tracking:** Participants can see who is in the room and watch their cursors move in real time, throttled to a fixed rate to keep network traffic under control.
- **In-room chat:** Built-in messaging between participants.
- **Session persistence:** Code is auto-saved to MongoDB on a debounce and reloaded automatically when a participant rejoins. A "My Sessions" view shows session history per user.
- **Full authentication:** JWT-based authentication protects both REST endpoints and the WebSocket connection itself. A custom Socket.io middleware verifies the token before a socket is permitted to join any room.
- **End-interview flow:** The interviewer can close a session. The candidate is redirected out, and the interviewer retains a read-only view of the final state.

## Architecture

```
┌─────────────┐      WebSocket / REST       ┌──────────────────┐
│   React     │ ───────────────────────────▶│   Express +      │
│  (Vite,     │◀─────────────────────────── │   Socket.io      │
│  Tailwind)  │                              │   (main server)  │
└─────────────┘                              └────────┬─────────┘
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    ▼                   ▼                   ▼
                              ┌──────────┐        ┌──────────┐       ┌────────────┐
                              │ MongoDB  │        │  Redis   │       │  BullMQ    │
                              │ (users,  │        │ (queue   │       │  Worker    │
                              │ sessions,│        │  backing │       │  (separate │
                              │ problems)│        │  store)  │       │  process)  │
                              └──────────┘        └──────────┘       └─────┬──────┘
                                                                            │
                                                                            ▼
                                                                    ┌───────────────┐
                                                                    │ Docker         │
                                                                    │ (sandboxed     │
                                                                    │ code execution)│
                                                                    └───────────────┘
```

## Design Decisions

### Why a job queue instead of executing code on the main server?

Executing arbitrary code takes time and can hang. If the main Node.js server ran submissions directly, a single slow or malicious submission would block the event loop for every connected user, including chat and cursor synchronization. Instead, the server enqueues a job and immediately returns to serving requests. A separate worker process picks up the job, runs it in Docker, and the result is pushed back to the appropriate room over WebSocket once it is ready.

### Why Docker for code execution?

Running untrusted, user-submitted code with `eval` or a raw subprocess call would give that code full access to the host, including its filesystem, environment variables, and network. Instead, each submission runs in a disposable container with `--network none` (no network access), hard memory and CPU limits, and a strict timeout. A malicious or faulty submission cannot exfiltrate data, exhaust resources, or run indefinitely.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React (Vite), Tailwind CSS, Monaco Editor |
| Backend | Node.js, Express, Socket.io |
| Database | MongoDB (Mongoose) |
| Queue / Cache | Redis (Upstash), BullMQ |
| Code Execution | Docker (isolated container per submission) |
| Authentication | JWT (custom middleware for both REST and Socket.io) |
| Deployment | Vercel (frontend), Render (backend) |

## Getting Started

### Prerequisites

- Node.js
- Docker Desktop
- A MongoDB connection string
- A Redis connection string

### Installation

```bash
# Clone the repository
git clone <repo-url>
cd real-time-interview-platform

# Backend
cd server
npm install
# Create a .env file (see Environment Variables below)
npm run dev      # Starts the main server
npm run worker   # Run in a separate terminal; starts the execution worker

# Frontend
cd ../client
npm install
npm run dev
```

Three processes must run simultaneously during development: the main server, the execution worker, and the frontend dev server.

### Environment Variables

Create `server/.env` with the following values:

```
MONGO_URI=
JWT_SECRET=
REDIS_HOST=
REDIS_PORT=
REDIS_PASSWORD=
```

## Roadmap

- Support for additional languages, including compiled languages such as Java and C++, which require a separate compile step before execution
- Rate limiting on code execution to prevent queue abuse
- WebRTC-based voice and video for a fully self-contained interview experience
- Automated tests for the sanitization layer, as it is the highest-stakes correctness boundary in the application

---

*Built as a solo project to explore real-time systems, background job processing, and sandboxed code execution in depth, rather than simply to ship a user interface.*
