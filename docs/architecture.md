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

- Presentation tier decision: Keep route-specific rendering and request state in the React components; use the shared API client for relative `/api` requests, and keep persistence and business rules out of the browser.
- Logic tier decision: Keep Express route handlers as thin request/response adapters and implement validation and capability behavior in `backend/src/services/octofitService.ts`; return deliberate 4xx errors for invalid or unknown resources.
- Data tier decision: Use Mongoose models for required-field validation, defaults, and User/Team references. Keep seed data idempotent and preserve records that are not owned by the seed fixtures.
- Activity tracking approach: Validate `userId` and `teamId` formats and existence before creating an activity; store them as `user` and `team`, derive `points` in the service, and set `occurredAt` on the server.
- Team management approach: Require a non-empty name, preserve the submitted name and mascot, and default members to an empty array when none are provided.
- Leaderboard approach: Sum activity points by team, sort from highest total to lowest, and add 1-based ranks in the service response.
- Workout suggestions approach: Find the user by email, use the user's goal and recent activities to select and explain the first matching workout, and return a deliberate 4xx when no user matches.

## Test plan

Run the focused contract suites in dependency order, then run both builds. A contract failure points to behavior in that tier; a build failure points to a type, import, or bundling issue that must be resolved before integration.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks model validation, defaults, references, and idempotent/preserving seed behavior. Failures indicate a schema or seed defect.
- API logic: `npm run test:api --prefix octofit-tracker/backend` checks request validation, persistence, response shapes, ranking, and suggestion selection. Failures indicate service behavior or route-to-service integration defects.
- Presentation: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering and accessible loading, empty, and error states. Failures indicate client/component behavior or presentation contract defects.
- Backend build: `npm run build --prefix octofit-tracker/backend` checks TypeScript compilation. Failures indicate backend type or module integration defects.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` checks TypeScript compilation and Vite bundling. Failures indicate frontend type, import, or bundling defects.

Baseline from the current scaffold: the data suite has 4 failing validation tests while the seed test passes; all 4 API contract tests fail; all 15 frontend contract tests fail. Rerun each suite after its tier changes, then rerun the full set and both builds before considering the implementation complete.
