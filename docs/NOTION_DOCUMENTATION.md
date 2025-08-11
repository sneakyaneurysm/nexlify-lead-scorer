# AI-Powered Lead Scoring Dashboard - Complete Notion Documentation

## Project Overview
**Status:** In Development  
**Version:** 1.0.0 MVP  
**Release Type:** Major  
**Documentation Status:** In Progress  

**Business Goal:** +15% conversion rate improvement (~$750K ARR)  
**Timeline:** 4-week MVP development  
**Team:** Nexlify Solutions  

## Quick Links
- [GitHub Repository](https://github.com/sneakyaneurysm/nexlify-lead-scorer)
- [Linear Epic](https://linear.app/nexlify-solutions/issue/NEX-66/epic-lead-scoring-mvp-implementation)
- [Technical Specification](#technical-specification)
- [API Documentation](#api-documentation)
- [Architecture Diagrams](#architecture)

## Features
- Real-time lead scoring with weighted algorithm
- Sortable dashboard with filtering capabilities  
- MongoDB integration with caching layer
- JWT authentication and tenant isolation
- Docker-based development environment

## Development Phases
### Phase 1: Foundation ✅ Ready
- Infrastructure & Docker setup (NEX-67)
- MongoDB schemas & data service (NEX-68)
- JWT authentication & security (NEX-69)

### Phase 2: Core Services 🔄 Next
- Lead scoring algorithm implementation (NEX-70)
- Express API service with /leads/scoring (NEX-71)
- Caching & performance optimization (NEX-72)

### Phase 3: User Interface 📋 Planned
- React dashboard foundation (NEX-73)
- Lead scoring table with sorting/filtering (NEX-74)
- API integration & real-time features (NEX-75)

### Phase 4: Quality & Deployment 📋 Planned
- Comprehensive testing suite (NEX-76)
- Production deployment & monitoring (NEX-77)
- Documentation & handoff (NEX-78)

---

# Technical Specification

## Architecture Overview

```mermaid
flowchart LR
  subgraph Client
    U[Sales User]
  end
  subgraph Web[Web App]
    FE[React + Tailwind\nLeadScoringDashboard]
    API[/API Service\n/leads/scoring]
  end
  subgraph SVC[Existing Nexlify Services]
    UM[(User Mgmt)]
    AN[(Analytics)]
  end
  subgraph DB[Data Layer]
    MDB[(MongoDB\ninteractions, leadScores)]
    CACHE[(Redis Cache - optional)]
  end
  U --> FE
  FE -->|fetch leads| API
  API -->|auth/profile| UM
  API -->|fetch interactions| AN
  API -->|read/write| MDB
  API -->|cache| CACHE
```

## Technology Stack
- **Frontend:** React + Vite + Tailwind CSS
- **Backend:** Node.js + Express + TypeScript
- **Database:** MongoDB with Redis caching
- **Authentication:** JWT with tenant scoping
- **Deployment:** Docker Compose (local) + Containerized services
- **Testing:** Jest + Supertest + Playwright

## Data Model

### interactions Collection (Read-Only)
```javascript
{
  tenantId: string,
  userId: string, 
  leadId: string,
  emailOpens: number,
  websiteVisits: number,
  timeSpent: number, // seconds
  lastActivityAt: ISODate,
  updatedAt: ISODate
}
```

**Indexes:**
- `{tenantId: 1, leadId: 1}`
- `{tenantId: 1, updatedAt: -1}`

### leadScores Collection (Materialized Cache)
```javascript
{
  tenantId: string,
  leadId: string,
  score: number, // 0-100
  breakdown: {
    emailOpens: number,
    websiteVisits: number, 
    timeSpent: number,
    decayFactor?: number
  },
  computedAt: ISODate,
  lastActivityAt: ISODate,
  ownerId?: string
}
```

**Indexes:**
- `{tenantId: 1, score: -1}`
- `{tenantId: 1, computedAt: -1}`
- `{tenantId: 1, ownerId: 1, score: -1}`

## Scoring Algorithm

**Formula:** `score = clamp01(score_raw) * decay(lastActivityAt) * 100`

Where:
- `score_raw = w1*norm(emailOpens) + w2*norm(websiteVisits) + w3*norm(timeSpent)`
- **Normalization caps:** emailOpens=10, websiteVisits=20, timeSpent=600s
- **Default weights:** w1=0.4, w2=0.35, w3=0.25
- **Time decay:** 7-day half-life → `decay = 0.5 ^ (days_since_last_activity/7)`

## Performance Requirements
- **API Latency:** P95 < 300ms @ 100 RPS
- **Cache Hit Ratio:** >80%
- **Test Coverage:** >90% for scoring module
- **Availability:** ≥99.9%

## Security Requirements
- **Authentication:** JWT validation (issuer, audience, signature)
- **Authorization:** Tenant scoping; optional ownerId filtering
- **Input Validation:** Bounds-check query params; sanitize inputs
- **Database:** Least-privileged roles (read interactions; readWrite leadScores)
- **Rate Limiting:** 60 RPM per user (configurable)

---

# API Documentation

## Base URL
- **Local Development:** `http://localhost:3001`
- **Production:** TBD

## Authentication
All endpoints require JWT authentication via `Authorization: Bearer <token>` header.

## Endpoints

### GET /leads/scoring

Retrieve paginated, scored leads for the authenticated tenant.

#### Request

**Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | number | 50 | Max results per page (≤200) |
| `page` | number | 1 | Page number (≥1) |
| `sort` | string | "score:desc" | Sort order: "score:desc", "lastActivityAt:desc" |
| `minScore` | number | - | Filter leads with score ≥ value |
| `ownerId` | string | - | Filter by lead owner |
| `includeRaw` | boolean | false | Include raw interaction data |
| `refresh` | boolean | false | Force score recalculation |

#### Response

**Success (200):**
```json
{
  "data": [
    {
      "tenantId": "tenant_123",
      "leadId": "lead_456", 
      "score": 85,
      "breakdown": {
        "emailOpens": 8,
        "websiteVisits": 15,
        "timeSpent": 420,
        "decayFactor": 0.95
      },
      "lastActivityAt": "2025-08-08T10:30:00Z",
      "ownerId": "user_789",
      "raw": { /* optional raw data */ }
    }
  ],
  "meta": {
    "page": 1,
    "limit": 50,
    "sort": "score:desc",
    "minScore": null,
    "refreshed": false
  }
}
```

**Error Responses:**
- **400 Bad Request:** Invalid query parameters
- **401 Unauthorized:** Missing or invalid JWT
- **429 Too Many Requests:** Rate limit exceeded (60 RPM)
- **500 Internal Server Error:** Server error

#### Example Requests

**Basic request:**
```bash
curl -H "Authorization: Bearer <token>" \
     "http://localhost:3001/leads/scoring?limit=10&sort=score:desc"
```

**Filtered request:**
```bash
curl -H "Authorization: Bearer <token>" \
     "http://localhost:3001/leads/scoring?minScore=70&ownerId=user_123&includeRaw=true"
```

### GET /health

Health check endpoint for monitoring.

#### Response
```json
{
  "status": "healthy",
  "timestamp": "2025-08-08T10:30:00Z",
  "services": {
    "mongodb": "connected",
    "redis": "connected"
  }
}
```

## Rate Limiting
- **Limit:** 60 requests per minute per user
- **Headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Error Handling
All errors follow this format:
```json
{
  "error": "error_code",
  "message": "Human readable message",
  "details": { /* optional additional context */ }
}
```

---

# Development Guide

## Getting Started

### Prerequisites
- Docker Desktop or Docker Engine + Docker Compose v2
- Node.js 18+ (for local development without containers)
- Git

### Local Setup

1. **Clone Repository**
   ```bash
   git clone https://github.com/sneakyaneurysm/nexlify-lead-scorer.git
   cd nexlify-lead-scorer
   ```

2. **Environment Setup**
   ```bash
   cp .env.example .env
   # Edit .env with your local settings
   ```

3. **Start MongoDB**
   ```bash
   docker compose up -d mongo
   ```

4. **Verify Setup**
   - MongoDB: `mongodb://root:example@localhost:27017/?authSource=admin`
   - Database: `lead_scoring`

### Development Workflow

#### Phase 1: Foundation
- [ ] **NEX-67:** Infrastructure & Docker setup
- [ ] **NEX-68:** MongoDB schemas & data service
- [ ] **NEX-69:** JWT authentication & security

#### Phase 2: Core Services  
- [ ] **NEX-70:** Lead scoring algorithm
- [ ] **NEX-71:** Express API service
- [ ] **NEX-72:** Caching & performance

#### Phase 3: User Interface
- [ ] **NEX-73:** React dashboard foundation
- [ ] **NEX-74:** Lead scoring table
- [ ] **NEX-75:** API integration

#### Phase 4: Quality & Deployment
- [ ] **NEX-76:** Testing suite
- [ ] **NEX-77:** Production deployment
- [ ] **NEX-78:** Documentation

### Code Standards
- **Language:** TypeScript across all services
- **Style:** ESLint + Prettier with snake_case naming
- **Testing:** Jest (unit) + Supertest (integration) + Playwright (E2E)
- **Coverage:** >90% for core modules

### Project Structure
```
├── services/
│   ├── lead_scoring/     # Express API service
│   ├── dashboard/        # React UI
│   └── data/            # MongoDB connectors
├── docs/                # Documentation
├── docker-compose.yml   # Local development
└── .app_spec.md        # Full specification
```

## Environment Variables

### MongoDB
```bash
MONGO_INITDB_ROOT_USERNAME=root
MONGO_INITDB_ROOT_PASSWORD=example
MONGODB_DB=lead_scoring
MONGODB_URI=mongodb://root:example@localhost:27017/?authSource=admin
```

### Scoring Algorithm
```bash
LEAD_SCORE_WEIGHTS=0.4,0.35,0.25
LEAD_SCORE_CAPS=10,20,600
LEAD_SCORE_DECAY_HALFLIFE_DAYS=7
SCORE_FRESHNESS_MINUTES=15
```

### Authentication
```bash
JWT_ISSUER=http://localhost/mock-issuer
JWKS_URI=http://localhost/mock-jwks
AUTH_SECRET=dev-secret
```

## Testing Strategy

### Unit Tests
- **Framework:** Jest
- **Coverage:** >90% for scoring algorithm
- **Focus:** Pure functions, edge cases, error handling

### Integration Tests
- **Framework:** Supertest + mongodb-memory-server
- **Coverage:** API endpoints, database operations
- **Focus:** Request/response validation, auth flows

### E2E Tests
- **Framework:** Playwright
- **Coverage:** Critical user journeys
- **Focus:** Dashboard interactions, sorting, filtering

### Performance Tests
- **Tools:** Artillery or k6
- **Targets:** P95 < 300ms, 100 RPS sustained
- **Focus:** API latency, database query performance

## Deployment

### Local Development
```bash
# Start all services
docker compose up --build

# API available at: http://localhost:3001
# Dashboard available at: http://localhost:3000
```

### Production
- **Containerization:** Docker images for each service
- **Orchestration:** Kubernetes or Docker Swarm
- **Monitoring:** Health checks, metrics, logging
- **Security:** Environment-based secrets, network policies

---

# Project Tracking

## Linear Tickets Overview

### Epic
- **NEX-66:** EPIC: Lead Scoring MVP Implementation

### Phase 1: Foundation (Todo)
- **NEX-67:** Setup project infrastructure and Docker development environment
- **NEX-68:** Implement MongoDB schemas and data service layer
- **NEX-69:** Implement JWT authentication middleware and security layer

### Phase 2: Core Services (Backlog)
- **NEX-70:** Implement lead scoring algorithm and calculation engine
- **NEX-71:** Build Express API service with /leads/scoring endpoint
- **NEX-72:** Implement caching layer and performance optimization

### Phase 3: User Interface (Backlog)
- **NEX-73:** Build React dashboard foundation with routing and layout
- **NEX-74:** Implement lead scoring table with sorting and filtering
- **NEX-75:** Integrate dashboard with API and implement real-time features

### Phase 4: Quality & Deployment (Backlog)
- **NEX-76:** Implement comprehensive testing suite (unit, integration, E2E)
- **NEX-77:** Setup production deployment and monitoring
- **NEX-78:** Complete documentation and project handoff

## Success Metrics

### Business Metrics
- **Primary:** +15% conversion rate improvement
- **Secondary:** $750K ARR increase
- **Adoption:** % of sales team using dashboard weekly

### Technical Metrics
- **Performance:** P95 API latency < 300ms
- **Reliability:** >99.9% uptime
- **Quality:** >90% test coverage
- **User Experience:** TTI < 2.5s for dashboard

### Development Metrics
- **Velocity:** Tickets completed per sprint
- **Quality:** Bug rate, code review feedback
- **Documentation:** API docs completeness, architecture clarity

---

# Troubleshooting

## Common Issues

### MongoDB Connection
**Problem:** Cannot connect to MongoDB  
**Solution:** Ensure Docker container is running: `docker compose ps`

### JWT Authentication
**Problem:** 401 Unauthorized errors  
**Solution:** Verify JWT_ISSUER and token format in .env

### Performance Issues
**Problem:** Slow API responses  
**Solution:** Check MongoDB indexes, enable Redis caching

### Docker Issues
**Problem:** Services won't start  
**Solution:** Check port conflicts, rebuild images: `docker compose up --build`

## Support Contacts
- **Technical Lead:** Bryan Feuling
- **Repository:** [GitHub Issues](https://github.com/sneakyaneurysm/nexlify-lead-scorer/issues)
- **Project Tracking:** [Linear Workspace](https://linear.app/nexlify-solutions)

---

*Last Updated: August 11, 2025*  
*Version: 1.0.0*  
*Status: In Development*
