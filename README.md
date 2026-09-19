# Cleaning Multi Service JC — repository audit & productionization handoff

**Prepared 19 September 2026.** For `ferjm-cell/JCcleaningService` (branch `main`).

---

## 0. Audit result, in one line

**The repository is empty.** At commit `64115f33fd77` it contains exactly one file — a 19-byte `README.md` holding the repo title. There is no frontend, backend, database, API, auth, state management, routing, package manifest, build system, deployment config, test suite, environment configuration or dependency list to audit.

Every audit item in the brief therefore resolves to the same answer, and the honest report is short:

| Audit target | Found in repo |
| --- | --- |
| Project structure, frontend, backend, database, API, auth | none |
| State management, components, routing, business logic | none |
| Existing data, localStorage, mock data, hard-coded data | none |
| Integrations, env config, dependencies, build, deployment | none |
| Tests, security posture, mobile responsiveness | none |

**The application does not live in this repository.** It lives in the design workspace, as `Cleaning MSJC.dc.html` — a single self-contained HTML file with its own state tree and browser persistence. That file is the real subject of the audit, and a full assessment of it was produced separately (*Technical Assessment & productionization plan*). Its findings are summarized in §1 and are the basis for the roadmap in §3.

**What this means practically:** there is nothing here to preserve, evolve, or avoid breaking. Phase 1 is not a refactor — it is standing up a new codebase in an empty repo, and porting a known, working interface onto it. That is a *lower*-risk starting point than the brief assumes.

---

## 1. Real vs prototype — the actual application

Assessed against `Cleaning MSJC.dc.html` (~3,700 lines) as of this handoff.

### Already implemented (works today)
- Twelve sections: Dashboard, Clients, Estimates & prices, Invoices & payments, Schedule / jobs, Employees, Financials, Legal & licenses, Flyers & marketing, Checklists, Guides & scripts, Connections, plus Settings.
- Full create / edit / delete on every record type, driven by a single declarative schema table.
- All financial figures derived from the records at render time — revenue, receivables, aging buckets, pipeline, P&L, cost per job, payroll. No stored or invented totals.
- Customer 360: one client record joining jobs, estimates, invoices, notes and a merged activity timeline.
- Estimate builder and a print-accurate letter-size estimate sheet.
- Bilingual EN/ES, light/dark, dense/roomy density.
- Apple-first responsive system: iPhone (bottom tab bar, sheets, card-form tables, safe areas), iPad (condensed sidebar, master/detail split), desktop.
- JSON export / import of the entire book; stable record ids; money-field normalization.

### Partially implemented (needs a backend)
- **Persistence** — browser local storage under key `cmsjc.book.v1`. Single device, single browser profile. Export/import is the only bridge between devices.
- **Connections** — the Integration Hub screen exists with per-provider status, but status is typed, not observed.

### Mocked / simulated (appears to work, is not connected)
- Gmail send, Drive filing, DocuSign routing, Calendar events. Each confirms in the interface; nothing leaves the machine.
- Payment status is hand-entered; no provider, no reconciliation.

### Missing entirely
Authentication and sessions · roles/permissions · server-side data · audit log · tasks · notification delivery · leads/CRM pipeline · recurring service generation · proof-of-work capture · documents/asset/vehicle registers · analytics beyond current derived figures · AI assistant · webhooks · automation runtime · backups · tests.

### Risky (address first)
1. **Data loss is one click away** — clearing browser data destroys the book. Export exists but is manual. Outranks every feature below.
2. **No device continuity** — iPhone, iPad and desktop each hold a separate, silently diverging copy.
3. **No audit trail** — nothing records who changed what, when, or the prior value.
4. **Integrations that confirm without acting** — an estimate marked "sent" that was never sent is worse than no send button. Keep these labelled unconnected until credentials are real.
5. **No server-side validation** — a typo in a rate propagates into revenue, margin and payroll.

---

## 2. What is in this bundle

```
design_handoff_cleaning_msjc/
├── README.md              ← this file: audit + roadmap
├── ARCHITECTURE.md        ← target architecture, schema, API, integrations, mobile
├── .env.example           ← every credential the system will need, no values
└── design-reference/
    ├── Cleaning MSJC.dc.html   the working application
    ├── support.js              its runtime
    ├── app.webmanifest         PWA manifest
    ├── assets/                 logo
    └── _ds/…                   Modernist design system (tokens + bundle)
```

