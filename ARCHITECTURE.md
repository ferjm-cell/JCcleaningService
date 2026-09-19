# Target architecture — Cleaning Multi Service JC

Companion to README.md. Concrete enough to implement from; opinionated where the brief left the decision open.

---

## 1. Shape

```
iPhone · iPad · Desktop — one client, three layouts
                │
        HTTPS · owner session cookie
                │
        ┌───────┴────────┐
        │  API (typed)   │  validate → authorize → act → audit → emit
        └───────┬────────┘
                │
   ┌────────────┼─────────────┐
   │            │             │
Postgres    Event bus    Integration workers
(truth)     (domain      (Gmail · Drive · Calendar
             events)      Docs · DocuSign · Stripe)
                │
        Automation engine → retry queue → run log
```

**Stack recommendation.** TypeScript end to end. React for the client (the reference is React-shaped already — a single component tree with derived values, which ports almost line for line). A Node server — Fastify or Nest, either is fine; Nest if you expect other developers, Fastify if speed matters more than ceremony. Postgres for data. Prisma or Drizzle for the ORM and migrations. Zod for validation shared between client and server. Deploy the API and database on one managed platform (Fly, Railway, Render) — a single owner does not need Kubernetes.

**One codebase, one business logic.** iPhone, iPad and desktop are layouts of the same client against the same API. No device-specific endpoints, no duplicated rules.

---

## 2. Source of truth

| Data | Authoritative store | The database keeps |
| --- | --- | --- |
| Customers, jobs, estimates, invoices, expenses, tasks | Application database | everything |
| Email | Gmail | thread id, message id, direction, timestamp |
| Documents, photos | Google Drive | file id, folder id, name, category |
| Calendar events | Google Calendar | event id, job id |
| Signatures | DocuSign | envelope id, status, completed-at |
| Card / ACH payments | Stripe | payment intent id, reconciled amount |

Never copy provider content into the database. Reference by id; fetch on demand; cache only what a list view must render.

---

## 3. Data model

Core tables. Types are Postgres. Every table carries `id uuid primary key default gen_random_uuid()`, `created_at timestamptz not null default now()`, `updated_at timestamptz not null default now()`, and — once multi-user arrives — `company_id uuid not null`.

```sql
-- identity (seeded, single row, until Phase 7)
users(email citext unique, password_hash text, name text, active bool)
roles(key text unique)                       -- owner|admin|manager|staff|accountant|employee|customer
permissions(key text unique)                 -- e.g. 'invoice.write'
user_roles(user_id, role_id)
role_permissions(role_id, permission_id)

-- sales
leads(name, company, contact, phone, email, address, service_requested,
      source text, estimated_value numeric(12,2), stage text,
      last_contact_at timestamptz, next_follow_up_at timestamptz,
      owner_id uuid, notes text, converted_customer_id uuid)
customers(name, kind text, status text, address, contact, phone, email,
          service text, frequency text, rate numeric(12,2), notes text)
estimates(customer_id, lead_id, number text unique, scope text, status text,
          subtotal numeric, tax numeric, total numeric, sent_at, viewed_at,
          accepted_at, expires_at, version int, drive_file_id text)
estimate_items(estimate_id, label, qty numeric, unit_price numeric, total numeric)

-- operations
recurring_services(customer_id, service, frequency text, rrule text,
                   rate numeric, status text)   -- active|paused|cancelled
jobs(customer_id, recurring_service_id, scheduled_for timestamptz,
     duration_minutes int, address, service, price numeric, status text,
     instructions text, notes text, calendar_event_id text)
job_assignments(job_id, employee_id)
job_events(job_id, kind text, at timestamptz, user_id, payload jsonb)  -- clock in/out, checklist, photos
employees(name, role, pay_rate numeric, status text)
employee_documents(employee_id, kind, expires_at, drive_file_id)

-- money
invoices(customer_id, job_id, number text unique, issued_on date, due_on date,
         subtotal numeric, tax numeric, discount numeric, total numeric,
         amount_paid numeric, status text)
invoice_items(invoice_id, label, qty, unit_price, total)
payments(invoice_id, amount numeric, method text, received_on date,
         provider text, provider_ref text, status text)
expenses(category text, vendor text, amount numeric, incurred_on date,
         job_id, notes text, receipt_file_id text)

-- shared
documents(entity_type text, entity_id uuid, category text, name text,
          drive_file_id text, docusign_envelope_id text, expires_at date, version int)
tasks(title, status text, priority text, due_on date, assignee_id,
      customer_id, job_id, invoice_id, estimate_id, notes text)
notifications(kind text, severity text, title, body, entity_type, entity_id,
              read_at timestamptz, resolved_at timestamptz)
activity(entity_type, entity_id, kind text, at timestamptz, user_id, summary text, payload jsonb)
audit_log(user_id, action text, entity_type, entity_id,
          before jsonb, after jsonb, source text, at timestamptz)

-- assets & marketing
assets(name, category, purchased_on, cost numeric, condition text,
       location text, assigned_to uuid, warranty_until date, replace_on date)
vehicles(name, vin, plate, mileage int, driver_id, insured_until date, registered_until date)
marketing_campaigns(name, channel text, spend numeric, started_on, ended_on)

-- plumbing
integrations(provider text unique, status text, account text,
             access_token_enc bytea, refresh_token_enc bytea,
             expires_at timestamptz, last_sync_at timestamptz, last_error text)
webhook_deliveries(provider, external_id text unique, received_at, payload jsonb, processed_at, error text)
automations(name, event text, conditions jsonb, actions jsonb, enabled bool)
automation_runs(automation_id, event_id, status text, started_at, finished_at, error text)
```

