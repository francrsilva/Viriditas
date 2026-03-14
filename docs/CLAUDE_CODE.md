# Claude Code — Rules & Guidance

> This file governs how Claude Code behaves when working on the Viriditas codebase.
> A summary copy lives at `.claude/CLAUDE.md` for automatic loading.

---

## Identity

You are working on **Viriditas**, a Portuguese fiscal optimization platform built with Django, PostgreSQL, Celery, Redis, and Stripe. You build clean, typed, testable Python code following GoF patterns documented in `docs/DESIGN_PATTERNS.md`.

---

## Mandatory Pre-Read

Before making **any** code changes, read these files:

1. `docs/PROJECT_STRUCTURE.md` — know where everything lives
2. `docs/DESIGN_PATTERNS.md` — know which pattern applies
3. `docs/CONVENTIONS.md` — know the style rules
4. `docs/DATABASE.md` — if touching data layer

---

## Rules

### Architecture Rules

1. **Views are thin.** Views in `apps/*/views.py` handle HTTP concerns only: validate input, call a service or facade, render a template. No business logic, no raw ORM queries.
2. **Business logic lives in services.** `apps/*/services/` contains all domain logic. Services implement GoF patterns.
3. **Repository pattern for all data access.** Never call `Model.objects.filter()` directly in views or services. Always go through `apps/*/repositories.py`.
4. **Facades for complex operations.** If a view needs 3+ service calls, use a Facade.
5. **External APIs use Adapter classes only.** Never call `httpx.get()` or `requests.get()` outside of `adapters/`.
6. **New tax regime = new Strategy class.** Never add `if/elif` branches to existing strategies.
7. **Async work goes through Celery.** Email sending, API polling, report generation — always a Celery task.
8. **Django signals for decoupled reactions.** Profile creation on user save, notification triggers, etc.
9. **Templates use includes for reusability.** Shared components live in `templates/components/`. Use `{% include %}` with context.
10. **Stimulus controllers are small and focused.** One controller per interactive behavior. Under 50 lines each.

### Code Style Rules

1. **Type hints everywhere.** All function signatures, method parameters, and return types. Use `from __future__ import annotations` at top of every file.
2. **No bare `except`.** Always catch specific exceptions.
3. **ABC for interfaces.** Use `abc.ABC` + `@abstractmethod` for all pattern interfaces.
4. **Dataclasses or Pydantic for DTOs.** Never pass raw dicts between layers.
5. **Django forms for all input validation.** Validate in the form, not in the view or service.
6. **Domain exceptions.** Services throw `ViriditasError` subclasses (defined in `apps/core/exceptions.py`). Views catch and render appropriate error responses.
7. **No magic strings.** Use `TextChoices` enums for all categorical data.

### File Rules

1. **One class per file** for services, commands, strategies, observers, and adapters. Repositories can have multiple classes if they belong to the same app.
2. **Naming:** `snake_case` for everything (files, functions, variables). `PascalCase` for classes only.
3. **Tests mirror source.** `apps/fiscal/services/strategies/irs_strategy.py` → `apps/fiscal/tests/test_strategies.py`.
4. **Max file size:** If a file exceeds ~200 lines, split it.

### Pattern Application Rules

| Situation | Required Pattern | Don't Do This |
|-----------|-----------------|---------------|
| Different calc logic per user type | Strategy | `if user.user_type == "INDIVIDUAL"` chains |
| Reacting to legislative updates | Observer + Celery | Hardcoded notification calls in the monitor |
| User actions that need undo/audit | Command | Direct repo calls without history |
| Creating type-specific service bundles | Factory | Constructor sprawl with conditionals |
| Calling external APIs | Adapter | Raw `httpx`/`requests` in services |
| Complex multi-service operations | Facade | Fat views doing everything |
| Cross-cutting concerns (cache, log) | Decorator | Mixing cache logic into business methods |

### Django-Specific Rules

1. **Custom User Model** — `AUTH_USER_MODEL = "accounts.User"` is set. Never reference `django.contrib.auth.User`.
2. **Settings split** — Use `settings.dev`, `settings.prod`, `settings.test`. Common config in `settings.base`.
3. **URL namespacing** — Every app has `app_name` in its `urls.py`. Use `{% url 'accounts:login' %}` in templates.
4. **Admin** — Register all models in admin. Admin is the first debugging tool.
5. **Migrations** — Never edit migrations by hand. If you need to fix a migration, create a new one.
6. **Static files** — Collected via `collectstatic`. Tailwind compiled via CLI. Never inline styles.

---

## Workflow

### When adding a new feature:

1. **Identify which app** it belongs to (accounts, fiscal, legislative, payments, core)
2. **Check if a pattern applies** (see decision guide in DESIGN_PATTERNS.md)
3. **Define model changes first** → `makemigrations` → `migrate`
4. **Build server-side** (repository → service → view)
5. **Build templates** (layout → page → partials)
6. **Add Stimulus controller** if interactivity needed
7. **Write tests** alongside implementation
8. **Update docs** if the feature introduces new patterns

### When fixing a bug:

1. **Write a failing test first**
2. **Fix in the correct layer** (don't patch symptoms in the template)
3. **Verify no pattern violations** were introduced

---

## Don'ts

- **Don't create files outside the structure.** Check PROJECT_STRUCTURE.md.
- **Don't install packages without reason.** Prefer what's in requirements. Ask before adding dependencies.
- **Don't skip type hints.** "I'll type it later" = technical debt.
- **Don't put Portuguese in code.** Variable names, comments, and docs in English. Only user-facing strings (templates, form labels) are in Portuguese.
- **Don't duplicate logic.** If two apps need it, it goes in `apps/core/`.
- **Don't use `render()` with complex logic.** Compute context in the view method or facade, then render.
- **Don't put Celery task logic in the task function.** Tasks should call services. The task is just the async wrapper.

---

## File Placement Quick Reference

| What you're creating | Where it goes |
|---------------------|---------------|
| New page/view | `apps/{app}/views.py` + `apps/{app}/templates/{app}/` |
| New model | `apps/{app}/models.py` → run `makemigrations` |
| Domain service | `apps/{app}/services/` |
| Strategy class | `apps/fiscal/services/strategies/` |
| Command class | `apps/fiscal/services/commands/` |
| Observer class | `apps/legislative/services/observers/` |
| Facade | `apps/{app}/services/facade.py` |
| Repository | `apps/{app}/repositories.py` |
| Form | `apps/{app}/forms.py` |
| Celery task | `apps/{app}/tasks.py` |
| URL routes | `apps/{app}/urls.py` → include in `viriditas/urls.py` |
| Stimulus controller | `static/js/controllers/` |
| Reusable template | `templates/components/` |
| App-specific template | `apps/{app}/templates/{app}/` |
| Template partial | `apps/{app}/templates/{app}/partials/` |
| External API client | `adapters/` |
| Shared utility | `apps/core/` |
| Management command | `apps/core/management/commands/` |
| Test | `apps/{app}/tests/` or `tests/` (E2E) |
