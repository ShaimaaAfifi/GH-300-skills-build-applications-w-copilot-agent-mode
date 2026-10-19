# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep route components responsible for rendering loading, error, empty, and populated states. Use the shared API client for requests and keep business rules out of the UI.
- Logic tier decision: Keep Express routes as thin HTTP adapters; implement validation and capability behavior in the service layer. Return deliberate 4xx responses for invalid input, malformed IDs, missing references, or unknown users.
- Data tier decision: Enforce required fields and sensible defaults in Mongoose schemas, use ObjectId references for relationships, and keep seeding idempotent without deleting unrelated records.
- Activity tracking approach: Validate the activity type, positive duration, and both well-formed, existing user and team IDs before writing. Store references, submitted activity fields, points computed one-for-one from minutes, and a server-generated occurrence time.
- Team management approach: Require a nonblank name, retain the submitted name and mascot, and store members as a default-empty array of user references.
- Leaderboard approach: Sum activity points per team, order totals descending with team name as a stable tie-breaker, and assign 1-based ranks.
- Workout suggestions approach: Find the user by email, prefer available workouts matching the user's fitness goal, and use recent activity as the fallback signal. Include a short reason with each recommendation; return 404 for an unknown email.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` checks schema validation, defaults, and seed preservation/idempotency. A failure points to model or seed behavior.
- API command: `npm run test:api --prefix octofit-tracker/backend` checks endpoint validation, persistence, ranking, and recommendations. A failure points to service or route behavior, or its data-layer integration.
- Frontend command: `npm test --prefix octofit-tracker/frontend` checks API-driven route rendering and accessible loading, empty, and error states. A failure points to presentation behavior or API-client integration.
- Backend build command: `npm run build --prefix octofit-tracker/backend` checks backend TypeScript types; a failure indicates a compile-time backend issue.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` checks frontend TypeScript and production bundling; a failure indicates a compile-time or bundling issue.
- Baseline run: `npm test --prefix octofit-tracker/backend` reported 8 failed and 1 passed (the idempotent seed test); `npm test --prefix octofit-tracker/frontend` reported 15 failed. Re-run both after implementation to confirm the full suites are green.
