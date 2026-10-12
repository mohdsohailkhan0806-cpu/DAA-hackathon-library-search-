# Digital Library Search Optimizer

**Fast and Intelligent Resource Search Using DAA** — a complete full-stack web application built for a Design and Analysis of Algorithms hackathon. Four hand-written search engines (linear scan, binary search, hash-based lookup, inverted index) race against each other over a live library database on every single query, with real comparison counting, relevance ranking, full CRUD, a polished dashboard and a bundled Postman/Thunder Client collection.

---

## 1. Project Overview

Digital libraries hold thousands of resources. Finding the right book, paper or journal requires search — and *how* you search is exactly what a DAA course teaches. This project turns a library catalog into a live algorithm laboratory:

- A **React frontend** (dashboard, search console, resource manager, DAA demo lab, API console)
- A **REST backend** exposing `/api/resources`, `/api/search`, `/api/search/compare` and `/api/health`
- A **persistent database** storing 63 seeded library resources (books, research papers, journals, reports)
- **Four DAA algorithms**, each implemented manually in dedicated files under `src/lib/algorithms/`, each instrumenting its own comparison count during execution — nothing is hard-coded.

## 2. Problem Statement

> Design an optimized search system for a digital library that can retrieve resources by title, author, keyword, category or publication year — and demonstrate, with measured evidence, why indexed search structures outperform exhaustive scanning.

## 3. Existing Solution

Naive library search typically relies on a **linear scan**: iterate every record, compare every field, return what matches. This is `O(N)` per query and gets slower with every resource added. Some systems hand the problem to a database `LIKE '%query%'` query — which is, internally, still a full table scan.

## 4. Proposed Solution

Move the work to **preprocessing time** and amortize it across queries:

| Structure | Build cost | Query cost | Idea |
| --- | --- | --- | --- |
| Linear scan | none | `O(N)` | probe everything, every time |
| Sorted array + binary search | `O(N log N)` once | `O(log N)` | halve a title-sorted array per probe |
| Hash index | `O(N)` once | `O(1)` average | `key → [resources]` in a `Map`, one probe per lookup |
| Inverted index | `O(N · tokens)` once | `O(k)` | `term → [posting lists]`, touch only matching documents |

The backend runs all four engines against the same live data (`GET /api/search/compare`) and returns the **actual** number of comparisons each one executed — the frontend visualizes the race.

## 5. Features

- **Dashboard** — live stats, category marquee, real-time algorithm telemetry fed by `/api/search/compare`
- **Search page** — query input, field scope (all/title/author/keyword/category/year), per-query algorithm selection, relevance-ranked result cards, comparison counts, search time, complexity
- **Algorithm comparison dashboard** — animated bar chart + spec table + winner verdict, all driven by the live API response
- **Resource management** — add / view / edit / delete / search resources through real REST calls with inline validation errors and toast notifications
- **DAA demonstration lab** — presets, pseudocode, theoretical complexity table, and a live race with per-engine explanations
- **API testing console** — every endpoint with purpose, params, editable request bodies, curl snippets, sample responses and a **live Send button**
- **Health check** — header status pill polls `GET /api/health` and shows *Backend · Connected*
- **Postman / Thunder Client collection** — `postman/Digital-Library.postman_collection.json` (also downloadable in-app)
- **Auto-seeding** — the catalog self-populates with 63 resources the first time the API is hit on an empty database

## 6. Technology Stack

| Requirement | Implementation |
| --- | --- |
| React.js frontend | React 19 + Next.js App Router (TypeScript, JSX → TSX) |
| HTML/CSS responsive UI | Tailwind CSS v4 design system |
| Axios or Fetch API | `fetch` wrapper with latency measurement (`src/lib/client-api.ts`) |
| Node.js backend | Node.js runtime |
| Express-style REST API | Next.js REST route handlers (`src/app/api/*`) — same HTTP contract: methods, paths, JSON bodies, status codes |
| CORS configuration | `Access-Control-Allow-*` headers + `OPTIONS` pre-flight on every route |
| Environment variables | `.env` (`DATABASE_URL`) |
| Database + ORM model | PostgreSQL via Drizzle ORM (`resources` table; `keywords` stored as a real array column) — the platform-managed MongoDB-equivalent document store |
| API testing | Postman collection + Thunder Client compatible + browser console |

> **Stack note:** this repository is packaged as a single deployable Next.js application so the frontend and REST backend ship together. The API layer is a true REST backend (independently testable in Postman exactly like an Express server), and PostgreSQL+JSON-arrays provides the same document persistence that Mongoose/MongoDB would. If you must split into `frontend/` + `backend/` folders, the entire backend lives in `src/app/api/` + `src/lib/`, and the entire frontend in `src/app/(pages)` + `src/components/` — the separation is one folder move away.

## 7. Project Structure

