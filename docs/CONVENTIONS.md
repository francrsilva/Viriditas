# Code Conventions — Viriditas

---

## Python Style

- **Python 3.12+** required.
- **Formatter:** `black` (line length 99).
- **Linter:** `ruff` (replaces flake8, isort, pyflakes).
- **Type checker:** `mypy` (strict mode).
- **All files** start with `from __future__ import annotations`.

## Naming

| Thing | Convention | Example |
|-------|-----------|---------|
| Files | snake_case | `tax_calculator.py` |
| Classes | PascalCase | `TaxCalculatorService` |
| ABCs | PascalCase (no `Base` prefix unless abstract) | `TaxStrategy` |
| Functions/methods | snake_case | `calculate_effective_rate()` |
| Constants | SCREAMING_SNAKE | `MAX_DEDUCTION_SAUDE` |
| Django models | PascalCase singular | `Deduction`, `User` |
| Django managers | PascalCase + Manager | `UserManager` |
| Template files | snake_case | `dashboard_individual.html` |
| Template partials | _snake_case (leading underscore) | `_deduction_row.html` |
| Stimulus controllers | snake_case + _controller | `deduction_form_controller.js` |
| CSS classes | Tailwind utilities (no custom BEM) | `class="bg-white p-4"` |
| URL names | snake_case | `deduction-list`, `login` |
| Celery tasks | snake_case | `poll_legislative_updates` |

## File Suffixes

Service files use descriptive names within their directory:

| Location | Purpose |
|----------|---------|
| `services/strategies/irs_strategy.py` | Strategy implementation |
| `services/commands/add_deduction.py` | Command implementation |
| `services/observers/email_notifier.py` | Observer implementation |
| `services/facade.py` | Facade for the app |
| `services/calculator.py` | Calculator service |
| `repositories.py` | All repositories for the app |
| `tasks.py` | All Celery tasks for the app |
| `forms.py` | All Django forms for the app |

## Imports

Order (enforced by `ruff`):
1. Standard library (`os`, `datetime`, `decimal`, `abc`)
2. Third-party (`django`, `celery`, `httpx`, `stripe`)
3. Local apps (`apps.core`, `apps.accounts`, `apps.fiscal`)
4. Relative imports (within the same app only)

```python
# Good
from __future__ import annotations

import logging
from decimal import Decimal

from django.db import models
from celery import shared_task

from apps.core.models import TimeStampedModel
from apps.accounts.models import User

from .repositories import DeductionRepository
```

## Views

- **Class-based views** for CRUD operations (Django generic views).
- **Function-based views** for simple pages or custom logic.
- **Views are thin:** get data from facade/service, pass to template.
- **Use mixins** from `apps/core/mixins.py` for auth and permissions.

```python
# Good — thin view using Facade
class DashboardView(LoginRequiredMixin, View):
    def get(self, request):
        facade = FiscalFacade(request.user)
        context = facade.get_dashboard_context()
        return render(request, context.pop("template"), context)

# Bad — fat view with business logic
class DashboardView(LoginRequiredMixin, View):
    def get(self, request):
        deductions = Deduction.objects.filter(user=request.user)  # Direct ORM!
        income = request.user.profile.gross_income
        if request.user.user_type == "INDIVIDUAL":  # Conditional logic!
            tax = calculate_irs(income, deductions)   # Business logic in view!
        # ... this goes on
```

## Templates

- **Inheritance:** `base.html` → `layouts/dashboard.html` → `fiscal/dashboard_individual.html`
- **Components:** Reusable snippets in `templates/components/` via `{% include %}`.
- **Partials:** App-specific fragments in `templates/{app}/partials/_name.html`.
- **Context:** Pass data from views; never import Python in templates.
- **Stimulus:** Wire interactivity via `data-controller`, `data-action` attributes.

```html
{# Good — using a reusable component #}
{% include "components/_card.html" with title="Deduções" content=deductions_html %}

{# Good — Stimulus controller #}
<div data-controller="deduction-form">
  <input data-deduction-form-target="amount" type="number">
  <button data-action="click->deduction-form#submit">Guardar</button>
</div>
```

## Stimulus Controllers

- **One file per controller** in `static/js/controllers/`.
- **Naming:** `{name}_controller.js` → registers as `data-controller="{name}"` (Stimulus auto-converts).
- **Keep small:** Under 50 lines. If larger, split into multiple controllers.
- **No business logic.** Controllers handle DOM interaction only. Data flows from server via templates.

## Git Conventions

### Commit Messages

Format: `type(scope): description`

Types: `feat`, `fix`, `refactor`, `docs`, `style`, `test`, `chore`, `perf`

Scopes: `accounts`, `fiscal`, `legislative`, `payments`, `core`, `adapters`, `templates`, `static`, `config`

Examples:
```
feat(accounts): add individual login with NIF validation
fix(fiscal): correct IRS bracket boundary calculation
refactor(legislative): extract observer interface from monitor
docs(patterns): add Decorator pattern Python example
test(fiscal): add deduction command unit tests
chore(config): update Celery beat schedule for 6h polling
```

### Branches

- `main` — production
- `develop` — integration
- `feat/scope-description` — feature branches
- `fix/scope-description` — bug fixes

## Error Handling

```python
# apps/core/exceptions.py

class ViriditasError(Exception):
    """Base exception for all domain errors."""
    def __init__(self, message: str, code: str = "UNKNOWN", status_code: int = 500):
        self.message = message
        self.code = code
        self.status_code = status_code
        super().__init__(message)


class NotFoundError(ViriditasError):
    def __init__(self, entity: str, identifier: str):
        super().__init__(f"{entity} not found: {identifier}", "NOT_FOUND", 404)


class ValidationError(ViriditasError):
    def __init__(self, message: str):
        super().__init__(message, "VALIDATION_ERROR", 400)


class FiscalCalculationError(ViriditasError):
    def __init__(self, message: str):
        super().__init__(message, "FISCAL_CALC_ERROR", 422)
```

## Testing

- **Framework:** `pytest-django` (not Django's TestCase).
- **Fixtures:** Shared fixtures in `tests/conftest.py`. Per-app fixtures in `apps/{app}/tests/conftest.py`.
- **Factories:** `factory_boy` for model creation in `tests/factories.py`.
- **Naming:** `test_{thing}_{behavior}` → `test_irs_strategy_calculates_progressive_brackets`.
- **Structure:** Arrange-Act-Assert in every test.
- **Coverage:** `pytest-cov` with minimum 80% target.

```python
# Good test structure
class TestIRSStrategy:
    def test_calculates_tax_for_first_bracket(self, irs_strategy):
        result = irs_strategy.calculate_tax(Decimal("5000"), [])
        assert result.effective_rate < Decimal("14")

    def test_applies_deductions_correctly(self, irs_strategy, sample_deductions):
        result = irs_strategy.calculate_tax(Decimal("30000"), sample_deductions)
        assert result.net_tax < result.gross_tax
```

## i18n

- Code, comments, docs, variable names — **English**.
- User-facing text (templates, form labels, error messages) — **Portuguese**.
- Future: `django.utils.translation` with `.po` files for full i18n support.
