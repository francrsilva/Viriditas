# 🌿 Viriditas

**Fiscal Optimization Platform for the Portuguese Landscape**

Viriditas helps individuals and businesses navigate Portuguese tax law, discover fiscal optimization strategies, and stay current with legislative changes from Diário da República and Portal das Finanças.

> *"Viriditas"* — Hildegard von Bingen's concept of greening life-force. Here, it represents financial vitality through informed fiscal decisions.

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Backend | **Django 5.x** | Batteries-included, mature ORM, admin panel |
| Database | **PostgreSQL 16** | Relational integrity for fiscal data |
| Templates | **Django Templates** + **Stimulus.js** | Server-rendered HTML with sprinkles of interactivity |
| Task Queue | **Celery** + **Redis** | Async jobs: API polling, notifications, reports |
| Cache / Broker | **Redis** | Celery broker + view/session caching |
| Payments | **Stripe** | Subscription billing (future premium tier) |
| CSS | **Tailwind CSS** (via django-tailwind or standalone CLI) | Utility-first styling |
| Testing | **pytest-django** + **Playwright** | Unit/integration + E2E |
| Deployment | **Docker Compose** (dev) → **production TBD** | Reproducible environments |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────┐
│                    PRESENTATION                      │
│  Django Templates + Stimulus.js Controllers          │
│  ┌────────────┐  ┌────────────┐  ┌───────────────┐  │
│  │ Individual  │  │  Empresa   │  │   Shared      │  │
│  │  Templates  │  │  Templates │  │   Partials    │  │
│  └──────┬──────┘  └─────┬──────┘  └───────┬───────┘  │
│         └───────┬───────┘                 │          │
│                 ▼                         │          │
│  ┌──────────────────────────────┐         │          │
│  │   Django Views (thin)        │◄────────┘          │
│  │   + Stimulus Controllers     │                    │
│  └──────────┬───────────────────┘                    │
├─────────────┼────────────────────────────────────────┤
│             ▼          DOMAIN LAYER                  │
│  ┌──────────────────────────────────────────────┐    │
│  │           Domain Services                    │    │
│  │  (Strategy, Observer, Command, Facade)       │    │
│  └───────────────────┬──────────────────────────┘    │
│                      ▼                               │
│  ┌─────────────────────────┐  ┌───────────────────┐  │
│  │  Repository Layer       │  │  External APIs    │  │
│  │  (Django ORM wrapped    │  │  (Adapter Pattern) │  │
│  │   in Repository classes)│  │                   │  │
│  └───────────┬─────────────┘  └────────┬──────────┘  │
│              ▼                         ▼             │
│  ┌───────────────────┐  ┌─────────────────────────┐  │
│  │   PostgreSQL      │  │  Diário da República    │  │
│  │                   │  │  Portal das Finanças    │  │
│  └───────────────────┘  └─────────────────────────┘  │
├──────────────────────────────────────────────────────┤
│                  ASYNC LAYER                         │
│  ┌───────────────────────────────────────────────┐   │
│  │  Celery Workers                               │   │
│  │  • Legislative polling tasks (periodic)       │   │
│  │  • Email notifications (on-demand)            │   │
│  │  • Report generation (on-demand)              │   │
│  │  • Stripe webhook processing                  │   │
│  └───────────────────────────────────────────────┘   │
│  ┌───────────────────────────────────────────────┐   │
│  │  Redis (broker + result backend + cache)      │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

---

## GoF Design Patterns Used

See [docs/DESIGN_PATTERNS.md](docs/DESIGN_PATTERNS.md) for full rationale and Python code examples.

| Pattern | Category | Where Used |
|---------|----------|-----------|
| **Strategy** | Behavioral | Tax calculation engines (IRS vs IRC vs regimes) |
| **Observer** | Behavioral | Legislative update notifications (via Django signals + Celery) |
| **Command** | Behavioral | User fiscal actions (undo/redo, audit log) |
| **Factory Method** | Creational | Creating user-type-specific services |
| **Singleton** | Creational | API client instances (module-level in Python) |
| **Adapter** | Structural | External API integration wrappers |
| **Facade** | Structural | Simplified service interfaces for views |
| **Repository** | Structural* | Data access abstraction over Django ORM |
| **Decorator** | Structural | Adding caching/logging to services (Python decorators) |

---

## Getting Started

```bash
# 1. Clone
git clone <repo-url> && cd viriditas

# 2. Create virtual environment
python -m venv .venv && source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements/dev.txt

# 4. Environment
cp .env.example .env
# Fill in DATABASE_URL, REDIS_URL, STRIPE keys, API keys

# 5. Database
python manage.py migrate
python manage.py seed_fiscal_data   # Custom command: loads tax brackets, categories

# 6. Redis (required for Celery)
# Ensure Redis is running on localhost:6379

# 7. Run
python manage.py runserver           # Django dev server
celery -A viriditas worker -l info   # Celery worker (separate terminal)
celery -A viriditas beat -l info     # Celery beat scheduler (separate terminal)
```

---

## Documentation Index

| Document | Purpose |
|----------|---------|
| [PROJECT_STRUCTURE.md](docs/PROJECT_STRUCTURE.md) | Full directory tree with annotations |
| [DESIGN_PATTERNS.md](docs/DESIGN_PATTERNS.md) | GoF patterns: rationale + Python code examples |
| [DATABASE.md](docs/DATABASE.md) | Models, ERD, migration strategy |
| [API_INTEGRATION.md](docs/API_INTEGRATION.md) | Diário da República & Finanças APIs |
| [CLAUDE_CODE.md](docs/CLAUDE_CODE.md) | Rules & guidance for Claude Code sessions |
| [CONVENTIONS.md](docs/CONVENTIONS.md) | Code style, naming, commit messages |
| [ROADMAP.md](docs/ROADMAP.md) | Feature phases and milestones |
