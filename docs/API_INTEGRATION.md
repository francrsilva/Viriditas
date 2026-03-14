# API Integration — External Data Sources

> Viriditas pulls legislative and fiscal data from two primary Portuguese government sources.
> Polling is handled by **Celery Beat** periodic tasks. API clients live in `/adapters/` (not a Django app).

---

## 1. Diário da República (DRE)

The official journal of the Portuguese Republic. All laws, decrees, and regulations are published here.

### Base Information

| Detail | Value |
|--------|-------|
| Website | https://dre.pt |
| API Docs | https://dre.pt/api (limited public docs) |
| Format | JSON / XML |
| Auth | API key (request via dre.pt) |
| Rate Limit | TBD — treat conservatively (1 req/sec) |

### Endpoints of Interest

```
GET /api/diplomas
  ?tipo=lei|decreto-lei|portaria|despacho
  &ministerio=financas
  &dataInicio=YYYY-MM-DD
  &dataFim=YYYY-MM-DD
  &pagina=1
  &porPagina=25
```

### Data We Extract

- New tax laws and amendments
- Portarias with updated tax brackets or thresholds
- Despachos affecting deduction rules
- Budget law (Orçamento de Estado) provisions

---

## 2. Portal das Finanças (AT — Autoridade Tributária)

The tax authority's portal. Contains tax guides, official interpretations, and taxpayer tools.

### Base Information

| Detail | Value |
|--------|-------|
| Website | https://www.portaldasfinancas.gov.pt |
| API | Scraping + RSS (limited official API) |
| Format | HTML / XML / RSS |
| Auth | Varies — some public, some require NIF + password |

### Data Sources

1. **Public RSS feeds** — News and circulars
2. **Informações Vinculativas** — Official binding interpretations
3. **Ofícios Circulados** — Circular letters with practical guidance
4. **Tax bracket tables** — Updated annually post-OE

### Scraping Ethics

- Respect `robots.txt` directives
- Minimum 2 seconds between requests
- Identify user-agent: `Viriditas/1.0 (fiscal-optimization)`
- Cache responses — don't re-scrape unchanged content
- If an official API becomes available, migrate immediately

---

## Adapter Architecture

All API clients live in `/adapters/` — **not a Django app**, just pure Python with `httpx`.

```
adapters/
├── base.py                    # BaseApiClient ABC (Template Method)
├── diario_republica.py        # DiarioRepublicaAdapter
├── portal_financas.py         # PortalFinancasAdapter
└── factory.py                 # ApiClientFactory
```

```python
# adapters/factory.py

from .diario_republica import DiarioRepublicaAdapter
from .portal_financas import PortalFinancasAdapter


class ApiClientFactory:
    """Factory for external API clients."""

    _clients = {
        "diario_republica": DiarioRepublicaAdapter,
        "portal_financas": PortalFinancasAdapter,
    }

    @classmethod
    def create(cls, source: str):
        client_class = cls._clients.get(source)
        if not client_class:
            raise ValueError(f"Unknown API source: {source}")
        return client_class()
```

---

## Celery Integration

Polling is fully async via Celery Beat. Tasks live in `apps/legislative/tasks.py`.

### Periodic Schedule

```python
# viriditas/celery.py

from celery.schedules import crontab

app.conf.beat_schedule = {
    "poll-legislative-updates": {
        "task": "apps.legislative.tasks.poll_legislative_updates",
        "schedule": crontab(hour="*/6"),  # Every 6 hours
    },
}
```

### Task Flow

```
Celery Beat (scheduler)
    │
    ▼
poll_legislative_updates (task)
    │
    ├── ApiClientFactory.create("diario_republica")
    │       └── adapter.get_recent_fiscal_updates()
    │               └── Returns list[LegislativeEvent]
    │
    ├── ApiClientFactory.create("portal_financas")
    │       └── adapter.get_recent_fiscal_updates()
    │               └── Returns list[LegislativeEvent]
    │
    ├── LegislativeUpdateRepository.create_from_event()
    │       └── Deduplicates by external_id, saves to DB
    │
    └── LegislativeMonitor.notify_all(event)
            ├── EmailNotifier → send_legislative_email.delay()
            └── InAppNotifier → create notification in DB
```

---

## Error Handling & Resilience

| Scenario | Strategy |
|----------|----------|
| API timeout | `httpx` timeout + Celery retry (3x, exponential backoff) |
| API down | Serve cached data, log warning, retry next cycle |
| Rate limited | Respect `Retry-After`, queue remaining requests |
| Malformed response | Log error, skip entry, continue processing batch |
| Auth expired | Re-authenticate, retry request |

```python
# In Celery tasks — automatic retry
@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def poll_legislative_updates(self):
    try:
        # ... polling logic
    except Exception as exc:
        self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

---

## Data Freshness

| Data Type | Max Staleness | Update Trigger |
|-----------|--------------|----------------|
| Legislative updates | 12 hours | Celery Beat every 6h |
| Tax brackets | 24 hours | Manual command post-OE |
| Binding interpretations | 24 hours | Celery Beat every 12h |
| Circulars | 12 hours | Celery Beat every 6h |

---

## Configuration

All API settings in environment variables:

```python
# viriditas/settings/base.py

DRE_API_KEY = env("DRE_API_KEY", default="")
DRE_API_BASE_URL = env("DRE_API_BASE_URL", default="https://dre.pt/api")
PORTAL_FINANCAS_BASE_URL = env(
    "PORTAL_FINANCAS_BASE_URL",
    default="https://www.portaldasfinancas.gov.pt",
)
```

---

## Future Sources

- **European Tax Observatory** — EU-level fiscal research
- **INE (Instituto Nacional de Estatística)** — Economic indicators
- **Segurança Social** — Social security contribution tables
