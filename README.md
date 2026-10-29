# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, and Alembic
migrations (Labs 4–7 of "Advanced Web Technologies").

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
`Copy-Item .env.example .env`. Optional checks with the server running:
`python check_db.py`, `python check_models.py`, `python try_errors.py` (prints `5 of 5 as expected`).

API documentation: http://localhost:8000/docs. The password `change-me` is for
teaching; in a real project, passwords are never put into the repository.

| File | What it does |
|------|--------------|
| `config.py` | `Settings`: `DATABASE_URL` and `SQL_ECHO` from the environment or `.env` |
| `db.py` | `naming_convention`, `Base`, the Engine, `SessionLocal`, the `get_db` dependency |
| `models.py` | SQLAlchemy models `Course`, `Student`, `Enrollment` |
| `schemas.py` | Pydantic schemas of the JSON (camelCase) |
| `main.py` | the routes and the "constraint name → HTTP response" mapping |
| `alembic/`, `alembic.ini` | migrations `0001` (the three tables) and `0002` (`courses.level`) |
| `seed.sql` | initial data; clears the tables first, for the training database only |
| `check_db.py`, `check_models.py`, `try_errors.py` | connection, models-vs-database, and error-response checks |
| `try_n_plus_1.py`, `try_lost_update.py`, `try_delete.py` | bonus experiments 3–5 |

## What changed in the API

- The data lives in PostgreSQL instead of the `data.py` list and the `ENROLLMENTS` list: likes, students, and enrollments survive a server restart (after two likes and a restart `api-design` still had 13 likes).
- `POST /enrollments` now takes `{"courseId": "...", "studentId": 4}` instead of `studentName`: an enrollment refers to a row of `students`. The response has the database's `id` and `enrolledAt`. The Lab 5 enrollment form of the frontend sends `studentName` and gets 422 until it is updated.
- New routes: `POST /students` (`{"name", "email"}`, 201 with the issued `id`) and `GET /stats/courses` (enrollments per course, courses without enrollments get 0). `GET /enrollments` returns `courseTitle` and `studentName`, joined from `courses` and `students`.
- Each course has a required `level` (`beginner` for electives, `intermediate` for the rest after migration `0002`); `GET /courses?level=beginner` filters by it and combines with `is_elective`, `sort`, `page`, `page_size`.
- Database constraint violations become responses by the constraint name: a taken email or a repeated enrollment — 409, an unknown course or student — 404, any other violation — 500. A short name is rejected by Pydantic with 422 before the database.
- Following the Lab 7 skeleton of `main.py`, the Lab 4–5 extras were dropped: the `q` search, `GET /stats`, `Literal` for `sort`, the `Query` bounds on pagination, and `CORS_ORIGINS` (the allowed origin is `http://localhost:3000`).

## Migrations

| Command | What it does |
|---------|--------------|
| `alembic upgrade head` | applies all the missing migrations |
| `alembic downgrade -1` | rolls back the last migration |
| `alembic current` | shows the version of the database |
| `alembic history` | shows the chain of migrations |
| `alembic revision --autogenerate -m "text"` | writes a draft from models vs. database |
| `alembic check` | reports whether the models and the database have diverged |
| `alembic upgrade 0001:0002 --sql` | prints the SQL of a migration without running it |
| `alembic stamp head` | marks the database as up to date without running migrations |

