# Design Patterns — GoF in Viriditas (Python/Django)

> Every pattern here earns its place. We don't use patterns for ceremony — each one solves a specific problem in the Portuguese fiscal domain.

---

## 1. Strategy Pattern (Behavioral)

**Problem:** Tax calculations differ fundamentally between IRS (individual) and IRC (corporate). New regimes (Regime Simplificado, NHR) add more variants. We can't hardcode `if/elif` chains.

**Solution:** Define a `TaxStrategy` ABC. Each tax regime is a concrete strategy. The `TaxCalculatorService` delegates to whichever strategy is active.

```python
# apps/fiscal/services/strategies/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from decimal import Decimal


@dataclass
class TaxResult:
    gross_tax: Decimal
    net_tax: Decimal
    effective_rate: Decimal
    breakdown: list[dict]
    optimization_suggestions: list[str]


@dataclass
class DeductionLimits:
    saude: Decimal
    educacao: Decimal
    habitacao: Decimal
    geral: Decimal


class TaxStrategy(ABC):
    """Abstract base for all tax calculation strategies."""

    @abstractmethod
    def calculate_tax(self, income: Decimal, deductions: list) -> TaxResult:
        ...

    @abstractmethod
    def get_applicable_brackets(self) -> list[dict]:
        ...

    @abstractmethod
    def get_max_deductions(self) -> DeductionLimits:
        ...
```

```python
# apps/fiscal/services/strategies/irs_strategy.py

from decimal import Decimal
from .base import TaxStrategy, TaxResult, DeductionLimits


class IRSStrategy(TaxStrategy):
    """IRS (individual income tax) progressive bracket calculation."""

    BRACKETS = [
        (Decimal("7703"), Decimal("0.1325")),
        (Decimal("11623"), Decimal("0.18")),
        (Decimal("16472"), Decimal("0.23")),
        (Decimal("21321"), Decimal("0.26")),
        (Decimal("27146"), Decimal("0.3275")),
        (Decimal("39791"), Decimal("0.37")),
        (Decimal("51997"), Decimal("0.435")),
        (Decimal("81199"), Decimal("0.45")),
        (None, Decimal("0.48")),  # No upper limit
    ]

    def calculate_tax(self, income: Decimal, deductions: list) -> TaxResult:
        gross = self._apply_brackets(income)
        total_deductions = sum(d.amount for d in deductions)
        net = max(gross - total_deductions, Decimal("0"))
        rate = (net / income * 100) if income else Decimal("0")
        return TaxResult(
            gross_tax=gross,
            net_tax=net,
            effective_rate=rate,
            breakdown=self._build_breakdown(income),
            optimization_suggestions=self._suggest(income, deductions),
        )

    def _apply_brackets(self, income: Decimal) -> Decimal:
        tax = Decimal("0")
        prev_limit = Decimal("0")
        for limit, rate in self.BRACKETS:
            if limit is None:
                tax += (income - prev_limit) * rate
                break
            if income <= limit:
                tax += (income - prev_limit) * rate
                break
            tax += (limit - prev_limit) * rate
            prev_limit = limit
        return tax

    def get_applicable_brackets(self) -> list[dict]:
        return [{"limit": l, "rate": r} for l, r in self.BRACKETS]

    def get_max_deductions(self) -> DeductionLimits:
        return DeductionLimits(
            saude=Decimal("1000"),
            educacao=Decimal("800"),
            habitacao=Decimal("502"),
            geral=Decimal("250"),
        )

    def _build_breakdown(self, income: Decimal) -> list[dict]:
        # Returns bracket-by-bracket detail
        ...

    def _suggest(self, income: Decimal, deductions: list) -> list[str]:
        # Compares current deductions vs limits, suggests maximization
        ...
```

```python
# apps/fiscal/services/calculator.py

from .strategies.base import TaxStrategy, TaxResult
from decimal import Decimal


class TaxCalculatorService:
    """Context class — delegates to the active TaxStrategy."""

    def __init__(self, strategy: TaxStrategy):
        self._strategy = strategy

    def set_strategy(self, strategy: TaxStrategy) -> None:
        self._strategy = strategy

    def calculate(self, income: Decimal, deductions: list) -> TaxResult:
        return self._strategy.calculate_tax(income, deductions)
```

