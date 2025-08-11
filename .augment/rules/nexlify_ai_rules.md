---
type: "always_apply"
---

globs:
  - "**/*.js"
  - "**/*.ts"
  - "**/*.tsx"

Context: To make sure that the AI is operating consistently, make sure to add the context of what the app is and does, what we have worked on, and what the next items are to work on within the .app_context.md document. ALWAYS make sure that this doc is up-to-date with the context of where we are

Troubleshooting: Any issues that we have while developing and testing this solution should be added to the .app_troubleshooting.md file. The issue, the explanation for why it is an issue, and the resolution for the issue should all be in this doc. If there are any issues while developing or testing, make sure to check the .app_troubleshooting.md doc first

Code Style: Use functional programming principles (e.g., immutability, pure functions, composition over inheritance). Prioritize well-documented code with detailed comments explaining logic, intent, and edge cases.
Naming Conventions: Use snake_case for all variables, functions, files, and database fields (e.g., lead_scoring_dashboard.tsx, calculate_lead_score).
Language: Write prompts, comments, and documentation in English for consistency and LLM compatibility.

Project Structure

Microservice Architecture:

Organize code into microservices: lead_scoring_service (back-end logic), dashboard_service (front-end UI), and data_service (MongoDB interactions).
Each microservice has its own folder: services/lead_scoring, services/dashboard, services/data.
Use RESTful APIs for inter-service communication, with endpoints documented in Notion.

Folder Structure:

Front-end: services/dashboard/src/components (React components), src/styles (Tailwind CSS), src/utils (functional utilities).
Back-end: services/lead_scoring/src/routes (Express routes), src/models (MongoDB schemas), src/middleware (authentication, validation).
Data: services/data/src/connectors (MongoDB connectors), src/schemas (data models).
Tests: services/*/tests/unit, tests/integration, tests/e2e for each microservice.
Documentation: Store API specs and architecture docs in Notion, with markdown files in docs/ for each microservice.
Coding Standards

Functional Programming:

Use pure functions and avoid side effects (e.g., no global state mutations).
Prefer immutability (e.g., use const, spread operators for arrays/objects).
Leverage higher-order functions and composition (e.g., map, reduce, pipe).
Avoid class-based React components; use functional components with hooks.

Frameworks and Libraries:

Use latest compatible versions: React 18.3.1, Express 4.21.0, MongoDB 6.9.0, Tailwind CSS 3.4.14 (as of August 2025).
Verify compatibility via npm/yarn dependency checks to ensure no conflicts.
Reference Notion for exact versions to avoid outdated suggestions.

Error Handling:

Wrap all async operations (e.g., API calls, MongoDB queries) in try-catch blocks.
Validate all inputs (e.g., user IDs, payloads) using libraries like joi or zod.
Return standardized error responses (e.g., { error: "invalid_user_id", status: 400 }) with appropriate HTTP status codes.
Log errors to Sentry (integrated via AugmentCode) with context for debugging.

Security Practices:

Use environment variables for sensitive data (e.g., process.env.MONGODB_URI).
Implement JWT-based authentication for all API endpoints, validated in middleware.
Sanitize inputs to prevent injection attacks (e.g., use mongo-sanitize for MongoDB queries).
Enforce HTTPS for API communication and use Helmet middleware for Express security headers.
Ensure tenant isolation for Nexlify’s data, aligning with AugmentCode’s enterprise-grade security.

Vibe Coding Workflow

Prompt Style: Accept detailed, spec-driven prompts (e.g., “Create a lead scoring microservice with a REST endpoint /leads/scoring that calculates scores from MongoDB data, validated with JWT and zod, and includes unit, integration, and E2E tests”). AugmentCode should only interact when clarification is needed for ambiguous requirements, proposing solutions based on industry best practices.
Clarification: If unsure, AugmentCode should ask targeted questions (e.g., “Should the lead scoring algorithm prioritize email opens over website visits?”) and proceed with scalable, secure defaults if no response is provided.
Optimization: Prioritize production-ready code over rapid prototyping, optimizing for performance (e.g., efficient MongoDB queries, memoized React components) and scalability (e.g., stateless microservices, horizontal scaling support).

Testing:

Generate comprehensive tests for all code:

Unit Tests: Use Jest for React components and Mocha/Chai for Node.js routes, covering success, failure, and edge cases (e.g., invalid inputs, null data).
Integration Tests: Test API endpoints and MongoDB interactions using Supertest and MongoDB Memory Server.
E2E Tests: Use Cypress to simulate user interactions with the lead scoring dashboard (e.g., sorting leads by score).
Ensure 90%+ test coverage, with reports generated in tests/coverage.
Integration-Specific Behavior

Linear:

Every code change (create, update, delete) must be associated with a Linear task.
If no relevant task exists, AugmentCode should create one in Linear (e.g., “Implement lead_scoring_service endpoint” with description and assignee).
Add Linear task IDs in code comments (e.g., // Linear: NL-123).
Update task status (e.g., “In Progress” to “Done”) upon completion.

GitHub:

Commit changes with descriptive messages (e.g., “Add lead_scoring_service with JWT validation // Linear: NL-123”).
Run git diff to verify changes before committing, ensuring only intended modifications are included.
Push commits to the Nexlify repository for real-time indexing by AugmentCode.

Notion:

Pull API specs and schemas from Notion for endpoint development (e.g., /leads/scoring details).
Generate markdown documentation in Notion for each microservice, including endpoint descriptions, request/response formats, and Linear task links.

Code Review and Documentation

Automation: Implement all functionality automatically, with no #TODO(agent) comments, mock data, mock APIs, or mock responses. Use real MongoDB data (e.g., interactions collection with user_id, email_opens, website_visits, time_spent).

Documentation:

Add detailed inline comments for all functions, explaining inputs, outputs, and edge cases (e.g., “Calculates lead score using weighted sum of interactions; handles null data”).
Generate markdown files in docs/ for each microservice, summarizing functionality, API endpoints, and test coverage.
Update Notion with architecture diagrams and endpoint documentation post-development.
Explainability: On request, provide detailed explanations of code files (e.g., “Explain lead_scoring_service.js in terms of microservice responsibilities”).
Industry Best Practices

JavaScript/TypeScript:

Follow Airbnb’s TypeScript style guide (adapted for snake_case).
Use ESLint and Prettier with custom rules to enforce snake_case and functional programming patterns.
Optimize for performance (e.g., avoid unnecessary renders in React, use indexes in MongoDB).

Microservice Architecture:

Ensure services are loosely coupled, with clear boundaries (e.g., lead_scoring_service handles scoring logic, data_service manages DB access).
Use REST APIs with OpenAPI 3.0 specs documented in Notion.
Implement health checks (/health) for each microservice to support monitoring.
Design for horizontal scaling (e.g., stateless services, Redis caching for high-traffic endpoints).

Scalability:

Optimize MongoDB queries with indexes and aggregations for lead scoring.
Use React memoization and lazy loading for the dashboard UI.
Implement rate limiting and circuit breakers in Express to handle traffic spikes.

Security:

Follow OWASP Top 10 guidelines (e.g., prevent XSS, SQL injection).
Use bcrypt for password hashing, JWT for authentication, and Helmet for security headers.
Encrypt sensitive data in transit and at rest, aligning with Nexlify’s enterprise requirements.