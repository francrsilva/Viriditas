# CLAUDE.md — Viriditas Project Instructions

> This file is read automatically by Claude Code at the start of every session.

## What is this project?

Viriditas is a Portuguese fiscal optimization platform built with Django 5, PostgreSQL, Celery + Redis, Stimulus.js, Tailwind CSS, and Stripe. It serves two user types: Individual (IRS) and Empresa (IRC).

## Before you write any code

Read these docs first — they are the source of truth:

- `docs/PROJECT_STRUCTURE.md` — where files go
- `docs/DESIGN_PATTERNS.md` — which GoF patterns to use and when (Python examples)
- `docs/CONVENTIONS.md` — naming, style, git conventions
- `docs/DATABASE.md` — Django models and migration strategy
- `docs/API_INTEGRATION.md` — external API adapters and Celery tasks
- `docs/CLAUDE_CODE.md` — full rules (extended version of this file)

## Critical rules (summary)

1. Views are thin — no business logic in `views.py`
2. All business logic in `apps/*/services/` using GoF patterns
3. All data access through Repository classes (`apps/*/repositories.py`)
4. External APIs through Adapter classes (`adapters/`)
5. Complex operations through Facade classes (`apps/*/services/facade.py`)
6. Type hints on all functions — no bare `except`, no magic strings
7. New tax regime = new Strategy class, never `if/elif` branches
8. Async work always goes through Celery tasks
9. Templates use `{% include %}` for reusable components
10. English in code, Portuguese only in user-facing strings

## App dependency order

```
core ← accounts ← fiscal ← legislative
                 ← payments
```
No circular imports. Use Django signals for cross-app communication.

## Common commands

```bash
python manage.py runserver                  # Dev server
python manage.py makemigrations             # Create migrations
python manage.py migrate                    # Apply migrations
python manage.py seed_fiscal_data           # Load tax brackets
python manage.py createsuperuser            # Admin user
celery -A viriditas worker -l info          # Celery worker
celery -A viriditas beat -l info            # Celery scheduler
pytest                                       # Run tests
pytest --cov=apps                            # Tests with coverage
```

## Tech stack

Django 5 · PostgreSQL 16 · Celery + Redis · Stimulus.js · Tailwind CSS · Stripe · pytest-django · Playwright · Docker Compose
