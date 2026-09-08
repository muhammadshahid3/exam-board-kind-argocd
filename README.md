# EduResult Portal — Examination Result Management System

A two-service Django application backed by a shared PostgreSQL database:

1. **Student Result Service** (`student_service`) — public-facing portal where
   students search for their result by roll number. Port **8000**.
2. **Admin Management Service** (`admin_service`) — authenticated admin
   portal for managing classes, sections, subjects, students and results.
   Port **8001**.

Both services are independent Django projects that share one PostgreSQL 16
database. The whole stack is orchestrated locally with **Docker Compose
only** — there is no Kubernetes, Helm, CI/CD, Terraform, or cloud deployment
configuration in this repository.

```
                         PostgreSQL
                              |
                 ┌────────────┴────────────┐
                 |                          |
        Student Result Service      Admin Management Service
              Django (8000)               Django (8001)
                 |                          |
                 └────────────┬─────────────┘
                          Docker Compose
```

## Architecture Notes

- The **Admin Management Service** owns the database schema. Its `core` app
  contains the real, managed Django models (`SchoolClass`, `Section`,
  `Subject`, `Student`, `Result`, `ResultItem`) and runs migrations on
  startup.
- The **Student Result Service** connects to the exact same database and
  tables using **unmanaged** mirror models (`managed = False`) — it never
  runs migrations and never writes admin data. This keeps the two services
  fully independent deployable units while sharing one source of truth.
- Grade calculation (90+=A+, 80-89=A, 70-79=B, 60-69=C, 50-59=D, <50=F) and
  percentage/total calculation are computed automatically from subject
  marks — never entered manually by the admin.

## Prerequisites

- Docker
- Docker Compose (v2, `docker compose` CLI)

## Quick Start

```bash
cp .env.example .env
docker compose up --build
```

Once the containers are healthy:

- **Student Portal:** http://localhost:8000
- **Admin Portal:** http://localhost:8001

The admin service automatically creates the default admin account (if it
doesn't already exist) via a custom management command
(`create_default_admin`) that runs on container startup, and applies all
database migrations.

> First boot tip: the student service may start slightly before the admin
> service finishes migrating the shared schema. If the homepage errors on
> the very first request, simply refresh after a few seconds, or run
> `docker compose restart student_service`.

## Local Development Credentials

**⚠️ For local development only. Do not use these credentials in any
public or production deployment.**

| Field    | Value          |
|----------|----------------|
| Username | `admin`        |
| Password | `Admin@12345`  |

Login at: http://localhost:8001/login/

You can override these defaults before first boot by editing `.env`:

```
DEFAULT_ADMIN_USERNAME=admin
DEFAULT_ADMIN_PASSWORD=Admin@12345
DEFAULT_ADMIN_EMAIL=admin@example.com
```

## Using the System

1. Log in to the Admin Portal (http://localhost:8001).
2. **Create Class** (e.g. `9th`, `10th`).
3. **Create Section** (e.g. `A`, `B`) and assign it to a class.
4. **Add Subject** (e.g. `Mathematics`, total marks `100`) per class.
5. **Add Student** with a unique roll number, assigned class & section.
6. **Add Result** → select a student → enter obtained marks per subject
   (validated: `0 <= obtained_marks <= total_marks`).
7. Use **Final Result** in the admin portal to preview any student's
   computed result before "publishing" it.
8. Students can now search their result on the Student Portal
   (http://localhost:8000) using their roll number.

## Project Structure

```
exam-result-system/
├── docker-compose.yml
├── .env.example
├── README.md
│
├── student_service/            # Public student portal (port 8000)
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── requirements.txt
│   ├── manage.py
│   ├── config/                 # settings, urls, wsgi, asgi
│   ├── results/                # unmanaged mirror models, search view
│   ├── templates/
│   └── static/css/
│
└── admin_service/               # Authenticated admin portal (port 8001)
    ├── Dockerfile
    ├── entrypoint.sh
    ├── requirements.txt
    ├── manage.py
    ├── config/                  # settings, urls, wsgi, asgi
    ├── core/                    # canonical models, dashboard, default-admin command
    ├── students/                # student CRUD
    ├── classes_app/             # class CRUD
    ├── sections/                # section CRUD
    ├── subjects/                # subject CRUD
    ├── results/                 # add-result workflow, final result lookup
    ├── templates/
    └── static/css/
```

## Environment Variables

All configuration is environment-driven — no secrets or DB credentials are
hardcoded in source. See `.env.example`:

```
POSTGRES_DB=exam_results
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432

DJANGO_SECRET_KEY=change-this-secret-key
DEBUG=True

STUDENT_PORTAL_URL=http://localhost:8000
ADMIN_PORTAL_URL=http://localhost:8001

DEFAULT_ADMIN_USERNAME=admin
DEFAULT_ADMIN_PASSWORD=Admin@12345
DEFAULT_ADMIN_EMAIL=admin@example.com
```

## Security Notes

- Django's built-in authentication system protects every admin page with
  `@login_required`.
- CSRF protection is enabled on all forms (Django default middleware).
- Roll numbers are unique + indexed at the database level as well as
  validated in forms.
- Marks are validated server-side: `0 <= obtained_marks <= subject.total_marks`.
- The student service has no login system and no write access to admin
  data — it only performs read-only `SELECT` queries against the shared
  schema.
- No credentials are hardcoded in Python source; everything comes from
  environment variables injected by Docker Compose / `.env`.

## Stopping / Resetting

```bash
docker compose down          # stop containers, keep DB volume
docker compose down -v       # stop containers AND wipe DB volume
```

## Tech Stack

- Python 3.12, Django 5.0
- PostgreSQL 16
- Django Templates + Bootstrap 5 + vanilla JavaScript
- Docker & Docker Compose (local development only — no Kubernetes, Helm,
  CI/CD, Terraform, or cloud deployment configuration included)
# exam-board-kind-argocd