**When to extend:** New tax regime? Create a new class inheriting `TaxStrategy`. Zero changes to existing code.

---

## 2. Observer Pattern (Behavioral)

**Problem:** When the Diário da República publishes a new fiscal law, multiple subsystems react: send emails, push in-app notifications, update cached tax data. These reactions shouldn't be coupled.

**Solution:** Combine the GoF Observer with Django signals + Celery for async dispatch.

```python
# apps/legislative/services/observers/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime


@dataclass
class LegislativeEvent:
    id: str
    topic: str          # "IRS", "IRC", "IVA", etc.
    title: str
    summary: str
    source_url: str
    published_at: datetime
    affects_user_types: list[str]


class LegislativeObserver(ABC):
    """Abstract observer for legislative updates."""

    @property
    @abstractmethod
    def observer_id(self) -> str:
        ...

    @abstractmethod
    def update(self, event: LegislativeEvent) -> None:
        ...
```

```python
# apps/legislative/services/observers/email_notifier.py

from .base import LegislativeObserver, LegislativeEvent
from apps.legislative.tasks import send_legislative_email


class EmailNotifier(LegislativeObserver):
    """Dispatches email notifications via Celery."""

    @property
    def observer_id(self) -> str:
        return "email_notifier"

    def update(self, event: LegislativeEvent) -> None:
        # Delegate to async Celery task
        send_legislative_email.delay(
            topic=event.topic,
            title=event.title,
            summary=event.summary,
            source_url=event.source_url,
        )
```

```python
# apps/legislative/services/monitor.py

from collections import defaultdict
from .observers.base import LegislativeObserver, LegislativeEvent


class LegislativeMonitor:
    """Subject — manages observers and dispatches legislative events."""

    def __init__(self):
        self._observers: dict[str, set[LegislativeObserver]] = defaultdict(set)

    def subscribe(self, topic: str, observer: LegislativeObserver) -> None:
        self._observers[topic].add(observer)

    def unsubscribe(self, topic: str, observer: LegislativeObserver) -> None:
        self._observers[topic].discard(observer)

    def notify(self, topic: str, event: LegislativeEvent) -> None:
        for observer in self._observers.get(topic, set()):
            observer.update(event)

    def notify_all(self, event: LegislativeEvent) -> None:
        """Notify observers for the event's topic."""
        self.notify(event.topic, event)
```

```python
# apps/legislative/tasks.py

from celery import shared_task


@shared_task
def poll_legislative_updates():
    """Periodic task: polls external APIs and notifies observers."""
    from adapters.factory import ApiClientFactory
    from .services.monitor import LegislativeMonitor
    from .services.observers.email_notifier import EmailNotifier
    from .services.observers.in_app_notifier import InAppNotifier
    from .repositories import LegislativeUpdateRepository

    monitor = LegislativeMonitor()
    monitor.subscribe("IRS", EmailNotifier())
    monitor.subscribe("IRS", InAppNotifier())
    monitor.subscribe("IRC", EmailNotifier())
    monitor.subscribe("IRC", InAppNotifier())

    repo = LegislativeUpdateRepository()

    for source in ["diario_republica", "portal_financas"]:
        client = ApiClientFactory.create(source)
        events = client.get_recent_fiscal_updates()
        for event in events:
            if not repo.exists_by_external_id(event.id):
                repo.create_from_event(event)
                monitor.notify_all(event)


@shared_task
def send_legislative_email(topic: str, title: str, summary: str, source_url: str):
    """Send email notification to subscribed users."""
    from apps.accounts.repositories import UserRepository
    from django.core.mail import send_mass_mail

    repo = UserRepository()
    users = repo.get_users_subscribed_to(topic, channel="EMAIL")
    # Build and send emails...
```

---

## 3. Command Pattern (Behavioral)

**Problem:** Users perform fiscal actions (add deduction, change regime). We need undo capability, audit trails, and the ability to batch actions.