The database from Lab 6 already had the tables, so it was marked with `alembic stamp head`
(`alembic upgrade head` failed with `DuplicateTable: relation "courses" already exists` and
rolled back). Migration `0001` was generated with autogenerate against an empty `scratch`
database. A new database is built with `alembic upgrade head`.

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | the `uq` template in `naming_convention` changed | `check_models.py`: names only in models `enrollments_course_id_unique`, `students_email_unique`; only in database `enrollments_course_id_student_id_key`, `students_email_key` | the template `%(table_name)s_%(column_0_name)s_unique` builds other names than PostgreSQL chose in Lab 6; Alembic would want to recreate the constraints, the error mapping would not find them | restored `%(table_name)s_%(column_0_N_name)s_key`, the check prints `none` twice |
| 2 | `min_length` removed from `StudentIn`, name `N` | `POST /students` → 500; in the log `IntegrityError: (psycopg.errors.CheckViolation) … violates check constraint "students_name_check"`, `Failing row contains (7, N, n@example.com)` | the request reached the database, the CHECK kept the row out, but `students_name_check` is not in `CONSTRAINT_ERRORS`, so it was treated as a bug | restored `Field(min_length=2)`: Pydantic answers 422, the CHECK stays as insurance; `try_errors.py` 5 of 5 |
| 3 | a typo in a constraint name in the dictionary | `POST /students` with `aigerim@example.com` → 500; the log shows the real name `unique constraint "students_email_key"` | the key `students_email_uniq` does not match `exc.orig.diag.constraint_name` | restored the key `students_email_key` |
| 4 | `title` without `NOT NULL` in the database, `alembic check` | `Detected NOT NULL on column 'courses.title'`, `FAILED: New upgrade operations detected: [[('modify_nullable', …` (non-zero exit code); with `varchar(100)`: `Detected type change from VARCHAR(length=100) to Text()`, `modify_type` | `alembic check` compares the models with the real database and sees manual changes | `ALTER COLUMN title SET NOT NULL`, `TYPE text`; `No new upgrade operations detected.` |
| 5 | automatic migration `0002` on existing data | `IntegrityError: (psycopg.errors.NotNullViolation) column "level" of relation "courses" contains null values`, `[SQL: ALTER TABLE courses ADD COLUMN level TEXT NOT NULL]`; `alembic current` stayed `0001` | autogenerate compares schemas, not data: the six existing rows have no value for a NOT NULL column without a default | rewrote `0002` in three steps: add the column with NULL, fill it with an UPDATE, set NOT NULL |
| 6 | `import models` commented out in `env.py` | `--autogenerate` detected `removed table 'courses'`, `'students'`, `'enrollments'` and the index; `0003_oops.py` had `op.drop_table` for all three (here even `courses` before `enrollments`) | without the import the models are not registered, `Base.metadata` is empty, so the whole database looks superfluous | deleted `0003_oops.py` without running it, restored `import models` |
| 7 | a `phone` field in the model without a migration | `Detected added column 'students.phone'`, `FAILED: New upgrade operations detected: [('add_column', None, 'students', Column('phone', …` | the model went ahead of the database: no migration was written | removed the line; `alembic check` clean |
| 8 | two migrations from one revision | `alembic heads`: `0003 (head)`, `0004 (head)`; `upgrade head`: `FAILED: Multiple head revisions are present for given argument 'head'` | `0003` and `0004` both have `down_revision = '0002'`, the history forked | `alembic merge heads --rev-id 0005` gave one head (`0003, 0004 -> 0005 (mergepoint)`, an empty merge); then deleted 0003–0005, one head `0002` |

The `upgrade → downgrade → upgrade` cycle for `0002`: after `downgrade -1` the query
`SELECT id, level FROM courses` failed with `column "level" does not exist`; after
`upgrade head` the levels were back, filled by the rule from the migration.

## Decisions

- The response is chosen by the constraint name because the name is exact: SQLSTATE `23505` is the same for every UNIQUE (we have two, the email and the enrollment pair, with different messages), and the message text depends on the server version and language. The names are fixed by `naming_convention` and checked by `check_models.py`. The error is caught after `commit()` because a check with a `SELECT` in advance is a race: between the check and the INSERT another request can take the same email, and only the database constraint is reliable; one attempt also costs one query instead of two.
- A like is `UPDATE courses SET likes = likes + 1 … RETURNING`: the database adds one under a row lock, so two simultaneous likes give +2. Reading the number into the program and writing a new one back loses an update: both requests read 11 and both write 12 (bonus 4 shows it).
- Migration `0002` is split into three steps because `ADD COLUMN level TEXT NOT NULL` cannot be applied to a table that already has rows: first the column allows NULL, then an UPDATE fills the existing rows by our rule (a decision about data that autogenerate cannot make), then NOT NULL is set. All three run in one transaction, so a failure leaves the database at `0001`.

## Migration 0002 self-check