```
digital-library-search-optimizer/
├── src/
│   ├── app/                        # Frontend (React pages)
│   │   ├── page.tsx                #   Dashboard
│   │   ├── search/page.tsx         #   Search console
│   │   ├── resources/page.tsx      #   Resource management
│   │   ├── algorithms/page.tsx     #   DAA demonstration lab
│   │   ├── api-testing/page.tsx    #   API console
│   │   ├── layout.tsx · globals.css
│   │   └── api/                    # Backend (REST API)
│   │       ├── health/route.ts                # GET  /api/health
│   │       ├── resources/route.ts             # GET·POST /api/resources
│   │       ├── resources/[id]/route.ts        # GET·PUT·DELETE /api/resources/:id
│   │       ├── search/route.ts                # GET  /api/search
│   │       └── search/compare/route.ts        # GET  /api/search/compare
│   ├── components/                 # React components (header, charts, consoles…)
│   ├── lib/
│   │   ├── algorithms/             # ★ DAA engines (manual implementations)
│   │   │   ├── linear-search.ts    #   O(N)    — field-by-field scan
│   │   │   ├── binary-search.ts    #   O(log N)— hand-written merge sort + halving
│   │   │   ├── hash-search.ts      #   O(1) avg— Map-based hash indexes
│   │   │   ├── inverted-index.ts   #   O(k)    — term → postings
│   │   │   ├── rank.ts             #   relevance scoring & sorting
│   │   │   └── types.ts            #   shared contract, tokenizer, registry
│   │   ├── search-service.ts       # orchestration: DB → engines → timing
│   │   ├── validation.ts           # server-side input validation
│   │   ├── api-utils.ts            # envelope helpers, CORS, error handling
│   │   ├── client-api.ts           # typed frontend fetch client
│   │   └── seed-data.json / .ts    # 63 sample resources
│   └── db/
│       ├── schema.ts               # `resources` table (Drizzle model)
│       └── index.ts                # pooled database client
├── scripts/
│   └── seed.mjs                    # standalone seed runner
├── postman/
│   └── Digital-Library.postman_collection.json
├── public/postman/…                # downloadable copy
├── .env
└── README.md
```

## 8. System Architecture

```
User
  ↓  (query / CRUD action)
React Frontend  (fetch client, src/lib/client-api.ts)
  ↓  HTTP + JSON
REST API  (/api/* route handlers, CORS enabled)
  ↓
DAA Algorithm Engine  (linear / binary / hash / inverted + ranker)
  ↓  loads collection
Database  (resources table via Drizzle ORM)
  ↓
JSON Response  { success, message, data, comparisons, complexity, ... }
  ↓
React UI  (ranked cards, charts, toasts, status pill)
```

## 9. DAA Algorithms

All four engines live in `src/lib/algorithms/` and count their own work — the API’s `comparisons` field is a real counter incremented inside the loops.

### Algorithm 1 — Linear Search `O(N)`
Walks resources `0..N-1`; for each one, probes title, author, every keyword, category (and year when scoped). `comparisons++` on every probe. Worst case = full collection scan.

### Algorithm 2 — Binary Search `O(log N)`
Builds a title-sorted array with a **hand-written merge sort** (comparisons counted separately as preprocessing), then runs a manual lower-bound halving loop — no `indexOf`/`find` — and expands forward while titles start with the query prefix.

### Algorithm 3 — Hash-Based Search `O(1)` average
Builds `Map` hash tables of normalized **titles, authors, categories, keywords and years** → resource lists:

```
machine learning  →  [resource1, resource5, resource12]
```

A query is `Map.get(key)` probes (1 per table in scope). Lookup itself is `O(1)` on average; total cost also scales with the `k` results returned.

### Algorithm 4 — Inverted Index `O(k)`
Tokenizes the catalog into posting lists per field:

```
python   → [1, 5, 8]
machine  → [2, 4, 10]
learning → [2, 4, 10]
```

A query fetches the posting lists of its own tokens and merges them by id — the resource table is never scanned. Every index lookup and every posting entry scanned increments the operation counter.

### Relevance Ranking

```
Exact title match        +10
Exact author match        +8
Exact keyword match       +5
Exact category match      +3
Publication year match    +3
(partial/sub-string matches earn smaller bonuses)
```

Results are sorted by descending score with deterministic tie-breaking.

## 10. API Documentation

Base URL: `http://localhost:3000` · all responses JSON · CORS enabled.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| GET | `/api/health` | Liveness probe |
| GET | `/api/resources` | All resources (optional `?page=1&limit=10`) |
| GET | `/api/resources/:id` | One resource |
| POST | `/api/resources` | Create (validated) |
| PUT | `/api/resources/:id` | Partial update (validated) |
| DELETE | `/api/resources/:id` | Delete |
| GET | `/api/search` | `?q=…&field=all&algorithm=inverted` |
| GET | `/api/search/compare` | `?q=…` — race all four engines |

### Example — create

```bash
curl -X POST http://localhost:3000/api/resources \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Data Structures and Algorithms",
    "author": "Thomas Cormen",
    "keywords": ["data structures", "algorithms"],
    "category": "Computer Science",
    "publicationYear": 2024,
    "type": "Book",
    "description": "Study of data structures and algorithms."
  }'
```

