# AI-Powered Lead Scoring Platform

Standalone lead scoring application that prioritizes leads using interaction data and machine learning algorithms.

## Overview

AI-Powered Lead Scoring Platform is a complete, self-contained application that demonstrates intelligent lead prioritization using customer interaction data. Built with React + Tailwind UI and Express API service, it provides a full-featured lead scoring solution.

**Business Goal:** Demonstrate effective lead scoring capabilities
**Value Proposition:** Complete lead scoring platform with transparent algorithms and actionable insights

## Architecture

```mermaid
flowchart LR
  subgraph Client
    U[Sales User]
  end
  subgraph App[Lead Scoring Platform]
    FE[React + Tailwind Dashboard]
    API[Express API Service]
    AUTH[Authentication Service]
  end
  subgraph DB[Data Layer]
    MDB[(MongoDB users, interactions, leadScores)]
    CACHE[(Redis Cache - optional)]
  end
  U --> FE
  FE -->|authenticate| AUTH
  FE -->|fetch leads| API
  API -->|validate auth| AUTH
  API -->|read/write| MDB
  API -->|cache| CACHE
```

### Components

- **Frontend:** React (Vite) with Tailwind CSS
- **Backend:** Express API service with authentication
- **Database:** MongoDB (users, interactions, leadScores)
- **Authentication:** JWT-based user authentication
- **Caching:** Optional Redis for performance optimization

## Features

### Lead Scoring Dashboard
- **Columns:** Lead Name, Score, Priority, Email Opens, Website Visits, Time Spent, Last Activity, Owner
- **Sorting:** By Score (desc default), then Last Activity; clickable headers
- **Filters:** minScore; ownerId (optional); pagination (page/limit)
- **Real-time:** Refresh button + auto-refresh every 60s
- **Accessibility:** Compliant table with loading/empty/error states

### API Endpoint (GET /leads/scoring)
- **Authentication:** Bearer JWT with tenant scoping
- **Query Parameters:** limit (≤200), page (≥1), sort, minScore, ownerId, includeRaw, refresh
- **Response:** Paginated lead data with scores, breakdown, and metadata
- **Caching:** Uses cached scores if fresh (≤15 min); recomputes on-demand or when refresh=true

## Scoring Algorithm

**Formula:** `score = clamp01(score_raw) * decay(lastActivityAt) * 100`

Where:
- `score_raw = w1*norm(emailOpens) + w2*norm(websiteVisits) + w3*norm(timeSpent)`
- **Normalization caps:** emailOpens cap=10, websiteVisits cap=20, timeSpent cap=600s
- **Default weights:** w1=0.4, w2=0.35, w3=0.25 (configurable via env)
- **Time decay:** half-life 7 days → `decay = 0.5 ^ (days_since_last_activity/7)`

**Explainable:** Returns score breakdown and lastActivityAt for transparency

## Data Model

### users (authentication & ownership)
```
userId, email:string, passwordHash:string, name:string,
role:string, createdAt:ISODate, lastLoginAt:ISODate
```
**Indexes:** `{email:1}` (unique), `{userId:1}`

### interactions (lead activity data)
```
userId, leadId, leadName:string, leadEmail:string, emailOpens:number,
websiteVisits:number, timeSpent:number(seconds), lastActivityAt:ISODate, updatedAt:ISODate
```
**Indexes:** `{userId:1, leadId:1}`, `{userId:1, updatedAt:-1}`, `{userId:1, lastActivityAt:-1}`

### leadScores (computed scores)
```
userId, leadId, leadName:string, score:number(0–100),
breakdown:{emailOpens, websiteVisits, timeSpent, decayFactor?},
computedAt:ISODate, lastActivityAt:ISODate
```
**Indexes:** `{userId:1, score:-1}`, `{userId:1, computedAt:-1}`, `{leadId:1}`

## Local Development

