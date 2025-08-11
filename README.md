# AI-Powered Lead Scoring Dashboard

Demo repository for Nexlify's CRM lead scoring application that prioritizes leads using interaction data and machine learning.

## Overview

AI-Powered Lead Scoring Dashboard prioritizes leads in Nexlify's CRM using a simple, transparent model based on interaction data. It integrates with existing services and MongoDB, and ships as a React + Tailwind UI with an Express API service.

**Business Goal:** +15% conversion rate improvement (~$750K ARR)
**Value Proposition:** Actionable lead prioritization; improved CRM stickiness and acquisition

## Architecture

```mermaid
flowchart LR
  subgraph Client
    U[Sales User]
  end
  subgraph Web[Web App]
    FE[React + Tailwind Dashboard]
    API[API Service /leads/scoring]
  end
  subgraph SVC[Existing Nexlify Services]
    UM[(User Mgmt)]
    AN[(Analytics)]
  end
  subgraph DB[Data Layer]
    MDB[(MongoDB interactions & leadScores)]
    CACHE[(Redis Cache - optional)]
  end
  U --> FE
  FE -->|fetch leads| API
  API -->|auth/profile| UM
  API -->|fetch interactions| AN
  API -->|read/write| MDB
  API -->|cache| CACHE
```

### Components

- **Frontend:** React (Next.js or Vite) with Tailwind CSS
- **Backend:** Express API service route /leads/scoring
- **Data:** MongoDB (interactions source; leadScores materialized cache)
- **Integrations:** User Mgmt (auth/owner), Analytics (optional interactions), GitHub + Linear + Notion

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

### interactions (read-only)
```
tenantId, userId, leadId, emailOpens:number, websiteVisits:number,
timeSpent:number(seconds), lastActivityAt:ISODate, updatedAt:ISODate
```
**Indexes:** `{tenantId:1, leadId:1}`, `{tenantId:1, updatedAt:-1}`

### leadScores (materialized cache)
```
tenantId, leadId, score:number(0–100),
breakdown:{emailOpens, websiteVisits, timeSpent, decayFactor?},
computedAt:ISODate, lastActivityAt:ISODate, ownerId?
```
**Indexes:** `{tenantId:1, score:-1}`, `{tenantId:1, computedAt:-1}`, `{tenantId:1, ownerId:1, score:-1}`

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
