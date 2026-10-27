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

- Presentation tier decision: Keep route-specific React components focused on rendering and interaction. Fetch through the shared API client, and give every view explicit loading, error, and empty states without duplicating backend business rules.
- Logic tier decision: Keep Express routes as thin request/response adapters. Put input validation, reference checks, and capability behavior in `octofitService.ts`; return deliberate 4xx responses for invalid or missing resources rather than leaking Mongoose cast or validation errors as 500s.
- Data tier decision: Define required fields and sensible defaults in the Mongoose schemas, including an empty default for team members. Preserve the seed script's idempotence and unrelated existing records; its contract already passes.
- Activity tracking approach: Validate the request shape and ObjectId syntax, confirm the referenced user and team exist, then persist the submitted type/minutes and references with `points` equal to validated minutes and `occurredAt` set at creation time.
- Team management approach: Require a non-empty name and mascot, default members to an empty array, and retain submitted team fields and member references in create/list responses.
- Leaderboard approach: Sum activity points by team, sort by total descending, and assign ranks starting at 1. Keep aggregation and response shaping in the service layer.
- Workout suggestions approach: Find the user by email; return a 4xx if absent. Prefer workouts matching the user's fitness goal, fall back to a focus inferred from recent activity, and include a concise reason tied to that goal or activity.

## Test plan

Record the commands you run and what each failure means.

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks required schema fields/defaults and repeatable, non-destructive seeding. Baseline: 4 schema assertions fail; the seed/idempotency test passes.
- API logic: `npm run test:api --prefix octofit-tracker/backend` checks activity/team creation and validation, leaderboard ordering/ranks, and user-specific workout suggestions. Baseline: all 4 tests fail; activity/team creation return 501, leaderboard is empty, and suggestions omit the required reason.
- Presentation: `npm test --prefix octofit-tracker/frontend` checks API data rendering, the workout request, and accessible loading/empty/error states for all views. Baseline: all 15 tests fail because the capability views still show "Implementation pending."
- Backend build: `npm run build --prefix octofit-tracker/backend` checks TypeScript compilation; a failure indicates backend type errors. Not run in the current baseline.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` checks TypeScript and the Vite production build; a failure indicates frontend type or bundling errors. Not run in the current baseline.
