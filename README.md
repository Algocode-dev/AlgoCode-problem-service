# AlgoCode — Problem Service

> REST API microservice for managing coding problems, test cases, and constraints in the AlgoCode online judge platform. Serves as the source of truth for problem data consumed by the Evaluator and API Gateway.

---

## Table of Contents

- [Overview](#overview)
- [System Position](#system-position)
- [Why a Dedicated Problem Service?](#why-a-dedicated-problem-service)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Tech Stack](#tech-stack)
- [Data Model](#data-model)
- [Project Structure](#project-structure)
- [API Reference](#api-reference)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Running Tests](#running-tests)
- [Part of the AlgoCode Platform](#part-of-the-algocode-platform)

---

## Overview

The Problem Service owns all data related to coding problems — titles, descriptions, difficulty levels, input/output constraints, and the test cases used to evaluate submissions. It exposes a versioned REST API that the API Gateway proxies for clients, and is queried directly by the Evaluator Service when fetching test cases for a submission.

This service does one thing and does it well: manage the lifecycle of problems and their associated test data. No auth logic, no execution logic — that separation is intentional.

---

## System Position

```
┌────────────────────────────────────────────────────┐
│                   AlgoCode Platform                 │
│                                                     │
│   Client ──► API Gateway ──► Problem Service ◄──┐  │
│                                    │              │  │
│                          (test cases fetched)    │  │
│                                    │              │  │
│                                    ▼              │  │
│                          Evaluator Service ───────┘  │
│                                                     │
└────────────────────────────────────────────────────┘
```

The Problem Service is queried in two scenarios:
1. **Client reads** — a user browsing problems; routed through the API Gateway
2. **Evaluator reads** — fetching test cases for a specific problem to evaluate a submission

---

## Why a Dedicated Problem Service?

### Problem: Mixing concerns breaks scalability
In a monolith, problem management, auth, and code execution live in the same process. A spike in submissions slows down problem reads. A schema change for problems risks breaking the execution pipeline.

### Solution: Own your domain
The Problem Service owns its own database and exposes a clean API contract. This means:
- Problem reads and writes can scale independently of submission load
- The schema can evolve without risk to the evaluator or auth layers
- Test cases are versioned and managed in one place — no duplication across services

---

## Key Engineering Decisions

### 1. Layered architecture (Controller → Service → Repository)
No controller directly touches the database. Every request flows through:

```
Route → Controller → Service → Repository → MongoDB
```

This separation means:
- Business rules (e.g. "a problem must have at least one test case before it's published") live in the service layer, not scattered in controllers
- The repository layer is the only place that knows about MongoDB — swap the DB and only one layer changes
- Each layer is independently testable

### 2. MongoDB for problem data
Problems are document-heavy — they have descriptions, constraints, examples, tags, and nested test cases. A relational schema would require awkward joins to reconstruct a single problem. MongoDB's document model stores a problem and all its test cases as a single document, making reads fast and the schema flexible as requirements evolve.

### 3. Versioned API routes (`/api/v1/`)
All routes are prefixed with `/v1`. When breaking changes are needed, a `/v2` prefix is introduced alongside rather than replacing the existing contract. Downstream consumers (Gateway, Evaluator) are never broken by problem service changes.

### 4. Structured Winston logging with file output
The service logs to both console and a rotating file (`app.log`). Every request logs method, route, status, and duration. This is the `app.log` you see committed to the repo — it's a live trace of a real test run, not a placeholder.

### 5. Test suite with real DB fixtures
The `tests/` directory contains integration tests that seed a test database with known problems and assert correct API responses. This means every API contract is verified — not just the happy path.

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Runtime | Node.js | Async I/O, consistent with platform |
| Framework | Express.js | Lightweight routing and middleware |
| Database | MongoDB | Document model fits problem/test case structure |
| ODM | Mongoose | Schema validation, nested document support |
| Logging | Winston | Structured logs with file and console transports |
| Testing | Jest / Supertest | Integration tests against a real Express app |

---

## Data Model

### Problem

```js
{
  _id: ObjectId,
  title: String,               // "Two Sum"
  description: String,         // Full problem statement (markdown supported)
  difficulty: Enum,            // "EASY" | "MEDIUM" | "HARD"
  tags: [String],              // ["arrays", "hash-map"]
  constraints: String,         // "1 ≤ nums.length ≤ 10^4"
  inputFormat: String,
  outputFormat: String,
  sampleInput: String,         // Shown to user
  sampleOutput: String,        // Shown to user
  testCases: [                 // Hidden — used by evaluator only
    {
      input: String,
      expectedOutput: String
    }
  ],
  isPublished: Boolean,
  createdAt: Date,
  updatedAt: Date
}
```

Test cases are embedded in the problem document. This means the evaluator fetches a single document and gets everything it needs — no multi-collection join, no round trips.

---

## Project Structure

```
src/
├── config/
│   ├── database.js        # Mongoose connection setup
│   ├── server.js          # Express app, middleware registration
│   └── logger.js          # Winston logger (console + file)
├── controllers/
│   └── problem.js         # HTTP handlers — delegates to service
├── middlewares/
│   ├── auth.js            # JWT validation (forwarded from gateway)
│   └── validate.js        # Request body schema checks
├── models/
│   └── problem.js         # Mongoose schema with embedded test cases
├── repositories/
│   └── problem.js         # All MongoDB queries — isolated here
├── routes/
│   └── v1/
│       └── problem.js     # /api/v1/problems routes
└── services/
    └── problem.js         # Business logic: publish checks, pagination

tests/
├── problem.test.js        # Integration tests with seeded test DB
└── fixtures/
    └── problems.js        # Sample problem data for test runs
```

---

## API Reference

Base URL: `http://localhost:3001/api/v1`

### Problem Routes — `/problems`

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `GET` | `/problems` | List all published problems (paginated) | Yes |
| `GET` | `/problems/:id` | Get a single problem by ID | Yes |
| `POST` | `/problems` | Create a new problem with test cases | Yes — ADMIN |
| `PATCH` | `/problems/:id` | Update problem details or add test cases | Yes — ADMIN |
| `DELETE` | `/problems/:id` | Delete a problem | Yes — ADMIN |
| `PATCH` | `/problems/:id/publish` | Mark a problem as published | Yes — ADMIN |

---

### GET /problems

Returns paginated list of published problems. Test case data is excluded from list responses — only returned on single problem fetch (to keep payloads small).

```json
// Response 200
{
  "success": true,
  "data": {
    "problems": [
      {
        "_id": "64abc123",
        "title": "Two Sum",
        "difficulty": "EASY",
        "tags": ["arrays", "hash-map"]
      }
    ],
    "total": 42,
    "page": 1,
    "limit": 10
  }
}
```

---

### GET /problems/:id

Returns full problem including test cases. This endpoint is consumed by the Evaluator when fetching test input/output for a submission.

```json
// Response 200
{
  "success": true,
  "data": {
    "_id": "64abc123",
    "title": "Two Sum",
    "description": "Given an array of integers nums...",
    "difficulty": "EASY",
    "constraints": "2 ≤ nums.length ≤ 10^4",
    "testCases": [
      { "input": "nums = [2,7,11,15], target = 9", "expectedOutput": "[0,1]" },
      { "input": "nums = [3,2,4], target = 6", "expectedOutput": "[1,2]" }
    ]
  }
}
```

---

### POST /problems

Creates a new problem. Must include at least one test case.

```json
// Request
{
  "title": "Two Sum",
  "description": "Given an array of integers nums and an integer target...",
  "difficulty": "EASY",
  "tags": ["arrays", "hash-map"],
  "constraints": "2 ≤ nums.length ≤ 10^4",
  "inputFormat": "First line: array nums. Second line: integer target.",
  "outputFormat": "Array of two indices.",
  "sampleInput": "nums = [2,7,11,15], target = 9",
  "sampleOutput": "[0,1]",
  "testCases": [
    { "input": "nums = [2,7,11,15], target = 9", "expectedOutput": "[0,1]" },
    { "input": "nums = [3,2,4], target = 6", "expectedOutput": "[1,2]" }
  ]
}

// Response 201
{
  "success": true,
  "message": "Problem created successfully",
  "data": { "_id": "64abc123", "title": "Two Sum", "isPublished": false }
}
```

> Problems are created in draft state. Use `PATCH /problems/:id/publish` to make them visible to users.

---

## Getting Started

### Prerequisites

- Node.js v18+
- MongoDB running locally or via Docker

### Installation

```bash
git clone https://github.com/Algocode-dev/AlgoCode-problem-service.git
cd AlgoCode-problem-service
npm install
```

### MongoDB via Docker (quickest setup)

```bash
docker run -d \
  --name mongo-algocode \
  -p 27017:27017 \
  mongo:6
```

### Run

```bash
# Development
npm run dev

# Production
npm start
```

---

## Environment Variables

Create a `.env` file in the root:

```env
PORT=3001
MONGO_URI=mongodb://localhost:27017/algocode_problems
JWT_SECRET=same_secret_as_gateway   # Used to verify forwarded tokens
LOG_LEVEL=info
```

---

## Running Tests

```bash
# Runs the full integration test suite against a test DB
npm test
```

Tests spin up the Express app, seed the test database with fixtures from `tests/fixtures/problems.js`, and assert correct HTTP responses for all endpoints. The test DB is wiped between runs.

---

## Part of the AlgoCode Platform

| Service | Repo | Description |
|---|---|---|
| API Gateway | [API_Gateway](https://github.com/Algocode-dev/API_Gateway) | Auth (JWT + RBAC), routing, entry point |
| **Problem Service** | **You are here** | Problem and test case management |
| Evaluator Service | [Algocode-Evaluator_Service](https://github.com/Algocode-dev/Algocode-Evaluator_Service) | Sandboxed code execution engine |
| Notification Service | [Notification-service](https://github.com/Algocode-dev/Notification-service) | Async event-driven result delivery |
