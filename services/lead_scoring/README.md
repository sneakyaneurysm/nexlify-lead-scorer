# lead_scoring_service

Purpose: Back-end service exposing the /leads/scoring endpoint (Express-compatible) to compute and return prioritized leads based on MongoDB interaction data.

Key responsibilities:
- Implement GET /leads/scoring with JWT auth, input validation, and standardized errors
- Calculate lead score using weighted sum + optional time decay
- Read from `interactions`, upsert/read from `leadScores`
- Follow functional programming style and snake_case naming

Planned structure:
- src/routes/leads_scoring.ts
- src/utils/calculate_lead_score.ts
- src/middleware/auth.ts, src/middleware/validate.ts
- src/models/* (type definitions only)

See project root /.app_spec.md for full specification.