```python
# apps/fiscal/services/commands/base.py

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from uuid import uuid4


@dataclass
class CommandResult:
    success: bool
    message: str
    data: dict | None = None


class FiscalCommand(ABC):
    """Abstract command with execute and undo."""

    def __init__(self):
        self.command_id: str = str(uuid4())
        self.timestamp: datetime = datetime.now()

    @property
    @abstractmethod
    def description(self) -> str:
        ...

    @abstractmethod
    def execute(self) -> CommandResult:
        ...

    @abstractmethod
    def undo(self) -> CommandResult:
        ...
```

```python
# apps/fiscal/services/commands/add_deduction.py

from .base import FiscalCommand, CommandResult
from apps.fiscal.repositories import DeductionRepository


class AddDeductionCommand(FiscalCommand):
    """Command to add a fiscal deduction."""

    def __init__(self, user_id: str, deduction_data: dict, repo: DeductionRepository):
        super().__init__()
        self._user_id = user_id
        self._data = deduction_data
        self._repo = repo
        self._created_id: str | None = None

    @property
    def description(self) -> str:
        return f"Add deduction: {self._data['category']} — €{self._data['amount']}"

    def execute(self) -> CommandResult:
        deduction = self._repo.create(self._user_id, self._data)
        self._created_id = str(deduction.id)
        return CommandResult(success=True, message="Deduction added", data={"id": self._created_id})

    def undo(self) -> CommandResult:
        if self._created_id:
            self._repo.delete(self._created_id)
            return CommandResult(success=True, message="Deduction removed")
        return CommandResult(success=False, message="Nothing to undo")
```

```python
# apps/fiscal/services/commands/invoker.py

from .base import FiscalCommand, CommandResult


class CommandInvoker:
    """Manages command execution history with undo/redo."""

    def __init__(self):
        self._history: list[FiscalCommand] = []
        self._undone: list[FiscalCommand] = []

    def execute(self, command: FiscalCommand) -> CommandResult:
        result = command.execute()
        if result.success:
            self._history.append(command)
            self._undone.clear()
        return result

    def undo(self) -> CommandResult | None:
        if not self._history:
            return None
        command = self._history.pop()
        result = command.undo()
        if result.success:
            self._undone.append(command)
        return result

    def redo(self) -> CommandResult | None:
        if not self._undone:
            return None
        command = self._undone.pop()
        result = command.execute()
        if result.success:
            self._history.append(command)
        return result

    def get_history(self) -> list[FiscalCommand]:
        return list(self._history)
```

---

## 4. Factory Method (Creational)

**Problem:** Individual and Empresa users need different service configurations, different strategies, different dashboards.

```python
# apps/accounts/factories.py

from abc import ABC, abstractmethod
from apps.fiscal.services.strategies.base import TaxStrategy
from apps.fiscal.services.strategies.irs_strategy import IRSStrategy
from apps.fiscal.services.strategies.irc_strategy import IRCStrategy


class UserServiceFactory(ABC):
    """Abstract factory for user-type-specific service creation."""

    @abstractmethod
    def create_tax_strategy(self) -> TaxStrategy:
        ...

    @abstractmethod
    def get_dashboard_template(self) -> str:
        ...

    @abstractmethod
    def get_allowed_deduction_categories(self) -> list[str]:
        ...


class IndividualServiceFactory(UserServiceFactory):
    def create_tax_strategy(self) -> TaxStrategy:
        return IRSStrategy()

    def get_dashboard_template(self) -> str:
        return "fiscal/dashboard_individual.html"

    def get_allowed_deduction_categories(self) -> list[str]:
        return ["SAUDE", "EDUCACAO", "HABITACAO", "LARES", "PENSAO_ALIMENTOS", "DESPESAS_GERAIS"]


class EmpresaServiceFactory(UserServiceFactory):
    def create_tax_strategy(self) -> TaxStrategy:
        return IRCStrategy()

    def get_dashboard_template(self) -> str:
        return "fiscal/dashboard_empresa.html"

    def get_allowed_deduction_categories(self) -> list[str]:
        return ["CUSTOS_PESSOAL", "AMORTIZACOES", "PROVISOES", "ENCARGOS_FINANCEIROS"]


def get_factory(user_type: str) -> UserServiceFactory:
    """Returns the appropriate factory for the user type."""
    factories = {
        "INDIVIDUAL": IndividualServiceFactory,
        "EMPRESA": EmpresaServiceFactory,
    }
    factory_class = factories.get(user_type)
    if not factory_class:
        raise ValueError(f"Unknown user type: {user_type}")
    return factory_class()
```

