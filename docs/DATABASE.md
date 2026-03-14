# Database Design — Viriditas

> PostgreSQL + Django ORM. Models are the source of truth. Migrations are versioned.

---

## Entity Relationship Overview

```
┌──────────────┐       ┌─────────────────┐
│    User       │──1:1──│  UserProfile     │
│  (auth base)  │       │  (individual or  │
│               │       │   empresa data)  │
└──────┬───────┘       └─────────────────┘
       │
       │ 1:N
       ▼
┌──────────────┐       ┌──────────────────┐
│  Deduction    │       │  TaxSimulation   │
│  (tracked     │       │  (saved calc     │
│   expenses)   │       │   snapshots)     │
└──────────────┘       └──────────────────┘

┌──────────────────────────────────────────┐
│          LegislativeUpdate               │
│  (pulled from DR / Finanças APIs)        │
└──────────────────────────────────────────┘
       │
       │ M:N via
       ▼
┌──────────────────────────────────┐
│  UserAlertSubscription           │
│  (which users watch which topics)│
└──────────────────────────────────┘

┌──────────────────────────────────┐
│  Subscription (Stripe)           │
│  (premium tier billing)          │
└──────────────────────────────────┘
```

---

## Abstract Base Model

All models inherit from a shared timestamped base:

```python
# apps/core/models.py

import uuid
from django.db import models


class TimeStampedModel(models.Model):
    """Abstract base with UUID pk and timestamps."""

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

---

## Core Models

### User (Custom User Model)

```python
# apps/accounts/models.py

from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin
from django.db import models
from apps.core.models import TimeStampedModel
from .managers import UserManager


class UserType(models.TextChoices):
    INDIVIDUAL = "INDIVIDUAL", "Individual"
    EMPRESA = "EMPRESA", "Empresa"


class User(AbstractBaseUser, PermissionsMixin, TimeStampedModel):
    email = models.EmailField(unique=True, db_index=True)
    user_type = models.CharField(max_length=20, choices=UserType.choices)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = UserManager()

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["user_type"]

    class Meta:
        db_table = "users"
        indexes = [
            models.Index(fields=["user_type"]),
        ]

    def __str__(self):
        return self.email

    @property
    def is_individual(self) -> bool:
        return self.user_type == UserType.INDIVIDUAL

    @property
    def is_empresa(self) -> bool:
        return self.user_type == UserType.EMPRESA
```

### UserProfile

```python
# apps/accounts/models.py (continued)

class MaritalStatus(models.TextChoices):
    SINGLE = "SINGLE", "Solteiro/a"
    MARRIED = "MARRIED", "Casado/a"
    DIVORCED = "DIVORCED", "Divorciado/a"
    WIDOWED = "WIDOWED", "Viúvo/a"
    DOMESTIC_PARTNERSHIP = "DOMESTIC_PARTNERSHIP", "União de facto"


class CompanyType(models.TextChoices):
    ENI = "ENI", "Empresário em Nome Individual"
    LDA = "LDA", "Sociedade por Quotas"
    SA = "SA", "Sociedade Anónima"
    UNIPESSOAL = "UNIPESSOAL", "Sociedade Unipessoal"


class UserProfile(TimeStampedModel):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="profile")
    name = models.CharField(max_length=255)
    nif = models.CharField(max_length=9, unique=True, db_index=True)
    phone = models.CharField(max_length=20, blank=True)
    address = models.TextField(blank=True)
    gross_income = models.DecimalField(max_digits=12, decimal_places=2, null=True, blank=True)

    # Individual-specific
    marital_status = models.CharField(
        max_length=25, choices=MaritalStatus.choices, blank=True
    )
    dependents = models.PositiveIntegerField(null=True, blank=True)

    # Empresa-specific
    company_type = models.CharField(
        max_length=20, choices=CompanyType.choices, blank=True
    )
    cae = models.CharField(max_length=10, blank=True, help_text="Código de Atividade Económica")
    employees = models.PositiveIntegerField(null=True, blank=True)

    class Meta:
        db_table = "user_profiles"

    def __str__(self):
        return f"{self.name} ({self.nif})"
```

### Deduction

```python
# apps/fiscal/models.py

from django.db import models
from apps.core.models import TimeStampedModel


class DeductionCategory(models.TextChoices):
    SAUDE = "SAUDE", "Saúde"
    EDUCACAO = "EDUCACAO", "Educação"
    HABITACAO = "HABITACAO", "Habitação"
    LARES = "LARES", "Lares"
    PENSAO_ALIMENTOS = "PENSAO_ALIMENTOS", "Pensão de Alimentos"
    DESPESAS_GERAIS = "DESPESAS_GERAIS", "Despesas Gerais Familiares"
    IVA_FATURA = "IVA_FATURA", "Exigência de Fatura (IVA)"
    # Empresa categories
    CUSTOS_PESSOAL = "CUSTOS_PESSOAL", "Custos com Pessoal"
    AMORTIZACOES = "AMORTIZACOES", "Amortizações"
    PROVISOES = "PROVISOES", "Provisões"
    ENCARGOS_FINANCEIROS = "ENCARGOS_FINANCEIROS", "Encargos Financeiros"


class DeductionStatus(models.TextChoices):
    PENDING = "PENDING", "Pendente"
    VERIFIED = "VERIFIED", "Verificado"
    REJECTED = "REJECTED", "Rejeitado"