```json
{ "success": true, "message": "Resource created successfully", "data": { "id": 64, "title": "Data Structures and Algorithms", "...": "..." } }
```

### Example — search

```bash
curl "http://localhost:3000/api/search?q=machine%20learning"
```

```json
{
  "success": true,
  "query": "machine learning",
  "field": "all",
  "algorithm": "inverted-index",
  "complexity": "O(k)",
  "comparisons": 27,
  "resultCount": 6,
  "searchTimeMs": 0.412,
  "data": [ { "title": "Introduction to Machine Learning", "author": "Andrew Ng", "relevanceScore": 15, "...": "..." } ]
}
```

### Example — comparison

```bash
curl "http://localhost:3000/api/search/compare?q=machine%20learning"
```

```json
{
  "success": true,
  "query": "machine learning",
  "results": {
    "linear":        { "comparisons": 255, "complexity": "O(N)",          "resultCount": 6 },
    "binary":        { "comparisons": 6,   "complexity": "O(log N)",      "resultCount": 0 },
    "hash":          { "comparisons": 4,   "complexity": "O(1) average",  "resultCount": 6 },
    "invertedIndex": { "comparisons": 27,  "complexity": "O(k)",          "resultCount": 6 }
  }
}
```

*(Values above are illustrative; your deployment reports the live measured counts.)*

### Error envelopes

```json
{ "success": false, "message": "Title is required", "errors": ["Title is required", ...] }
{ "success": false, "message": "Resource not found" }
{ "success": false, "message": "Search query cannot be empty" }
{ "success": false, "message": "Internal server error" }
```

### Validation rules

- `title` — required, non-empty string (≤ 300 chars)
- `author` — required, non-empty string (≤ 200 chars)
- `keywords` — must be an array of non-empty strings (1–25)
- `category` — required string
- `publicationYear` — integer between `1450` and next year
- `type` — one of `Book · Research Paper · Article · Journal · Thesis · Report`
- search `q` — required, non-empty

## 11. Database Setup

The app reads `DATABASE_URL` from `.env`:

```bash
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/app_db
```

Apply the schema (creates the `resources` table):

```bash
npx drizzle-kit push
```

Point it at any PostgreSQL instance (local Docker, Neon, Supabase…) by editing `.env` — MongoDB users can equivalently point at a Mongo-backed service layer; the HTTP contract stays identical.

## 12. Installation & Running

```bash
# 1. install dependencies
npm install

# 2. configure the database
echo 'DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:5432/app_db' > .env
npx drizzle-kit push

# 3. seed 63 sample resources (safe to re-run; use --force to reset)
node scripts/seed.mjs

# 4. run the app (frontend + backend together)
npm run dev        # development  → http://localhost:3000
# or:
npm run build && npm start   # production
```

> The database also **auto-seeds on first request** when empty, so a fresh deployment works immediately.

### Seed commands

```bash
node scripts/seed.mjs           # inserts only when the catalog is empty
node scripts/seed.mjs --force   # wipes and re-seeds all 63 resources
```

## 13. Postman / Thunder Client Testing

1. Get the collection: download in-app from the **API Testing** page, or use `postman/Digital-Library.postman_collection.json` in this repo.
2. **Postman** → Import → choose the file. **Thunder Client** → Collections → menu → Import.
3. Set the collection variable `baseUrl` (default `http://localhost:3000`).
4. Folders included: **Health**, **Resources** (list, paginated, by id, create, update, delete), **Search** (one-engine + compare), and **Error Handling** (400/404 examples).

Every request also works directly in the browser:

```
http://localhost:3000/api/health
http://localhost:3000/api/resources?page=1&limit=10
http://localhost:3000/api/search?q=python&field=keyword
http://localhost:3000/api/search/compare?q=machine%20learning
```

## 14. Time Complexity Summary

| Algorithm | Best | Average | Worst | Preprocessing |
| --- | --- | --- | --- | --- |
| Linear Search | O(1) | O(N) | O(N) | — |
| Binary Search | O(1) | O(log N) | O(log N) | O(N log N) sort |
| Hash-Based Search | O(1) | O(1) | O(N) collisions | O(N) build |
| Inverted Index | O(1) | O(k) | O(N) ultra-common term | O(N · tokens) build |

`k` = number of matched posting entries. The overall query cost can also depend on the number of results returned.

## 15. Future Improvements

- BM25 / TF-IDF ranking as an extension of the current scoring scheme
- Autocomplete from the hash index + trie (`O(prefix)` suggestions)
- B-tree index on `publication_year` for range queries
- Fuzzy matching (Levenshtein / n-gram indexes)
- Query caching with memoization + time-decayed invalidation
- Collection sharding to demo distributed indexing
- Export comparison runs to CSV for the viva/presentation

---

*Built for the DAA hackathon. Every number in the UI is measured live — no simulated results, no hard-coded comparison counts.*
