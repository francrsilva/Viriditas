# Roadmap — Viriditas

---

## Phase 1 — Foundation (Current)

> Goal: Core infrastructure, auth, homepage, basic navigation.

- [x] Project structure and documentation
- [ ] Django project scaffold (settings split, apps, templates)
- [ ] Custom User model + migrations
- [ ] Docker Compose (PostgreSQL + Redis)
- [ ] Homepage with Individual / Empresa selection + Cadastre-se
- [ ] Auth flow (register, login, logout with Django auth)
- [ ] Dashboard shell (sidebar, topbar, base templates)
- [ ] Tailwind CSS setup + reusable template components
- [ ] Stimulus.js setup + first controllers

## Phase 2 — Fiscal Core

> Goal: Tax calculation engine and deduction tracking.

- [ ] Strategy pattern: IRS and IRC calculation engines
- [ ] Deduction CRUD (Command pattern with undo)
- [ ] DeductionRepository + forms
- [ ] Tax simulation: input income + deductions → result
- [ ] Save/compare simulations
- [ ] Individual dashboard: IRS overview, deduction tracker
- [ ] Empresa dashboard: IRC overview, benefit tracker
- [ ] Stimulus controllers for interactive forms

## Phase 3 — Legislative Intelligence

> Goal: Real-time legislative monitoring and user alerts.

- [ ] Celery + Redis setup (worker + beat)
- [ ] Diário da República adapter + polling task
- [ ] Portal das Finanças adapter + RSS parser task
- [ ] Observer pattern: notification system
- [ ] User alert subscription management
- [ ] Legislative feed in dashboard
- [ ] Email notification via Celery task

## Phase 4 — Payments & Premium

> Goal: Stripe billing for premium features.

- [ ] Stripe integration (Customer, Subscription, Webhook)
- [ ] Pricing page
- [ ] Checkout flow
- [ ] Customer portal (manage subscription)
- [ ] Premium feature gating (middleware or decorator)
- [ ] Webhook processing via Celery

## Phase 5 — Optimization Engine

> Goal: Proactive fiscal optimization suggestions.

- [ ] Optimization rules engine (underutilized deductions)
- [ ] "What-if" scenario builder (change regime, add deductions)
- [ ] Comparative analysis (current vs optimal setup)
- [ ] Deadline awareness (key fiscal dates, submission windows)

## Phase 6 — Polish & Scale

> Goal: Production readiness.

- [ ] Gunicorn + Nginx production setup
- [ ] Performance (Redis caching, query optimization, N+1 fixes)
- [ ] Comprehensive test suite (pytest + Playwright E2E)
- [ ] Accessibility audit (WCAG 2.1 AA)
- [ ] Security audit (CSRF, XSS, rate limiting, input validation)
- [ ] Sentry error tracking
- [ ] CI/CD pipeline
- [ ] Documentation for end users

## Future Possibilities

- Mobile-responsive progressive web app
- Accountant multi-tenant mode
- AI-powered document analysis (upload invoice → auto-categorize)
- Integration with e-fatura data
- Social security optimization module
- Full i18n (PT + EN)
