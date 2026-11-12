# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, Alembic and
GraphQL on Strawberry as a second entrance (Labs 4–8, "Advanced Web Technologies").

## How to run

You need Python 3.11 or newer and PostgreSQL 16 or newer.

Once, as the `postgres` superuser, create the role and the database:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Then, from the project folder:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
alembic upgrade head
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql
alembic check
fastapi dev main.py
```

On Windows (PowerShell) the activation is `.venv\Scripts\Activate.ps1` and the copy is
`Copy-Item .env.example .env`. With the server running, the checks in a second window:
`python try_errors.py` (5 of 5), `python try_graphql.py` (8 of 8),
`python try_n_plus_1.py` (3 and 3), `python measure.py`.

REST docs: http://localhost:8000/docs. GraphiQL: http://localhost:8000/graphql.
The password `change-me` is for training; real projects don't commit passwords.

| File | What it does |
|------|--------------|
| `services.py` | application rules (`create_student`, `enroll`, `like_course`) and `ServiceError` with a code |
| `problem.py` | REST errors in the Problem Details format |
| `graphql_api.py` | the GraphQL schema: types, queries, mutations, DataLoaders, protection |
| `counter.py` | counts SQL queries per HTTP request into the `X-SQL-Queries` header |
| `main.py` | REST routes; mounts `/graphql` |
| `config.py`, `db.py`, `models.py`, `schemas.py`, `alembic/` | settings, Engine and sessions, models, JSON schemas, migrations (Lab 7) |
| `try_errors.py`, `try_graphql.py`, `try_n_plus_1.py`, `try_protection.py`, `flood_likes.py`, `measure.py`, `print_schema.py` | check and measurement scripts |
| `seed.sql`, `more_students.sql` | starting data; 17 more students for experiment 4 |

## What changed in the API

- REST errors are Problem Details (RFC 9457): content type `application/problem+json`, body `{type, title, status, detail, instance}`. A rule error has its own `type` (`https://example.com/problems/already-enrolled`, `…/email-taken`, `…/course-not-found`, `…/student-not-found`); Starlette's own errors (unknown address, wrong method) get `about:blank` and the status phrase, `405` keeps the `Allow` header. The rules and their codes live in `services.py`, the translation into statuses in `problem.py`; `main.py` has no `HTTPException` or `IntegrityError` left.
- `/graphql` is a second entrance to the same database and services: one address, one schema (`Course`, `Student`, `Enrollment`; queries `courses`, `course`, `students`, `student`, `enrollments`). A browser GET opens GraphiQL, an editor with hints and schema docs; an ordinary client sends `POST` with JSON `{query, variables}` and gets JSON `{data, errors}`.
- Mutations `enroll(input: {courseId, studentId})` and `likeCourse(id)` return an envelope with the value and `userErrors`. An expected rule error (repeated enrollment, unknown course or student) is in `data` as `userErrors[].code` with the same code as in REST; an error in the query text or an unexpected failure is in `errors`. GraphQL answers both with status 200; mutations over `GET` are rejected with 400.
- Every response has the `X-SQL-Queries` header: the number of SQL queries the server ran for this HTTP request. It made the N+1 problem visible and proves the effect of the DataLoaders.
- Protection of `/graphql`: depth at most 5, at most 10 aliases, `limit` must be 1..100, `id` must be a number; the text of unexpected errors is replaced with `Unexpected error.` (the cause stays in the server log). Introspection is controlled by `GRAPHQL_INTROSPECTION` (`true` by default, `false` turns it off).
- New REST routes: `GET /students/{id}` and `GET /students/{id}/enrollments` (404 `student-not-found` for an unknown student).

## Example queries

The "My courses" query (in the GraphiQL Variables panel: `{"id": "2"}`):

```graphql
query MyCourses($id: ID!) {
  student(id: $id) {
    name
    enrollments {
      id
      course { title credits level }
    }
  }
}
```

```json
{"data": {"student": {"name": "Daniyar Bekov", "enrollments": [
  {"id": "3", "course": {"title": "Modern Frontend: React & Next.js", "credits": 5, "level": "intermediate"}},
  {"id": "4", "course": {"title": "Backend Foundations: FastAPI", "credits": 5, "level": "intermediate"}},
  {"id": "5", "course": {"title": "AI/LLM Integration", "credits": 5, "level": "beginner"}}]}}}
```

The `enroll` mutation (Variables: `{"input": {"courseId": "web-security", "studentId": "3"}}`):

```graphql
mutation Enroll($input: EnrollInput!) {
  enroll(input: $input) {
    enrollment { id enrolledAt course { title } }
    userErrors { code message }
  }
}
```

