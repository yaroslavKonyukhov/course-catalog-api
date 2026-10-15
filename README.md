# Course Catalog API

A FastAPI backend for the course catalog from the `course-catalog` Next.js project (Advanced Web Technologies, Labs 4–5).

## How to run

Requires Python 3.10 or newer.

```
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
fastapi dev main.py
```

On macOS and Linux create the environment with `python3 -m venv .venv` and activate it with `source .venv/bin/activate`.

The server starts on http://127.0.0.1:8000, the interactive documentation is at http://127.0.0.1:8000/docs.

## Routes

| Route | Description |
| --- | --- |
| `GET /` | Checks that the server is alive |
| `GET /courses` | Course list with filtering (`is_elective`), title search (`q`), sorting (`sort`) and pagination (`page`, `page_size`) |
| `GET /courses/{course_id}` | A single course or a 404 error |
| `POST /courses/{course_id}/like` | Adds a like and returns the updated course (Lab 5) |
| `POST /enrollments` | Enrolls a student in a course (Lab 5) |
| `GET /enrollments` | Lists the enrollments received so far (Lab 5) |
| `GET /stats` | Catalog statistics (Lab 4 bonus task) |

## What was verified in Lab 4

All requests were sent to the running `fastapi dev main.py` server.

| Request | Result |
| --- | --- |
| `/` | `{"message": "Course Catalog API is running"}` |
| `/courses` | 6 courses: ai-integration, modern-frontend, web-security, backend-fastapi, databases-postgresql, api-design |
| `/courses/web-security` | 200, the Web Security Essentials course |
| `/courses/nope` | 404, `{"detail": "Course not found"}` |
| `/courses?is_elective=true` | 2 courses: ai-integration, api-design |
| `/courses?is_elective=false` | 4 courses: modern-frontend, web-security, backend-fastapi, databases-postgresql |
| `/courses?sort=title` | 6 courses: ai-integration, api-design, backend-fastapi, modern-frontend, databases-postgresql, web-security |
| `/courses?page=1&page_size=2` | ai-integration, modern-frontend |
| `/courses?page=2&page_size=2` | web-security, backend-fastapi |
| `/courses?page=3&page_size=2` | databases-postgresql, api-design |
| `/courses?page=4&page_size=2` | `[]` |
| `/docs` | Swagger UI with the routes and the `Course` schema |
| `/redoc`, `/openapi.json` | ReDoc page and the OpenAPI description of the API |

## Bonus tasks

| Bonus | Request | Result |
| --- | --- | --- |
| `sort` is `Literal["popular", "title"]` | `/courses?sort=banana` | 422, `Input should be 'popular' or 'title'`; `/docs` shows a dropdown |
| `page` and `page_size` are limited with `Query` | `/courses?page=0`, `/courses?page_size=1000` | 422 for both |
| Title search with `q` | `/courses?q=react` | modern-frontend |
| | `/courses?q=api` | backend-fastapi, api-design |
| `GET /stats` with the `Stats` model | `/stats` | `{"total": 6, "totalCredits": 28, "electives": 2}` (camelCase since Lab 5) |

## What changed in Lab 5

### CORS

`CORSMiddleware` allows exactly one origin, `http://localhost:3000` — the Next.js
frontend. The browser lets the frontend's JavaScript read a response from this API
only if the response carries `access-control-allow-origin` with the page's origin.
Requests from the Next.js server (Server Components) do not need it, requests from
Client Components (the filter, the like button, the enrollment form) do.

| Request | Result |
| --- | --- |
| `curl -i -H "Origin: http://localhost:3000" http://localhost:8000/` | 200, `access-control-allow-origin: http://localhost:3000`, `vary: Origin` |
| `curl -i -H "Origin: http://evil.example" http://localhost:8000/` | 200, no `access-control-allow-origin` |
| preflight `OPTIONS /enrollments` from `http://localhost:3000` | 200, `access-control-allow-methods` includes POST, `access-control-max-age: 600` |

### camelCase in JSON

All models inherit from `CamelModel` (`alias_generator=to_camel`,
`validate_by_name=True`, `validate_by_alias=True`). Python keeps snake_case,
the JSON uses camelCase, so it matches the TypeScript types in the frontend.
Query parameters keep their names.

| Request | Result |
| --- | --- |
| `/courses` | keys `id, title, description, credits, isElective, likes` |
| `/courses?is_elective=true` | 2 courses: ai-integration, api-design |
| `/courses?isElective=true` | all 6 courses — an unknown parameter is ignored |

### New routes

Likes and enrollments live in memory and reset on every restart of the server.

| Request | Result |
| --- | --- |
| `POST /courses/web-security/like` | 200, `{"id": "web-security", …, "isElective": false, "likes": 22}` |
| `POST /courses/nope/like` | 404, `{"detail": "Course not found"}` |
| `POST /enrollments` `{"courseId": "web-security", "studentName": "Aigerim"}` | 201, `{"courseId": "web-security", "studentName": "Aigerim", "id": 1}` |
| the same with `"studentName": "A"` | 422, `"loc": ["body", "studentName"]`, `"msg": "String should have at least 2 characters"` |
| the same with `"courseId": "nope"` | 404, `{"detail": "Course not found"}` |
| `GET /enrollments` | `[{"courseId": "web-security", "studentName": "Aigerim", "id": 1}]` |