### Quick Start (Docker Compose)
1. **Prerequisites:** Docker Desktop or Docker Engine + Docker Compose v2
2. **Setup:** `cp .env.example .env`
3. **Start MongoDB:** `docker compose up -d mongo`
4. **Verify:** MongoDB listening on `localhost:27017`

**Connection:** `mongodb://root:example@localhost:27017/?authSource=admin`
**Database:** `lead_scoring` (created on first use)

### When API/UI are implemented
- Uncomment `api` and `dashboard` services in `docker-compose.yml`
- Build and start: `docker compose up --build`
- **API:** `http://localhost:3001`
- **Dashboard:** `http://localhost:3000`

See [docs/LOCAL_DEV.md](docs/LOCAL_DEV.md) for detailed instructions.

## Configuration

### Environment Variables
```bash
# MongoDB
MONGODB_URI=mongodb://root:example@localhost:27017/?authSource=admin
MONGODB_DB=lead_scoring

# Scoring Algorithm
LEAD_SCORE_WEIGHTS=0.4,0.35,0.25
LEAD_SCORE_CAPS=10,20,600
LEAD_SCORE_DECAY_HALFLIFE_DAYS=7
SCORE_FRESHNESS_MINUTES=15

# Authentication
JWT_ISSUER=http://localhost/mock-issuer
JWKS_URI=http://localhost/mock-jwks
AUTH_SECRET=dev-secret
```

## Project Structure

```
├── services/
│   ├── lead_scoring/     # Express API service
│   ├── dashboard/        # React UI (planned)
│   └── data/            # MongoDB connectors (planned)
├── docs/                # Documentation
├── docker-compose.yml   # Local development
├── .env.example        # Environment template
└── .app_spec.md        # Full specification
```

## API Reference

### GET /leads/scoring

**Headers:** `Authorization: Bearer <jwt>`

**Query Parameters:**
- `limit` (default 50, max 200)
- `page` (default 1)
- `sort` (default score:desc)
- `minScore` (optional)
- `ownerId` (optional)
- `includeRaw` (default false)
- `refresh` (default false)

**Response (200):**
```json
{
  "data": [
    {
      "tenantId": "string",
      "leadId": "string",
      "score": 85,
      "breakdown": {
        "emailOpens": 8,
        "websiteVisits": 15,
        "timeSpent": 420,
        "decayFactor": 0.95
      },
      "lastActivityAt": "2025-08-08T10:30:00Z",
      "ownerId": "string",
      "raw": { /* optional */ }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "sort": "score:desc",
    "refreshed": false
  }
}
```

**Errors:** 400 (invalid params), 401 (unauthorized), 429 (rate limited), 500 (internal error)

## Testing Strategy

- **Unit:** Scoring function (weights, caps, decay, edge cases)
- **Integration:** API with mongodb-memory-server; cache behavior; filters/sort
- **UI:** React Testing Library (sorting, loading/error states)
- **E2E:** Playwright/Cypress for user interactions
- **Coverage:** >90% for scoring module

## Performance & Scalability

- **Target:** P95 < 300 ms @ 100 RPS
- **Caching:** Optional Redis TTL = freshness window
- **Indexes:** Optimized for tenant + score/activity queries
- **Future:** Scheduled batch recompute for active leads

## Security

- **Authentication:** JWT validation (issuer, audience, signature)
- **Authorization:** Tenant scoping; optional ownerId filtering
- **Input Validation:** Bounds-check query params; sanitize inputs
- **Database:** Least-privileged roles (read interactions; readWrite leadScores)
- **Rate Limiting:** 60 RPM per user (configurable)

## Deployment

**Local:** Docker Compose for development
**Production:** Containerized services (Docker/Kubernetes/VM)

## Links

- **GitHub:** [sneakyaneurysm/nexlify-lead-scorer](https://github.com/sneakyaneurysm/nexlify-lead-scorer)
- **Full Spec:** [.app_spec.md](.app_spec.md)
- **Local Dev:** [docs/LOCAL_DEV.md](docs/LOCAL_DEV.md)