The first answer and the answer on a repeated enrollment:

```json
{"data": {"enroll": {"enrollment": {"id": "10", "enrolledAt": "2026-11-13T01:23:28.748776+05:00",
  "course": {"title": "Web Security Essentials"}}, "userErrors": []}}}

{"data": {"enroll": {"enrollment": null, "userErrors": [
  {"code": "ALREADY_ENROLLED", "message": "Student is already enrolled in this course"}]}}}
```

## REST vs GraphQL comparison

The "My courses" screen of student 2; numbers from `measure.py`, starting data.

| Criterion | REST | GraphQL | Source |
|-----------|------|---------|--------|
| Requests and rounds per screen | 5 requests in 2 rounds (student + enrollments, then 3 courses) | 1 request in 1 round | `measure.py`, step 8 |
| Bytes in the responses | 1145: full courses (description, likes, isElective), names and times in enrollments | 339: only name, title, credits, level | `measure.py`, step 8 |
| SQL queries per screen | 5 (1 student + 1 enrollments + 1 per course) | 3 (student, enrollments, courses in one `IN`) | `measure.py`, `X-SQL-Queries` |
| SQL growth with more students | each route is one query with a known shape; the screen grows by 1 request and 1 query per course of the student | before DataLoaders 7 → 27 and 7 → 24 (3 → 20 students); after them 3 and 3 regardless of the number of rows | `try_n_plus_1.py`, experiments 4–5 |
| Error in the query text | a real status: 404 for an unknown address, 422 for invalid parameters | status 200, `data: null`, `errors[].message` with `Did you mean …`; the client must check `errors` | experiments 1 and 8 |
| Expected rule error (repeated enrollment) | 409, `application/problem+json`, `type …/already-enrolled` | 200, `data.enroll.userErrors[0].code = ALREADY_ENROLLED` | steps 1 and 4 |
| HTTP cache | `GET` on a unique URL: browser, proxy and CDN cache it with `Cache-Control` / `ETag` | `POST` to one `/graphql` address is not cached by HTTP; needs a client cache or persisted queries over `GET` | lecture section 14 |
| Protection from "expensive" queries | the cost is fixed by the route (one query, its own limit); a rate limiter counts requests | the client builds the query: needed depth and alias limits, argument checks, error masking; without them a depth-6 cycle ran, 12 likes went in one POST (24 SQL), SQL text leaked | experiments 6 and 7 |

