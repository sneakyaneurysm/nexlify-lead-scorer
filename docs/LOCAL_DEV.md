## Local Development (Docker Compose)

This repository includes a Docker Compose setup for local development.

### Prerequisites
- Docker Desktop or Docker Engine + Docker Compose v2
- Node.js 18+ (for local runs without containers)

### Quick start (MongoDB only)
1. Copy env file: `cp .env.example .env`
2. Start MongoDB: `docker compose up -d mongo`
3. Verify: `docker ps` shows `lead_scoring_mongo` and Mongo is listening on `localhost:27017`

Connection string (local):
- `mongodb://root:example@localhost:27017/?authSource=admin`
- Database: `lead_scoring` (created on first use)

### When API/UI are implemented
- Uncomment `api` and `dashboard` services in `docker-compose.yml`
- Build and start all: `docker compose up --build`
- API available at `http://localhost:3001`
- Dashboard available at `http://localhost:3000`

### Environment variables
- See `.env.example`; copy to `.env` and edit as needed.
- Compose automatically loads `.env`.

### Data persistence
- MongoDB data is persisted in a named volume `mongo_data`.
- To reset: `docker compose down -v` (removes volumes) then `docker compose up -d mongo`.

### Troubleshooting
- Port conflicts: Change published ports in `docker-compose.yml`.
- Auth errors to Mongo: Ensure `.env` is loaded and matches Compose variables.
- Connectivity from API to Mongo: Use `mongo` as hostname inside the Compose network.

