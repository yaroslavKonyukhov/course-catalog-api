# Course Catalog API

A FastAPI backend for the course catalog from the `course-catalog` Next.js project (Advanced Web Technologies, Lab 4).

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
| `GET /courses` | Course list with filtering (`is_elective`), sorting (`sort`) and pagination (`page`, `page_size`) |
| `GET /courses/{course_id}` | A single course or a 404 error |

## What was verified

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
| `/docs` | Swagger UI with three routes and the `Course` schema |
| `/redoc`, `/openapi.json` | ReDoc page and the OpenAPI description of the API |