**Conclusion.** For a screen that joins several resources and shows a few fields of each, like "My courses", GraphQL wins: 1 request instead of 5, one round instead of two, 339 bytes instead of 1145, and with DataLoaders 3 SQL queries that don't grow with the data. Simple public reads of one resource (the course list and a course page) I would keep on REST: they are cached by HTTP for free, errors have honest statuses, and their cost is fixed by the route. GraphQL needs extra work that REST does not: DataLoaders against N+1, protection against expensive queries, and a client that checks `errors`, because status 200 says nothing.

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | a typo in a field name, an argument name in `snake_case` | `titel`: HTTP 200, `data: null`, `Cannot query field 'titel' on type 'Course'. Did you mean 'title', 'level', or 'likes'?` (line 1, column 30); `is_elective`: 200, `Unknown argument 'is_elective' on field 'Query.courses'. Did you mean 'isElective'?`. REST `/courses/no-such` gave a real 404 | the engine validates the query against the schema before execution and reports problems in `errors`; HTTP status only describes the transport. Strawberry exposes snake_case names as camelCase | used `title` and `isElective`; a client must check `errors`, not the status |
| 2 | `\| None` removed from the `course` query, a nonexistent course | the schema became `course(id: ID!): Course!`; `course(id: "nope")` → `data: null`, `Cannot return null for non-nullable field Query.course.`, `path: ["course"]`; with aliases `a` (exists) and `b` (nope) the whole `data` was null, `path: ["b"]`; in the server log `TypeError` | `!` is a promise; a null in a non-null field propagates to the nearest nullable parent, here the whole `data`, and the valid field `a` was lost | returned `CourseType \| None`: `{"a": {"title": "API Design: REST vs GraphQL"}, "b": null}` |
| 3 | a mutation with the GET method | a GET query worked (200, likes 13); `GET /graphql?query=mutation{likeCourse…}` → 400, `text/plain`, `mutations are not allowed when using GET`; likes stayed 13 | GET must be safe: it can be repeated, prefetched and cached, a link must not change data, so the server accepts mutations only in POST | nothing to fix: mutations are sent with POST |
| 4 | `more_students.sql`: 3 and 20 students, the SQL counter | 3 students: 7 and 7 SQL queries; 20 students (23 enrollments): 27 and 24. The SQL log: 1 students + 3 enrollments queries (`student_id = 1, 2, 3`) + 3 course queries | lazy loading in resolvers: one query per parent object (N+1). 1 + 20 + 6 and 1 + 6 + 17; the identity map saves repeated courses. Status, JSON size and speed in GraphiQL don't show it, only the counter | DataLoaders (step 6): 3 and 3 on 20 students |
| 5 | a DataLoader batch function that doesn't sort rows by key | the students query: `data: null`, `Received wrong number of results in dataloader, expected: 3, received: 6` (`WrongNumberOfResultsReturned`); the enrollments query: no errors, but enrollments 2, 4, 5, 6 got someone else's course (2: Backend instead of AI/LLM, 4: AI/LLM instead of Backend, 5 and 6: Backend instead of AI/LLM) | the loader hands out results by position; with `ORDER BY id DESC` the rows came as modern-frontend, backend-fastapi, ai-integration while the keys were modern-frontend, ai-integration, backend-fastapi. The second error is worse: the length matched, so nothing was reported and the data was silently wrong | restored the original function: rows are grouped by key value in `groups` and returned in key order |
| 6 | six dangerous queries without protection and with it | see the table below | without limits the server executes whatever the client wrote and puts the full exception text into `errors` | `QueryDepthLimiter(5)`, `MaxAliasesLimiter(10)`, `check_limit`, `parse_id`, `MaskErrors(should_mask)` |
| 7 | introspection off, a typo in a field name | `GRAPHQL_INTROSPECTION=false`: `{ __schema … }` rejected (`GraphQL introspection has been disabled, but the requested query contained the field '__schema'.` and the same for `queryType`); ordinary queries and `try_graphql.py` 8 of 8 kept working; `{ cours { title } }` → `Did you mean 'course' or 'courses'?` | `DisableIntrospection` blocks only `__schema`/`__type`; validation hints still name real fields, so the schema can be guessed field by field. Introspection is not authorization | removed the line from `.env`, introspection is on again for GraphiQL |
| 8 | client failures: a typo, no `errors` check, the API stopped | typo: page 500, `Error [GraphQLRequestError]: Cannot query field 'titel' on type 'Course'. Did you mean …` at `gql (lib/graphql.ts:30:34)`; check removed: `TypeError: Cannot destructure property 'student' of '(intermediate value)' as it is null.`; API stopped: `TypeError: fetch failed` | GraphQL errors come with status 200, so `res.ok` is true; only the `errors` check turns them into an exception with the real cause. Without it `data: null` crashes the page later and the cause is lost. With the API down there is no answer at all | restored both files; started the API again, the page is 200 |

Experiment 6, the same queries without protection and with it (`try_protection.py`, `flood_likes.py`):

| Query | Without protection | With protection |
|-------|--------------------|-----------------|
| a depth-6 cycle | executed, data arrived | `'Cycle' exceeds maximum operation depth of 5` |
| 12 aliases | executed; 12 `likeCourse` in one POST: likes 12 … 23, 24 SQL queries | `12 aliases found. Allowed: 10`, 0 SQL queries, likes unchanged |
| `limit: -1` | `(psycopg.errors.InvalidRowCountInLimitClause) LIMIT must not be negative` + the SQL text and `[parameters: {'param_1': -1}]` | `limit must be between 1 and 100` |
| `id: "abc"` | `invalid literal for int() with base 10: 'abc'` | `Invalid ID: 'abc'` (`student` is null, the rest of `data` stays) |
| `id` of 20 digits | `(psycopg.errors.NumericValueOutOfRange) bigint out of range` + the SQL text and parameters | `Unexpected error.` (the `DataError` is in the server log) |
| introspection | the schema arrived | the schema arrived (it is on); with `GRAPHQL_INTROSPECTION=false` rejected |

Answers. Without protection the client learned the table and column names, the full SQL text with the parameter values, the driver (`psycopg`) and the ORM (the `sqlalche.me` link), and that `id` is parsed with Python's `int()`. We wrote `check_limit` and `parse_id` because `Invalid ID: 'abc'` tells the client what to fix, while `Unexpected error.` only says that something broke; masking is the safety net for cases we did not foresee. With `max_depth=2` the "My courses" query (`student → enrollments → course`, depth 3) and the try_graphql case `student(id: 2) → enrollments → course` would be rejected.

## Decisions

