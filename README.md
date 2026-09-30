# CareerTrack

AI-powered career & opportunity management platform — React + Java Spring Boot + MongoDB.

This is a working MVP implementation of the SRS: authentication, profiles, company
management, private job/internship tracking, government job tracking, competitive
exam tracking, interviews, assessments, a unified dashboard & calendar, notifications,
a preparation tracker, an AI email assistant, opportunity aggregation with
personalized matching, and basic analytics.

## What's implemented vs. what's a documented extension point

Everything in Section 6 (Functional Requirements) is implemented as real, working
CRUD + business logic against MongoDB, protected by JWT auth. Two areas that the
SRS explicitly scopes as needing external, deployment-specific integration are
implemented as **working defaults with a clear extension point**, rather than
faked:

- **AI Email Assistant (FR-12–FR-14):** you paste/forward the text of a career
  email in the app (no live inbox access, which trivially satisfies "minimize
  email processing"). If you set `ANTHROPIC_API_KEY`, extraction is done by a
  real call to the Anthropic API; if you don't, a deterministic heuristic
  extractor is used instead, so the feature works out of the box. Applying an
  extracted event to an application always requires you to explicitly confirm
  it — see `EmailEventService` / `EmailEventController`.
  To go further and wire up **live** Gmail/Outlook OAuth polling, see the
  Javadoc at the top of `EmailEventController`.
- **Opportunity Aggregation (FR-10/FR-11):** opportunities are entered via
  `POST /api/opportunities` (by hand, by an import script, or by a scheduled
  job you add). No scraper for any specific third-party site is included,
  since picking sources and respecting their terms of use is a deployment
  decision. See `OpportunityCollectorScheduler`'s Javadoc for the extension
  point, and `OpportunityService` for the personalized relevance scoring
  (FR-16) that already runs against whatever opportunities exist.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 18 + Vite + React Router + Tailwind CSS + Axios + Recharts |
| Backend | Java 21 + Spring Boot 3 (Web, Security, Data MongoDB, Validation) |
| Auth | Spring Security + JWT (stateless) |
| Database | MongoDB |
| AI | Anthropic Messages API (optional) with a heuristic fallback |

## Project layout

```
careertrack/
├── backend/                 Spring Boot API (Maven project)
│   ├── src/main/java/com/careertrack/
│   │   ├── model/            MongoDB documents
│   │   ├── repository/       Spring Data repositories
│   │   ├── service/          business logic
│   │   ├── controller/       REST controllers
│   │   ├── security/         JWT filter, util, current-user helper
│   │   ├── config/           Security, CORS, Mongo auditing, RestClient
│   │   ├── scheduler/        daily reminders + opportunity-collector stub
│   │   └── exception/        global error handling
│   └── src/main/resources/application.yml
├── frontend/                 React app (Vite project)
│   └── src/
│       ├── api/               one file per REST resource (axios)
│       ├── context/           AuthContext (JWT, current user)
│       ├── components/        Layout (Navbar/Sidebar) + reusable ResourcePage/Modal/etc.
│       └── pages/              one page per module (Dashboard, Applications, ...)
└── docker-compose.yml
```

## Option A: Run everything with Docker (easiest)

Requires Docker Desktop (or Docker Engine + Compose) with internet access to
pull base images and dependencies.

```bash
cd careertrack
docker compose up --build
```

- Frontend: http://localhost:5173
- Backend API: http://localhost:8080/api
- MongoDB: localhost:27017 (persisted in a named volume)

To enable the LLM-backed email assistant, set an env var before starting:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
docker compose up --build
```

## Option B: Local development (recommended while actively coding)

You'll want three terminals. Prerequisites: **Java 21**, **Maven**, **Node.js 18+**,
and either a local **MongoDB** install or just Mongo via Docker.

**1. Start MongoDB** (skip if you already have Mongo running locally):

```bash
docker run -d --name careertrack-mongo -p 27017:27017 mongo:7
```

**2. Start the backend:**

```bash
cd backend
mvn spring-boot:run
```

The API starts on `http://localhost:8080`. Configuration is entirely via env
vars (see `src/main/resources/application.yml` for defaults) — e.g.:

```bash
export MONGODB_URI="mongodb://localhost:27017/careertrack"
export JWT_SECRET="<base64-encoded-32+-byte-secret>"   # generate your own for anything beyond local dev
export ANTHROPIC_API_KEY="sk-ant-..."                  # optional
mvn spring-boot:run
```

**3. Start the frontend:**

```bash
cd frontend
npm install
cp .env.example .env   # defaults to http://localhost:8080/api
npm run dev
```

Open http://localhost:5173, register an account, and you're in.

## Generating a real JWT secret

The default secret in `application.yml` is fine for local development but
**must** be replaced for anything shared or deployed. Generate a proper one with:

```bash
openssl rand -base64 48
```

Set it as `JWT_SECRET`.

## API overview

All endpoints are under `/api`, and (except `/api/auth/**`) require
`Authorization: Bearer <token>`, obtained from `POST /api/auth/login` or
`POST /api/auth/register`.

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register`, `POST /auth/login` |
| Profile | `GET/PUT /users/me` |
| Companies | `GET/POST /companies`, `GET/PUT/DELETE /companies/{id}` |
| Applications | `GET/POST /applications` (supports `?status=`), `GET/PUT/DELETE /applications/{id}` |
| Interviews | `GET/POST /interviews`, `GET/PUT/DELETE /interviews/{id}` |
| Assessments | `GET/POST /assessments`, `GET/PUT/DELETE /assessments/{id}` |
| Government jobs | `GET/POST /government-opportunities`, `.../{id}` |
| Competitive exams | `GET/POST /competitive-exams`, `.../{id}` |
| Preparation tracker | `GET/POST /study-progress`, `.../{id}` |
| Dashboard | `GET /dashboard` |
| Calendar | `GET /calendar?from=YYYY-MM-DD&to=YYYY-MM-DD` |
| Notifications | `GET /notifications`, `PUT /notifications/{id}/read`, `PUT /notifications/read-all` |
| Opportunities | `GET /opportunities?category=&role=&location=`, `GET /opportunities/for-me`, `POST/PUT/DELETE` |
| AI Email Assistant | `GET /email-events`, `POST /email-events/extract`, `POST /email-events/{id}/confirm`, `DELETE /email-events/{id}` |

## Notes on scope / next steps

Per the SRS's own MVP definition (section 18), this build prioritizes solid
CRUD + auth + dashboard over speculative AI polish. Natural next steps, in
roughly the order the SRS's roadmap (section 17) suggests:

1. Add integration tests (`spring-boot-starter-test` + `spring-security-test`
   are already on the classpath).
2. Wire up real Gmail/Outlook OAuth (see `EmailEventController`'s Javadoc).
3. Implement one real, authorized opportunity source in
   `OpportunityCollectorScheduler`.
4. Add role-based access (e.g. an admin role for curating `opportunities`)
   if you don't want every user to be able to add/edit them.
5. Move the JWT secret and Anthropic key to a real secrets manager for any
   non-local deployment.