**Migrating the current book.** Export JSON from the app (Backup → Download JSON). Its arrays map: `clients → customers`, `estimates → estimates`, `invoices → invoices`, `jobs → jobs`, `crew → employees`, `expenses → expenses`, `legal → documents` (category `legal`), `campaigns → marketing_campaigns`, `notes → activity` (kind `note`), `prices` → a price-book table or seeded `estimate_items` presets, `settings → company settings`. Records already carry a stable `_id` — keep it as `legacy_id` so the import is re-runnable and idempotent.

---

## 4. API

REST, JSON, versioned under `/api/v1`. Session cookie auth; CSRF token on mutations.

```
GET    /customers?query=&status=&cursor=&limit=
POST   /customers
GET    /customers/:id            → includes counts + recent activity (the Customer 360 payload)
PATCH  /customers/:id
DELETE /customers/:id
```

Same shape for `/leads`, `/estimates`, `/jobs`, `/invoices`, `/payments`, `/expenses`, `/employees`, `/documents`, `/tasks`, `/notifications`, `/assets`, `/vehicles`.

```
POST /estimates/:id/send                 → Gmail + status transition
POST /estimates/:id/accept               → emits estimate.accepted
POST /invoices/:id/payments
GET  /search?q=                          → cross-entity universal search
GET  /dashboard                          → one payload for the command centre
GET  /integrations
POST /integrations/:provider/connect     → returns OAuth redirect URL
POST /integrations/:provider/disconnect
POST /integrations/:provider/test
POST /webhooks/{google|docusign|stripe}
```

Conventions: every response `{ data, meta? }`; every error `{ error: { code, message, fields? } }` with a proper status; cursor pagination (never offset); validation at the edge with a shared schema; every mutation writes `audit_log` and emits a domain event; never return tokens, hashes or provider secrets.

---

## 5. Domain events

`customer.created` · `customer.updated` · `lead.created` · `lead.converted` · `estimate.created` · `estimate.sent` · `estimate.viewed` · `estimate.accepted` · `estimate.declined` · `job.created` · `job.scheduled` · `job.completed` · `invoice.created` · `invoice.sent` · `invoice.paid` · `invoice.overdue` · `payment.received` · `payment.failed` · `contract.created` · `signature.completed` · `document.uploaded` · `calendar.event.created` · `integration.connected` · `integration.failed`

Events are persisted before handlers run, so a failed handler is retried rather than lost. Automations are rows — `when event, if conditions, do actions` — not code branches.

---

## 6. Integrations

Uniform contract per provider: `connect() · disconnect() · test() · status()`, tokens encrypted at rest with a key from the environment, refresh handled centrally, and a circuit breaker that flips the provider to `REQUIRES_AUTHORIZATION` rather than throwing into the request path.

Sync states: `SYNCED · PENDING · SYNCING · FAILED · REQUIRES_AUTHORIZATION`.

**A disconnected provider must never break the core app.** A job saves whether or not Calendar accepted it; the row simply carries `calendar_sync: FAILED` and a retry.

Drive folder convention, created lazily and recorded by id so it is never duplicated:

```
Cleaning Multi Service JC/
├── Customers/<customer>/{Contracts,Estimates,Invoices,Job Photos,Documents}/
├── Employees/  ├── Legal/  ├── Finance/  ├── Marketing/  └── Operations/
```

---

## 7. Security

Owner-only today, multi-user-ready underneath. Argon2id password hashing; httpOnly + Secure + SameSite=Lax session cookies; CSRF tokens on mutations; rate limiting on auth and webhooks; every input validated server-side; every mutation audited; secrets from the environment or a managed secret store, never in the repo or the client; HTTPS everywhere; nightly database backups with a restore drill before go-live, kept independently of Google Drive.

---

## 8. Mobile / iPad architecture

The reference already implements the layout system; port it rather than redesigning it.

- **Breakpoints:** ≤744px iPhone · 745–1180px iPad · ≥1181px desktop. Not "desktop, then mobile" — each tier is authored.
- **iPhone:** bottom tab bar (Today · Schedule · Clients · Invoices · More), floating primary action, modals as bottom sheets, data tables as tappable cards, 44px minimum targets, 16px inputs, `env(safe-area-inset-*)` respected top and bottom, overlays suppress the tab bar.
- **iPad:** condensed sidebar, master/detail split from 1000px, roomier rows, 40px controls.
- **PWA:** `app.webmanifest` + `viewport-fit=cover` + apple-touch-icon are in the reference. Add a service worker for static caching and an offline write queue in Phase 5.
- **Native APIs:** camera and file capture via `<input type="file" accept="image/*" capture>`; directions via `maps.apple.com` links (already wired in Customer 360); push via the Web Push API once the backend exists.