- The rules and their errors are written once in `services.py`, and each layer only translates the code (`PROBLEMS` in REST, `UserError` in GraphQL). Checking everything in both layers would duplicate the constraint names and messages; the copies drift apart, and then the same repeated enrollment gives different answers in REST and GraphQL. Now a new rule is added in one place, and both APIs report the same code.
- Loaders live in `Context` because it is created per HTTP request: the loader's cache by key must not outlive the request, otherwise one user would get another's data and changes in the database would not be seen; the loader also holds this request's session. The batch function matches rows to keys by value because the database returns rows in any order, and a loader distributes results by position: experiment 5 showed silently wrong courses when the lengths happened to match.
- Two layers: foreseen bad input (`limit`, `id`) gets a clear `GraphQLError` that helps the client fix the query; everything unforeseen (a database failure, a bug) is hidden by `MaskErrors` as `Unexpected error.`, so the internals (SQL, tables, parameters) don't leak, while the full text stays in the server log. Masking alone would hide useful messages, our checks alone would leak everything we did not think of.

## Bonus

### 1. Validation errors as Problem Details too

`problem.py` got a third handler, for `RequestValidationError`. Before it `python try_post.py N bad` gave `422 application/json` with FastAPI's `{"detail": [...]}`; now it gives `422 application/problem+json`, `type …/validation-error`, `title "Validation failed"`, `detail "2 invalid value(s) in the request"` and an `errors` list: `{"in": "body", "field": "name", "detail": "String should have at least 2 characters"}` and the same for `email`. `try_errors.py` prints `validation-error` in the fifth row instead of `-`; the same format works for query parameters (`/courses?page=abc` → `"in": "query", "field": "page"`). The list comes from `exc.errors()`: Pydantic collects every invalid value with `loc` (where: `body`/`query`/`path` and the field) and `msg`, so the client fixes all fields at once. `errors` is an extension member: RFC 9457 defines five standard members and allows a problem type to add its own; a client that doesn't know `validation-error` still reads `title`, `status` and `detail`.

### 2. The createStudent mutation: one rule for REST and GraphQL

Before: `python try_post.py "  " sp@example.com` → `201`, a student whose name is two spaces (Pydantic's `min_length=2` counts the spaces). The rule "at least two characters after `strip()`" went into `services.create_student` as `ServiceError("INVALID_NAME", …)`, `problem.py` got `"INVALID_NAME": (422, "invalid-name", "Invalid name")`, GraphQL got `CreateStudentInput`, `CreateStudentPayload` and `createStudent`. GraphQL: name `N` → `userErrors: INVALID_NAME`; a taken email → `EMAIL_TAKEN`; a new student → `student {id: "5"}`, `userErrors: []`. REST, the same two spaces → `422 application/problem+json`, `type …/invalid-name`. `try_graphql.py` still 8 of 8, the schema has `createStudent(input: CreateStudentInput!): CreateStudentPayload!`. With the `INVALID_NAME` line commented out in `PROBLEMS`, GraphQL kept answering `userErrors: INVALID_NAME` (it doesn't use the table), while REST answered `500 Internal Server Error` with `KeyError: 'INVALID_NAME'` in the server log. The rule is written once in the services layer, so both APIs enforce it the same way; a new error code needs one more step — a row in `PROBLEMS`, the REST translation table — otherwise REST can't translate the code and fails with 500.

### 3. A blocking resolver: four requests at once

`blocking_app.py` (port 8001, `SELECT pg_sleep(0.5)` through the session from `db.py`) and `blocking_client.py` were run from the project root and not added to the repository. Two runs, four simultaneous requests to each entrance:

| Entrance | Run 1 | Run 2 |
|----------|-------|-------|
| REST, `def` route | 0.95 s | 0.82 s |
| GraphQL, sync resolver | 2.32 s | 2.32 s |
| GraphQL, `async` + `run_in_threadpool` | 0.80 s | 0.80 s |

FastAPI runs a `def` route in a thread pool, so the four waits for the database overlap. Strawberry calls a synchronous resolver directly in the event-loop thread: while it waits for `pg_sleep`, the loop can't accept other requests, and the waits line up (4 × 0.5 s plus overhead). The difference between the second and the third variant is the line `return await run_in_threadpool(pause, info.context.db)`: the same blocking call is moved to a thread, and the loop stays free. Our parallel variants took 0.8 s rather than 0.5 s, probably because new connections to PostgreSQL are opened on Windows for the first concurrent requests; the ratio is the same. In our project the queries take milliseconds, so the loop is blocked for a moment and nobody notices; it becomes a problem with slow queries (big reports, a slow or remote database, external HTTP calls in resolvers) under many concurrent users: every user then waits for everyone else's queries. One session must not be shared between fields that run in parallel threads, so `run_in_threadpool` with the shared session is not a universal fix.