**Fidelity: high.** `design-reference/` is a working, high-fidelity application — final colors, typography, spacing, interactions and copy, in both languages. Open the HTML file directly in a browser; it runs with no build step.

**It is a design reference, not production code to lift.** The task is to recreate these screens in the target stack (see ARCHITECTURE.md) using that stack's patterns, while matching the reference pixel for pixel. Read the reference for exact values rather than approximating: the Modernist tokens in `_ds/…/styles.css` carry the palette, type scale and spacing; the inline styles in the HTML carry per-screen layout.

---

## 3. Implementation roadmap

Sequenced so that each phase is shippable and nothing depends on a credential that has not arrived.

### Phase 1 — Foundation *(no external blockers)*
Stand up the repo and the data layer.

- **Files:** new — `package.json`, `tsconfig.json`, `/api` (server), `/web` (client), `/db/migrations`, `docker-compose.yml`, `.env.example` (from this bundle), CI workflow.
- **Database:** initial migration creating the core schema in ARCHITECTURE.md §3.
- **API:** health check, session endpoints, `/customers` as the first vertical slice.
- **Auth:** single owner account, email + password with a strong hash, httpOnly session cookie. Roles table seeded but unused.
- **Dependencies:** Postgres, a typed ORM, a server framework, a session/auth library, Zod (or equivalent) for validation.
- **Risks:** low — empty repo, nothing to break.
- **Migration:** export the JSON book from the running app (sidebar → Backup → Download JSON) and write a one-shot importer. Verify record counts and revenue totals against the live app before cutting over.
- **Testing:** migration round-trip (export → import → totals match), auth happy/unhappy paths, `/customers` CRUD.

### Phase 2 — Core business on real data
Port the existing UI onto the API, section by section, in this order: Clients → Estimates → Invoices → Schedule → Financials → Employees → the rest.

- **Components:** each section becomes a route; the schema table in the reference maps directly onto form definitions.
- **API:** full CRUD per entity, cursor pagination, filter/sort params, consistent error envelope.
- **DB:** add tasks, notifications, activity/audit tables.
- **Risks:** behavioral drift from the reference. Mitigate by porting one section at a time and diffing against the reference screen.
- **Migration:** keep local storage as an offline read cache; the server is authoritative.
- **Testing:** per-entity CRUD, permission checks, derived-figure parity with the reference.

### Phase 3 — Integrations *(blocked on credentials — see §4)*
Gmail, Drive, Calendar, Docs, DocuSign, Stripe. One provider at a time, each behind a feature flag, each degrading to a visible "not connected" state rather than an error. Store only external ids; never copy provider data. Webhook endpoints with signature verification, idempotency keys and a delivery log.

### Phase 4 — Automation
Domain event bus (event names in ARCHITECTURE.md §5), rule storage, a worker with retries and a run log. No hard-coded workflows.

### Phase 5 — Mobile
The responsive system is already built and ported in Phase 2. This phase adds what needs native APIs: camera capture, file picker, Add-to-Home-Screen polish, offline queue for writes made without a signal, push notifications.

### Phase 6 — Intelligence
Analytics views over the accumulated data; AI assistant reading authorized business data, drafting but never sending without confirmation.

### Phase 7 — Future users
Activate roles and permissions; add employee and customer portals on the same core data. No rebuild required if Phase 1 seeds the role tables.

---

## 4. Blockers requiring you (the owner)

These cannot be inferred or safely faked. Everything else proceeds without you.

1. **Hosting decision** — where the API and database run, and who pays for it.
2. **Google Cloud project** — OAuth client id/secret, and consent-screen verification for Gmail/Drive/Calendar/Docs scopes. Verification can take days; start it early.
3. **DocuSign developer account** — integration key, user id, RSA keypair for JWT grant.
4. **Stripe account** — publishable/secret keys and a webhook signing secret.
5. **Domain name** for the app and for OAuth redirect URIs.

Nothing in Phases 1, 2, 4 or 6 is blocked by any of these.

---

## 5. Git conventions for this work

- Branch per phase: `phase-1-foundation`, `phase-2-core`, and so on. Never commit directly to `main`.
- One logical change per commit; migrations in their own commit, reversible where practical.
- `.env` is git-ignored from the first commit; update `.env.example` instead.
- Tag the end of each phase so any phase can be rolled back to.
