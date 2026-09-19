# PollFlow

Real-Time Polling Application — built for the GUVI × HCL Developer Internship task.

Create a poll, share one link, and watch results update live as votes come in — no refresh needed.

## Features

- JWT + bcrypt authentication for poll owners (voting itself stays public and link-based)
- Authenticated poll creation and ownership checks (only the owner can close their poll)
- Public voting with both browser-level *and* backend-enforced duplicate prevention
- MongoDB persistence for users, polls, and a per-vote audit log
- Redis doing real work: atomic vote counters (`HINCRBY`), atomic duplicate-vote checks (`SADD`), and Pub/Sub fan-out into Server-Sent Events
- Live result bars that update without a refresh, for every open browser tab watching a poll
- Responsive dashboard for managing polls, and a clean public voting/results page

## Technology Stack

React (Vite) · Go (Gin) · MongoDB · Redis · Server-Sent Events · JWT · bcrypt

## Architecture

```
frontend (React)  --REST + SSE-->  backend (Go/Gin)  --driver-->  MongoDB (users, polls, votes)
                                          |
                                          +--> Redis: vote counters, voter dedupe set, Pub/Sub
```

1. React talks to the Go/Gin REST API for auth, poll CRUD, and voting.
2. A vote is only accepted once per `(pollId, voterId)` — enforced atomically in Redis with `SADD`, not just trusted from the client.
3. An accepted vote increments a Redis hash (`HINCRBY`) for instant counts, is written to MongoDB as an audit record, and updates the poll's denormalized count in MongoDB.
4. The backend publishes the new counts to a Redis Pub/Sub channel for that poll.
5. A single in-process hub subscribes to Redis and fans each update out to every SSE connection currently open for that poll, in any backend replica.
6. Every browser on the poll's page holds an `EventSource` connection and repaints its result bars the instant an update arrives.

## Project Structure

```
/frontend   > React app (Vite, react-router)
/backend    > Go service (Gin, MongoDB driver, go-redis)
  /config       env loading
  /db           Mongo + Redis connection helpers
  /models       User, Poll, Option, Vote
  /middleware   JWT auth guard
  /handlers     auth + poll HTTP handlers, SSE stream
  /realtime     Redis-backed pub/sub hub
/docker-compose.yml   local MongoDB + Redis
DEPLOYMENT.md         step-by-step guide to a live, working deployment
```

## Key Decisions

- **Voting stays public, poll creation doesn't.** The brief asks for "a real check before someone can create or manage a poll," not a login wall on voting — a public poll link that requires an account to vote would kill the use case. So `GET /api/polls/:id`, `POST /api/polls/:id/vote`, and the SSE stream are open; poll creation and closing require a valid JWT and an ownership check.
- **Redis is load-bearing, not decorative.** Vote counts live in a Redis hash updated with `HINCRBY`, duplicate votes are rejected with an atomic `SADD` (so a byte-for-byte replay of the vote request still only counts once), and every count update is fanned out over Redis Pub/Sub — the piece that lets multiple backend replicas serve one consistent live stream.
- **MongoDB is the durable record.** Redis holds the fast/live view; MongoDB holds users, poll documents, and a full vote audit log, so nothing is lost if Redis is ever flushed or restarted.
- **SSE over WebSockets.** Updates only flow server → client here (nobody needs to send anything over the live channel), so SSE is a simpler, HTTP-native fit, and it plays well with the "actually deployed" requirement since it works over plain HTTPS with no extra protocol upgrade to configure.

## Requirements

Node.js 18+, Go 1.22+, Docker Desktop, MongoDB, and Redis. Docker Compose supplies MongoDB and Redis locally.

## Installation

```powershell
docker compose up -d
Copy-Item .env.example .env
```

Run the backend:

```powershell
cd backend
go mod tidy
go run .
```

Run the frontend in a second terminal:

```powershell
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`.

## Environment Variables

See `.env.example` at the repo root — both the backend and the Vite frontend read from this one file (Vite is configured with `envDir` pointing at the repo root). The backend reads `PORT`, `MONGO_URI`, `MONGO_DATABASE`, `REDIS_ADDR`, `JWT_SECRET`, and `FRONTEND_URL`. Vite reads `VITE_API_URL`.

## API Endpoints

- `POST /api/auth/signup`
- `POST /api/auth/login`
- `GET /api/auth/me`
- `POST /api/polls`
- `GET /api/polls`
- `GET /api/polls/:id`
- `POST /api/polls/:id/vote`
- `PATCH /api/polls/:id/close`
- `GET /api/polls/:id/events` (SSE)
- `GET /api/health`

## Testing Real Time

Open the same public poll URL in two browser windows. Keep the results view open in Window A. Vote from Window B. Window A receives the Redis-to-SSE event and updates the progress bars immediately. A voter id in local storage, checked again against Redis on the backend, prevents repeated votes from the same browser.

## Security

Passwords are bcrypt-hashed and never returned. JWTs are signed with `JWT_SECRET`. Backend validation checks ownership, poll status, option membership, lengths, duplicates, and malformed input on every write route. CORS is limited to `FRONTEND_URL`.

## Deployment

See [`DEPLOYMENT.md`](./DEPLOYMENT.md) for the full step-by-step: MongoDB Atlas, a managed Redis instance, deploying the Go API as a container, and deploying the React frontend as a static site.