---

## 5. Adapter Pattern (Structural)

**Problem:** Diário da República and Portal das Finanças have wildly different APIs. Our domain code shouldn't care.

```python
# adapters/base.py

from abc import ABC, abstractmethod
import httpx
import logging

logger = logging.getLogger(__name__)


class BaseApiClient(ABC):
    """Abstract base for external API clients. Uses Template Method."""

    def __init__(self):
        self._client = httpx.Client(timeout=30.0)

    @property
    @abstractmethod
    def base_url(self) -> str:
        ...

    @abstractmethod
    def _authenticate(self) -> dict[str, str]:
        """Returns auth headers."""
        ...

    @abstractmethod
    def _transform_response(self, raw: dict) -> list:
        """Transforms API-specific format → domain objects."""
        ...

    def fetch(self, endpoint: str, params: dict | None = None) -> list:
        """Template Method: shared fetch flow, subclasses customize steps."""
        headers = self._authenticate()
        url = f"{self.base_url}{endpoint}"
        try:
            response = self._client.get(url, params=params, headers=headers)
            response.raise_for_status()
            return self._transform_response(response.json())
        except httpx.HTTPStatusError as e:
            logger.error(f"API error {e.response.status_code}: {url}")
            raise
        except httpx.RequestError as e:
            logger.error(f"Request failed: {url} — {e}")
            raise
```

```python
# adapters/diario_republica.py

from .base import BaseApiClient
from apps.legislative.services.observers.base import LegislativeEvent
from datetime import datetime


class DiarioRepublicaAdapter(BaseApiClient):

    @property
    def base_url(self) -> str:
        return "https://dre.pt/api"

    def _authenticate(self) -> dict[str, str]:
        from django.conf import settings
        return {"Authorization": f"Bearer {settings.DRE_API_KEY}"}

    def _transform_response(self, raw: dict) -> list[LegislativeEvent]:
        return [
            LegislativeEvent(
                id=diploma["id"],
                topic=self._classify_topic(diploma),
                title=diploma["titulo"],
                summary=diploma.get("sumario", ""),
                source_url=diploma["url"],
                published_at=datetime.fromisoformat(diploma["dataPublicacao"]),
                affects_user_types=self._detect_affected_types(diploma),
            )
            for diploma in raw.get("diplomas", [])
        ]

    def get_recent_fiscal_updates(self, since_days: int = 2) -> list[LegislativeEvent]:
        from datetime import timedelta
        since = (datetime.now() - timedelta(days=since_days)).strftime("%Y-%m-%d")
        return self.fetch("/diplomas", params={"tipo": "fiscal", "desde": since})

    def _classify_topic(self, diploma: dict) -> str:
        # Keyword-based classification into IRS, IRC, IVA, etc.
        ...

    def _detect_affected_types(self, diploma: dict) -> list[str]:
        # Determines which user types are affected
        ...
```

---

## 6. Facade Pattern (Structural)

**Problem:** Views shouldn't juggle repositories, strategies, and commands directly.

