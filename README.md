# JobGraph — CognoDB Graph Database Application

A complete graph-first job discovery application built for the **WEXA AI CognoDB Take-Home Assignment**.

## 1. What does it solve?

Job boards normally treat a developer, a skill and a job as rows that are filtered independently. JobGraph models the connections directly:

**Developer → HAS_SKILL → Skill ← REQUIRES ← Job → OFFERS → Company**

A user can therefore discover jobs through shared skills and traverse multiple relationships instead of only filtering columns.

## 2. Why a graph database?

The interesting questions are relationship-heavy:

- Which jobs can a developer reach through the skills they already have?
- Which companies offer jobs requiring a particular skill?
- Which skills connect a developer to a target job?
- What percentage of a job's required skills does a developer already possess?

The recommendation query uses a multi-hop path:

`Developer → Skill → Job → Company`

A relational database can represent these facts with join tables, but increasingly deep relationship queries require several joins and become harder to reason about. In a graph, the traversal is expressed directly through typed relationships.

## 3. Data model

```text
(:Developer)-[:HAS_SKILL]->(:Skill)<-[:REQUIRES]-(:Job)-[:OFFERS]->(:Company)

Developer
  id, name, experience

Skill
  name, category

Job
  id, title, location, level, description, salaryMin, salaryMax

Company
  id, name, industry, size
```

### Relationship semantics

| Relationship | Meaning |
|---|---|
| `HAS_SKILL` | Developer possesses a skill |
| `REQUIRES` | Job requires a skill |
| `OFFERS` | Company offers a job |

## 4. Main graph query

The recommendation query starts at a developer, traverses their skills, finds jobs requiring those skills, then reaches the company offering each job.

```cypher
MATCH (d:Developer {id: $developerId})-[:HAS_SKILL]->(s:Skill)
      <-[:REQUIRES]-(j:Job)-[:OFFERS]->(c:Company)
WITH j, c, count(DISTINCT s) AS matchedSkills
MATCH (j)-[:REQUIRES]->(required:Skill)
WITH j, c, matchedSkills, count(DISTINCT required) AS totalSkills
RETURN j.title, c.name, matchedSkills, totalSkills,
       round(100.0 * matchedSkills / totalSkills) AS matchPercent
ORDER BY matchPercent DESC;
```

This is parameterized and uses the official Neo4j JavaScript driver.

## 5. Tech stack

- React + Vite
- Node.js + Express
- Neo4j JavaScript driver
- CognoDB using Bolt/openCypher
- Plain CSS for the UI

The assignment specifies that CognoDB works with official Neo4j drivers over Bolt, so the application uses `neo4j-driver` rather than a custom database SDK.

## 6. Project structure

```text
cognodb-jobgraph/
├── cypher/
│   ├── 01_schema.cypher
│   └── 02_graph_queries.cypher
├── data/
│   └── seed.json
├── server/
│   ├── db.js
│   ├── index.js
│   ├── queries.js
│   └── seed.js
├── src/
│   ├── main.jsx
│   └── styles.css
├── .env.example
├── .gitignore
├── index.html
├── package.json
├── vite.config.js
└── README.md
```

## 7. CognoDB setup

1. Create a CognoDB Cloud account.
2. Create a free C0 instance.
3. Copy the generated Bolt URI and password.
4. Create `.env` from `.env.example`.
5. Put the values in:

```env
COGNODB_URI=bolt+s://<instance-id>.databases.cognodb.cloud
COGNODB_USER=cognodb
COGNODB_PASSWORD=<generated-password>
PORT=3000
```

Never commit `.env`.

## 8. Run locally

```bash
npm install
npm run seed
npm run dev
```

For a production build:

```bash
npm run build
npm start
```

The production server serves the Vite build and the API from one Express application.

## 9. Seed data

`server/seed.js` contains realistic demo data:

- 5 developers
- 14 skills
- 6 jobs
- 5 companies

The seed operation uses `UNWIND` and parameterized values rather than concatenating user input into Cypher.

## 10. API

| Endpoint | Purpose |
|---|---|
| `GET /api/health` | Database connectivity check |
| `GET /api/stats` | Graph node counts |
| `GET /api/jobs` | List jobs |
| `GET /api/jobs?search=react` | Search jobs |
| `GET /api/jobs/:id` | Job details |
| `GET /api/developers/d1/recommendations` | Multi-hop recommendations |
| `GET /api/developers/d1/jobs/j1/path` | Skills connecting developer and job |
| `GET /api/jobs/:id/neighborhood` | Graph neighborhood |

## 11. Error handling

Database requests are wrapped in `try/catch`. When CognoDB is unreachable, the API returns HTTP 503 and the React UI displays a visible connection error instead of failing silently.

## 12. Deployment

### Render — simplest option

This project can be deployed as a Node web service because Express serves both the API and built React application.

Build command:

```bash
npm install && npm run build
```

Start command:

```bash
npm start
```

Add the three CognoDB environment variables in the Render dashboard.

### Vercel alternative

The React frontend can also be deployed to Vercel, while the Express API is deployed separately. Configure the frontend API base URL accordingly.

## 13. Screen recording plan

Record a 2–3 minute walkthrough:

1. Show the JobGraph landing page.
2. Explain the graph model.
3. Show the job cards and search.
4. Click a job and show its relationship path.
5. Show the recommendation section.
6. Explain that recommendations come from:
   `Developer → Skill → Job → Company`.
7. Briefly show `server/queries.js`.
8. Show the CognoDB instance and seed data.
9. Finish by explaining why the graph model is useful.

## 14. Interview talking points

### Why CognoDB?

Because the primary data relationships are skills, jobs, developers and companies. The application needs graph traversal rather than only CRUD operations.

### Why Neo4j driver?

The assignment explicitly supports official Neo4j drivers with CognoDB. The JavaScript driver lets the backend send parameterized openCypher queries over Bolt.

### What is the strongest query?

The recommendation query because it traverses multiple relationship types and calculates skill overlap for ranking.

### What would you improve next?

- User-selected developer profiles
- Job filtering by salary/location
- Skill-gap recommendations
- Graph visualization
- Authentication
- Pagination and caching
- More robust automated tests