| What to check | Result |
|---------------|--------|
| Goal and steps | yes: one change — a required `courses.level`; three steps: add with NULL, fill, forbid NULL |
| Data | yes: the six existing rows get `beginner` (electives) or `intermediate` (the rest) by an UPDATE before NOT NULL |
| Renames and drops | yes: nothing is renamed or dropped in `upgrade`; only a new column is added |
| Rollback | yes: `downgrade` drops the column; the levels are lost, which is acceptable here because the rule restores them, but not if users had chosen levels by hand |
| Verification | yes: the SQL from `alembic upgrade 0001:0002 --sql` was read (BEGIN, ALTER, UPDATE, ALTER, version update, COMMIT); the `downgrade -1 → upgrade head` cycle passed; `alembic check` is clean |
| History | yes: `alembic heads` shows one head `0002`; the migration is in the same commit as the model, the schema, the `level` filter, and `seed.sql` |
| Application | yes: responses get a new `level` field, `GET /courses` gets an optional `level` filter; `CourseOut` and `seed.sql` are updated; nothing else changes for clients |

## Bonus

### 1. A CHECK that autogenerate does not see

With `CheckConstraint("level IN ('beginner', 'intermediate', 'advanced')", name="level")` in the model, `alembic check` printed `No new upgrade operations detected.` and `--autogenerate` produced an empty migration (`pass`): Alembic does not compare CHECK constraints, their SQL text is normalized by PostgreSQL differently from the source and cannot be compared reliably. The hand-written `0003` with `op.create_check_constraint(op.f("courses_level_check"), …)` worked: `UPDATE courses SET level = 'expert'` failed with `violates check constraint "courses_level_check"`. `downgrade` with the plain string `"courses_level_check"` failed: `constraint "courses_courses_level_check_check" … does not exist` — a name not wrapped in `op.f` goes through the `ck` template again. With `op.f("courses_level_check")` the rollback worked. `op.f` marks a name as final. The model's `students_email_format` (from the Lab 6 bonus) uses `conv(...)`, the same marker, for the same reason. The file and the model line were removed afterwards.

### 2. Renaming a column without losing data

On a `scratch` copy (`alembic upgrade head` + `seed.sql`), renaming `description` to `summary` in the model made autogenerate propose `add_column('summary', nullable=False)` and `drop_column('description')`: all descriptions would be lost, and the NOT NULL column would fail on a non-empty table. With `op.alter_column("courses", "description", new_column_name="summary")` the migration ran, `alembic check` was clean, and `SELECT id, summary` showed the old descriptions; `downgrade` renamed it back. `alter_column(new_column_name=…)` is one `ALTER TABLE … RENAME COLUMN`: the data stays in place. The "add and drop" pair creates an empty column and destroys the old one with its data. Everything was reverted and `scratch` dropped.

### 3. The N+1 problem

`try_n_plus_1.py`: `lazy enrollments: 6, queries: 7`, `selectinload enrollments: 6, queries: 2`. Lazy loading sends one query for the list and one per course when `course.enrollments` is touched. With `SQL_ECHO=true` N+1 is visible as the same `SELECT … FROM enrollments WHERE %(param_1)s::VARCHAR = enrollments.course_id` repeated after one list query, with `[cached since …]` and a new course id each time. `selectinload` loads all related rows with one extra `SELECT … WHERE course_id IN (…)`; `joinedload` adds a `LEFT OUTER JOIN` to the main query, one query in total but the course columns are repeated for every enrollment, which is good for many-to-one and worse for long one-to-many lists.

### 4. A lost update in the ORM

`try_lost_update.py` with `a.likes += 1` / `b.likes += 1`: `Alice reads 11 | Bob reads 11`, `in the database: 12` — both sessions computed 12 in Python from the same old value and sent `SET likes = 12`. With `a.likes = Course.likes + 1` (the version kept in the repository) the result is 13: the ORM sends `SET likes = likes + 1`, and the database computes it from the current row. Bob's UPDATE waits for the row lock held by Alice's transaction (the lock from Lab 6, bonus 2), then reads the committed 12 and adds one.

### 5. passive_deletes and the cascade in the database

Without `passive_deletes=True` on `Student.enrollments`, `try_delete.py` failed: `IntegrityError: (psycopg.errors.NotNullViolation) null value in column "student_id"`, `[SQL: UPDATE enrollments SET student_id=… ]`, `[parameters: {'student_id': None, 'enrollments_id': 6}]` — the ORM loaded the enrollments and tried to detach them by writing NULL into a required column; nothing was deleted (3 students, 6 enrollments). With `passive_deletes=True` the ORM sends only `DELETE FROM students`, and `ON DELETE CASCADE` in the database removes the enrollment: 2 students, 5 enrollments. The two go together: the database does the cascade, and the ORM must not do its own "detach" in front of it. The data was restored with `seed.sql`.