```python
# apps/fiscal/services/facade.py

from decimal import Decimal
from apps.fiscal.services.calculator import TaxCalculatorService
from apps.fiscal.services.commands.invoker import CommandInvoker
from apps.fiscal.services.commands.add_deduction import AddDeductionCommand
from apps.fiscal.repositories import DeductionRepository, SimulationRepository
from apps.accounts.factories import get_factory


class FiscalFacade:
    """Simplifies complex fiscal operations for views."""

    def __init__(self, user):
        self._user = user
        self._factory = get_factory(user.user_type)
        self._strategy = self._factory.create_tax_strategy()
        self._calculator = TaxCalculatorService(self._strategy)
        self._deduction_repo = DeductionRepository()
        self._simulation_repo = SimulationRepository()
        self._invoker = CommandInvoker()

    def get_dashboard_context(self) -> dict:
        """Returns everything the dashboard template needs."""
        deductions = self._deduction_repo.find_by_user(str(self._user.id))
        income = self._user.profile.gross_income or Decimal("0")
        tax_result = self._calculator.calculate(income, deductions)
        return {
            "tax_result": tax_result,
            "deductions": deductions,
            "suggestions": tax_result.optimization_suggestions,
            "template": self._factory.get_dashboard_template(),
        }

    def add_deduction(self, deduction_data: dict):
        """Adds a deduction via Command pattern."""
        command = AddDeductionCommand(
            str(self._user.id), deduction_data, self._deduction_repo
        )
        return self._invoker.execute(command)

    def undo_last_action(self):
        return self._invoker.undo()
```

---

## 7. Repository Pattern

**Problem:** Scattering `Model.objects.filter(...)` across services creates coupling and makes testing hard.

```python
# apps/fiscal/repositories.py

from abc import ABC, abstractmethod
from django.db.models import QuerySet


class BaseRepository(ABC):
    """Abstract repository — wraps Django ORM."""

    @property
    @abstractmethod
    def model(self):
        ...

    def find_by_id(self, pk: str):
        return self.model.objects.filter(pk=pk).first()

    def find_all(self, **filters) -> QuerySet:
        return self.model.objects.filter(**filters)

    def create(self, **kwargs):
        return self.model.objects.create(**kwargs)

    def delete(self, pk: str) -> None:
        self.model.objects.filter(pk=pk).delete()


class DeductionRepository(BaseRepository):
    @property
    def model(self):
        from .models import Deduction
        return Deduction

    def find_by_user(self, user_id: str, fiscal_year: int | None = None) -> QuerySet:
        qs = self.find_all(user_id=user_id)
        if fiscal_year:
            qs = qs.filter(fiscal_year=fiscal_year)
        return qs.order_by("-date")

    def total_by_category(self, user_id: str, fiscal_year: int) -> dict:
        from django.db.models import Sum
        return (
            self.find_all(user_id=user_id, fiscal_year=fiscal_year)
            .values("category")
            .annotate(total=Sum("amount"))
        )
```

---

## 8. Decorator Pattern (Structural)

Python's native decorators map perfectly to the GoF Decorator.

```python
# apps/core/decorators.py

import functools
import logging
import time
from django.core.cache import cache

logger = logging.getLogger(__name__)


def log_service_call(func):
    """Logs entry, exit, and duration of service methods."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        name = f"{func.__module__}.{func.__qualname__}"
        logger.info(f"[SERVICE] {name} called")
        start = time.perf_counter()
        try:
            result = func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            logger.info(f"[SERVICE] {name} completed in {elapsed:.3f}s")
            return result
        except Exception as e:
            logger.error(f"[SERVICE] {name} failed: {e}")
            raise
    return wrapper


def cache_result(timeout: int = 3600, key_prefix: str = ""):
    """Caches the result of a service method."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            cache_key = f"{key_prefix}:{func.__qualname__}:{hash((args, tuple(sorted(kwargs.items()))))}"
            cached = cache.get(cache_key)
            if cached is not None:
                return cached
            result = func(*args, **kwargs)
            cache.set(cache_key, result, timeout)
            return result
        return wrapper
    return decorator
```

Usage in services:
```python
class TaxCalculatorService:
    @log_service_call
    @cache_result(timeout=3600, key_prefix="tax_calc")
    def calculate(self, income, deductions):
        return self._strategy.calculate_tax(income, deductions)
```

---

## Pattern Decision Guide

When adding new features, ask:

1. **Multiple ways to do the same thing?** → Strategy
2. **Multiple things react to one event?** → Observer (+ Django signals + Celery)
3. **Need undo, audit trail, or queueing?** → Command
4. **Different object families per context?** → Factory
5. **Wrapping an external API?** → Adapter
6. **Complex subsystem needs simple interface?** → Facade
7. **Adding cross-cutting concerns?** → Decorator (Python native)
8. **Data access abstraction?** → Repository
