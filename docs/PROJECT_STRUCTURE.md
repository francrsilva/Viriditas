# Project Structure

> Every directory has a purpose. Every file has one job.
> This is a Django project using the "domain apps" approach — each Django app owns a domain, not a feature.

```
viriditas/
├── .env.example                         # Environment template (never commit .env)
├── .claude/
│   └── CLAUDE.md                        # Claude Code auto-read instructions
├── manage.py                            # Django management entry point
├── pyproject.toml                       # Project metadata, tool configs (black, ruff, mypy)
├── requirements/
│   ├── base.txt                         # Shared dependencies
│   ├── dev.txt                          # Dev tools (pytest, debug-toolbar, etc.)
│   └── prod.txt                         # Production (gunicorn, sentry-sdk, etc.)
│
├── docker-compose.yml                   # PostgreSQL + Redis for local dev
├── Dockerfile                           # Production image
│
├── viriditas/                           # ─── DJANGO PROJECT CONFIG ───
│   ├── __init__.py                      # Celery app auto-discovery
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py                      # Shared settings
│   │   ├── dev.py                       # Development overrides
│   │   ├── prod.py                      # Production overrides
│   │   └── test.py                      # Test overrides
│   ├── urls.py                          # Root URL configuration
│   ├── wsgi.py                          # WSGI entry point
│   ├── asgi.py                          # ASGI entry point (if needed)
│   └── celery.py                        # Celery app configuration
│
├── apps/                                # ─── DJANGO APPS (one per domain) ───
│   │
│   ├── accounts/                        # ─── AUTH & USER MANAGEMENT ───
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── models.py                    # User, UserProfile (custom user model)
│   │   ├── admin.py                     # Django admin config
│   │   ├── forms.py                     # Login, registration forms
│   │   ├── views.py                     # Login, register, logout views
│   │   ├── urls.py                      # /accounts/* routes
│   │   ├── managers.py                  # Custom user manager
│   │   ├── repositories.py             # UserRepository
│   │   ├── services.py                  # Account-related domain services
│   │   ├── factories.py                 # UserServiceFactory (Factory Method)
│   │   ├── signals.py                   # Post-save signals (create profile, etc.)
│   │   ├── tests/
│   │   │   ├── __init__.py
│   │   │   ├── test_models.py
│   │   │   ├── test_views.py
│   │   │   ├── test_services.py
│   │   │   └── test_forms.py
│   │   ├── templates/
│   │   │   └── accounts/
│   │   │       ├── login.html           # Login page (receives user_type)
│   │   │       ├── register.html        # Registration page
│   │   │       └── profile.html         # User profile
│   │   ├── templatetags/
│   │   │   └── account_tags.py          # Custom template filters
│   │   └── migrations/
│   │
│   ├── fiscal/                          # ─── FISCAL CORE (tax calculations) ───
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── models.py                    # Deduction, TaxSimulation
│   │   ├── admin.py
│   │   ├── forms.py                     # Deduction forms, simulation input forms
│   │   ├── views.py                     # Dashboard views, simulation views
│   │   ├── urls.py                      # /fiscal/* routes
│   │   ├── repositories.py             # DeductionRepository, SimulationRepository
│   │   │
│   │   ├── services/                    # ─── DOMAIN SERVICES (GoF patterns) ───
│   │   │   ├── __init__.py
│   │   │   ├── strategies/              # ── Strategy Pattern ──
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py              # TaxStrategy ABC
│   │   │   │   ├── irs_strategy.py      # IRS calculation
│   │   │   │   ├── irc_strategy.py      # IRC calculation
│   │   │   │   ├── irs_simplificado.py  # Regime Simplificado
│   │   │   │   └── nhr_strategy.py      # Non-Habitual Resident
│   │   │   │
│   │   │   ├── commands/                # ── Command Pattern ──
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py              # FiscalCommand ABC
│   │   │   │   ├── add_deduction.py     # AddDeductionCommand
│   │   │   │   ├── remove_deduction.py  # RemoveDeductionCommand
│   │   │   │   └── invoker.py           # CommandInvoker (history, undo)
│   │   │   │
│   │   │   ├── calculator.py            # TaxCalculatorService (context for Strategy)
│   │   │   └── facade.py               # FiscalFacade (simplifies view calls)
│   │   │
│   │   ├── tests/
│   │   │   ├── __init__.py
│   │   │   ├── test_models.py
│   │   │   ├── test_strategies.py
│   │   │   ├── test_commands.py
│   │   │   ├── test_views.py
│   │   │   └── test_facade.py
│   │   ├── templates/
│   │   │   └── fiscal/
│   │   │       ├── dashboard_individual.html
│   │   │       ├── dashboard_empresa.html
│   │   │       ├── deduction_list.html
│   │   │       ├── deduction_form.html
│   │   │       ├── simulation.html
│   │   │       └── partials/            # HTMX-friendly / Stimulus partials
│   │   │           ├── _deduction_row.html
│   │   │           ├── _tax_summary.html
│   │   │           └── _optimization_tip.html
│   │   ├── templatetags/
│   │   │   └── fiscal_tags.py           # Currency formatting, tax display helpers
│   │   └── migrations/
│   │
│   ├── legislative/                     # ─── LEGISLATIVE INTELLIGENCE ───
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── models.py                    # LegislativeUpdate, UserAlertSubscription
│   │   ├── admin.py
│   │   ├── views.py                     # Alert feed, subscription management
│   │   ├── urls.py                      # /legislative/* routes
│   │   ├── repositories.py             # LegislativeUpdateRepository
│   │   │
│   │   ├── services/                    # ─── DOMAIN SERVICES ───
│   │   │   ├── __init__.py
│   │   │   ├── monitor.py              # LegislativeMonitor (Observer Subject)
│   │   │   ├── observers/              # ── Observer Pattern ──
│   │   │   │   ├── __init__.py
│   │   │   │   ├── base.py             # LegislativeObserver ABC
│   │   │   │   ├── email_notifier.py   # Sends email via Celery task
│   │   │   │   └── in_app_notifier.py  # Creates in-app notification
│   │   │   └── facade.py              # LegislativeFacade
│   │   │
│   │   ├── tasks.py                     # Celery tasks (poll APIs, send alerts)
│   │   ├── tests/
│   │   │   ├── __init__.py
│   │   │   ├── test_models.py
│   │   │   ├── test_monitor.py
│   │   │   ├── test_observers.py
│   │   │   └── test_tasks.py
│   │   ├── templates/
│   │   │   └── legislative/
│   │   │       ├── feed.html
│   │   │       ├── subscriptions.html
│   │   │       └── partials/
│   │   │           ├── _alert_card.html
│   │   │           └── _subscription_toggle.html
│   │   └── migrations/
│   │
│   ├── payments/                        # ─── STRIPE BILLING ───
│   │   ├── __init__.py
│   │   ├── apps.py
│   │   ├── models.py                    # Subscription, PaymentHistory
│   │   ├── admin.py
│   │   ├── views.py                     # Checkout, portal, webhook endpoint
│   │   ├── urls.py                      # /payments/* routes
│   │   ├── services.py                  # StripeService (wraps stripe SDK)
│   │   ├── webhooks.py                  # Stripe webhook handler
│   │   ├── tasks.py                     # Async webhook processing via Celery
│   │   ├── tests/
│   │   │   └── ...
│   │   ├── templates/
│   │   │   └── payments/
│   │   │       ├── pricing.html
│   │   │       └── success.html
│   │   └── migrations/
│   │
│   └── core/                            # ─── SHARED / CROSS-CUTTING ───
│       ├── __init__.py
│       ├── apps.py
│       ├── models.py                    # Abstract base models (TimeStampedModel, etc.)
│       ├── mixins.py                    # View mixins (LoginRequired, UserTypeRequired)
│       ├── decorators.py               # Python decorators (caching, logging)
│       ├── exceptions.py               # Domain exceptions (ViriditasError, NotFoundError)
│       ├── validators.py               # Shared validators (NIF, IBAN, etc.)
│       ├── context_processors.py       # Global template context
│       ├── middleware.py               # Custom middleware (user type, etc.)
│       ├── management/
│       │   └── commands/
│       │       ├── seed_fiscal_data.py  # Load tax brackets, categories
│       │       └── poll_legislative.py  # Manual trigger for API polling
│       ├── tests/
│       │   └── ...
│       └── templatetags/
│           └── core_tags.py            # Shared template helpers
│
├── adapters/                            # ─── EXTERNAL API ADAPTERS ───
│   ├── __init__.py                      # (Separate from apps — not a Django app)
│   ├── base.py                          # BaseApiClient ABC (Template Method)
│   ├── diario_republica.py             # DiarioRepublicaAdapter
│   ├── portal_financas.py              # PortalFinancasAdapter
│   └── factory.py                       # ApiClientFactory
│
├── static/                              # ─── STATIC ASSETS ───
│   ├── css/
│   │   ├── input.css                    # Tailwind input file
│   │   └── output.css                   # Compiled Tailwind (gitignored in dev)
│   ├── js/
│   │   ├── application.js              # Stimulus app entry point
│   │   └── controllers/               # Stimulus controllers
│   │       ├── user_type_controller.js  # Homepage Individual/Empresa toggle
│   │       ├── deduction_form_controller.js
│   │       ├── simulation_controller.js
│   │       ├── alert_toggle_controller.js
│   │       └── flash_controller.js      # Auto-dismiss flash messages
│   ├── images/
│   │   └── logo.svg
│   └── fonts/
│
├── templates/                           # ─── GLOBAL TEMPLATES ───
│   ├── base.html                        # Root layout (head, nav, footer, Stimulus init)
│   ├── layouts/
│   │   ├── auth.html                    # Centered minimal layout for login/register
│   │   └── dashboard.html              # Sidebar + topbar layout for logged-in users
│   ├── partials/
│   │   ├── _navbar.html
│   │   ├── _sidebar.html
│   │   ├── _footer.html
│   │   ├── _flash_messages.html
│   │   └── _loading_spinner.html
│   ├── pages/
│   │   └── home.html                   # Homepage: Individual / Empresa / Cadastre-se
│   └── components/                      # ─── REUSABLE TEMPLATE COMPONENTS ───
│       ├── _button.html                 # {% include "components/_button.html" with ... %}
│       ├── _card.html
│       ├── _form_field.html
│       ├── _empty_state.html
│       ├── _data_table.html
│       └── _modal.html
│
├── tests/                               # ─── TOP-LEVEL TEST CONFIG ───
│   ├── conftest.py                      # Shared fixtures (users, deductions, etc.)
│   ├── factories.py                     # Factory Boy model factories
│   └── e2e/
│       ├── test_auth_flow.py            # Playwright E2E
│       └── test_dashboard_flow.py
│
└── docs/                                # ─── DOCUMENTATION ───
    ├── PROJECT_STRUCTURE.md             # This file
    ├── DESIGN_PATTERNS.md              # GoF patterns: rationale + Python examples
    ├── DATABASE.md                      # Models, ERD, migration strategy
    ├── API_INTEGRATION.md              # External API docs
    ├── CLAUDE_CODE.md                  # Claude Code rules
    ├── CONVENTIONS.md                  # Code style, naming, commit messages
    └── ROADMAP.md                      # Feature phases
```