class Deduction(TimeStampedModel):
    user = models.ForeignKey(
        "accounts.User", on_delete=models.CASCADE, related_name="deductions"
    )
    category = models.CharField(max_length=30, choices=DeductionCategory.choices)
    description = models.CharField(max_length=500)
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    date = models.DateField()
    document_ref = models.CharField(max_length=255, blank=True)
    status = models.CharField(
        max_length=20, choices=DeductionStatus.choices, default=DeductionStatus.PENDING
    )
    fiscal_year = models.PositiveIntegerField()

    class Meta:
        db_table = "deductions"
        ordering = ["-date"]
        indexes = [
            models.Index(fields=["user", "fiscal_year"]),
            models.Index(fields=["category"]),
        ]

    def __str__(self):
        return f"{self.category} — €{self.amount} ({self.date})"
```

### TaxSimulation

```python
# apps/fiscal/models.py (continued)

class TaxSimulation(TimeStampedModel):
    user = models.ForeignKey(
        "accounts.User", on_delete=models.CASCADE, related_name="simulations"
    )
    fiscal_year = models.PositiveIntegerField()
    regime = models.CharField(max_length=50)
    gross_income = models.DecimalField(max_digits=12, decimal_places=2)
    total_deductions = models.DecimalField(max_digits=12, decimal_places=2)
    tax_due = models.DecimalField(max_digits=12, decimal_places=2)
    effective_rate = models.DecimalField(max_digits=5, decimal_places=2)
    breakdown = models.JSONField(default=dict)

    class Meta:
        db_table = "tax_simulations"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["user", "fiscal_year"]),
        ]
```

### LegislativeUpdate

```python
# apps/legislative/models.py

from django.db import models
from django.contrib.postgres.fields import ArrayField
from apps.core.models import TimeStampedModel


class LegislativeSource(models.TextChoices):
    DIARIO_REPUBLICA = "DIARIO_REPUBLICA", "Diário da República"
    PORTAL_FINANCAS = "PORTAL_FINANCAS", "Portal das Finanças"


class FiscalTopic(models.TextChoices):
    IRS = "IRS", "IRS"
    IRC = "IRC", "IRC"
    IVA = "IVA", "IVA"
    IMI = "IMI", "IMI"
    IMT = "IMT", "IMT"
    SELO = "SELO", "Imposto do Selo"
    INCENTIVOS = "INCENTIVOS", "Benefícios Fiscais"
    SEGURANCA_SOCIAL = "SEGURANCA_SOCIAL", "Segurança Social"
    OUTRO = "OUTRO", "Outro"


class AlertChannel(models.TextChoices):
    EMAIL = "EMAIL", "Email"
    IN_APP = "IN_APP", "Notificação"
    BOTH = "BOTH", "Ambos"


class LegislativeUpdate(TimeStampedModel):
    external_id = models.CharField(max_length=255, unique=True)
    source = models.CharField(max_length=30, choices=LegislativeSource.choices)
    topic = models.CharField(max_length=30, choices=FiscalTopic.choices)
    title = models.CharField(max_length=500)
    summary = models.TextField()
    source_url = models.URLField()
    published_at = models.DateTimeField()
    affects_user_types = ArrayField(
        models.CharField(max_length=20), default=list
    )
    fetched_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "legislative_updates"
        ordering = ["-published_at"]
        indexes = [
            models.Index(fields=["topic"]),
            models.Index(fields=["published_at"]),
            models.Index(fields=["source", "external_id"]),
        ]

    def __str__(self):
        return f"[{self.topic}] {self.title}"


class UserAlertSubscription(TimeStampedModel):
    user = models.ForeignKey(
        "accounts.User", on_delete=models.CASCADE, related_name="alert_subscriptions"
    )
    topic = models.CharField(max_length=30, choices=FiscalTopic.choices)
    channel = models.CharField(
        max_length=10, choices=AlertChannel.choices, default=AlertChannel.IN_APP
    )
    is_active = models.BooleanField(default=True)

    class Meta:
        db_table = "user_alert_subscriptions"
        unique_together = ["user", "topic"]
```

### Subscription (Stripe)

```python
# apps/payments/models.py

from django.db import models
from apps.core.models import TimeStampedModel


class SubscriptionStatus(models.TextChoices):
    ACTIVE = "ACTIVE", "Ativo"
    PAST_DUE = "PAST_DUE", "Pagamento em atraso"
    CANCELED = "CANCELED", "Cancelado"
    TRIALING = "TRIALING", "Período de teste"


class Subscription(TimeStampedModel):
    user = models.OneToOneField(
        "accounts.User", on_delete=models.CASCADE, related_name="subscription"
    )
    stripe_customer_id = models.CharField(max_length=255, unique=True)
    stripe_subscription_id = models.CharField(max_length=255, unique=True, blank=True)
    status = models.CharField(
        max_length=20, choices=SubscriptionStatus.choices, default=SubscriptionStatus.TRIALING
    )
    current_period_end = models.DateTimeField(null=True, blank=True)
    plan_name = models.CharField(max_length=50, default="free")

    class Meta:
        db_table = "subscriptions"

    @property
    def is_premium(self) -> bool:
        return self.status == SubscriptionStatus.ACTIVE and self.plan_name != "free"
```

---

## Migration Strategy

```bash
# Development — create and apply migrations
python manage.py makemigrations
python manage.py migrate

# Production — apply only (never makemigrations in prod)
python manage.py migrate --no-input

# Seed data — custom management command
python manage.py seed_fiscal_data
```

---

## Important: Custom User Model

Django requires `AUTH_USER_MODEL` to be set **before the first migration**:

```python
# viriditas/settings/base.py
AUTH_USER_MODEL = "accounts.User"
```

---

## Future Considerations

- **Audit log table** — Persist `CommandInvoker` history for compliance
- **Document storage** — Uploaded invoices/receipts (Django Storages + S3)
- **Tax bracket versioning** — Brackets change yearly; historical data needed
- **Multi-tenancy** — For accounting firms managing multiple clients
- **Read replicas** — If query load justifies splitting reads/writes