---

## Directory Principles

**`viriditas/`** (project config) — Settings, URL root, WSGI/ASGI, Celery config. Nothing else. Split settings by environment.

**`apps/`** — One Django app per domain. Each app is self-contained: models, views, forms, services, repositories, templates, tests. Apps communicate through services and signals, never by importing each other's models directly.

**`apps/core/`** — Shared infrastructure: base models, mixins, exceptions, validators, management commands. Every other app can depend on `core`, but `core` depends on no other app.

**`adapters/`** — External API clients. **Not a Django app** — these are pure Python classes with no Django model dependencies. They transform external data into domain objects that apps consume.

**`templates/`** — Global templates: base layouts, shared partials, reusable components. App-specific templates live inside each app's `templates/` directory.

**`static/js/controllers/`** — Stimulus controllers. Each controller handles one interactive behavior. Controllers are small (typically under 50 lines) and compose with HTML data attributes.

**`tests/`** — Top-level test config: conftest fixtures, Factory Boy factories, E2E tests. Unit/integration tests live inside each app's `tests/` directory.

---

## App Dependency Rules

```
core ← accounts ← fiscal ← legislative
                 ← payments
```

- `core` depends on nothing (except Django itself)
- `accounts` depends on `core`
- `fiscal` depends on `core` and `accounts`
- `legislative` depends on `core` and `accounts`
- `payments` depends on `core` and `accounts`
- `adapters/` depends on nothing (pure Python + requests/httpx)
- **No circular dependencies.** If two apps need to communicate, use Django signals or a shared service in `core`.
