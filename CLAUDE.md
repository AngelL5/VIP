# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Neelitech Vendor Invoice Portal — a Power Apps **code app** (React SPA hosted inside Power Pages/Dataverse) for vendors to submit and track invoices against purchase orders, and for Neelitech staff to review, query, and manage them.

**Branding: always "Neelitech" — never "LTTS" or "L&T Technology Services".**

The app currently runs entirely on **mock/stub data** (see Data layer below); no live Dataverse or Power Automate calls are wired up yet.

Reference docs live in `doc/`: `USER_STORIES.md` (feature specs by story ID, e.g. `US-5.1`), `SPRINT_PLAN.md`, `ARCHITECTURE_REFERENCE.md` (target file tree + route map), `AGENT_FILE_OWNERSHIP.md` (which files belong to which feature area), `MANUAL_TESTING_GUIDE.md`. Page/component comments and commit messages often reference these story IDs — check `USER_STORIES.md` for the full spec behind one before changing behavior.

## Commands

```bash
npm run dev       # start Vite dev server
npm run build     # tsc -b && vite build — type-check then bundle to dist/
npm run lint       # eslint .
npm run preview   # preview the production build locally
```

There is no automated test suite (no test runner configured) — verification is manual, tracked in `doc/MANUAL_TESTING_GUIDE.md`. `npm run build` is the closest thing to CI since it runs the full TypeScript check before bundling.

Deploy is via PAC CLI: `pac pages upload-code-site --rootPath .` (after `npm run build`). Vite is configured with `@microsoft/power-apps-vite` to produce a single inlined bundle (`inlineDynamicImports: true`) suitable for Power Pages hosting.

## Architecture

### Routing & basename

`src/router.tsx` builds a single `createBrowserRouter` tree. **Do not remove the `BASENAME` normalization block at the top of the file** — it derives the router basename from `location.href` and strips a trailing `/index.html`, which is required for the app to route correctly once hosted inside a Power Pages web page. All page components are `React.lazy`-loaded and wrapped in a shared `SuspenseWrapper`.

Route nesting order: `ProtectedRoute` (auth check) → `Layout` (`app-shell`) → `RoleGuard` (role check) → page. Route path strings are centralized in `src/constants/routes.ts` (`ROUTES`) — use that instead of hardcoding paths, and keep it in sync when adding a route to `router.tsx`.

### Auth & roles

Two roles only: `"vendor"` and `"neelitech"` (`src/types/auth.ts`). Auth state is a Zustand store (`src/stores/auth-store.ts`) persisted to `sessionStorage`, with a hardcoded `MOCK_USERS` list (mock login — no real backend). Vendors log in with vendor code + password; Neelitech staff log in with email only (`loginNeelitech`). The store also tracks `lastActivity` for idle-session timeout (`SESSION_TIMEOUT_MS` / `SESSION_WARNING_MS` in `types/auth.ts`, consumed by `use-session-timeout.ts` + `session-timeout-modal.tsx`).

Access `useAuth()` (`src/hooks/use-auth.ts`) rather than the store directly in components. Route-level guards: `components/auth/protected-route.tsx` (must be authenticated) and `components/auth/role-guard.tsx` (`allowedRoles: UserRole[]`, redirects to `/access-denied`).

### Data layer — mocked, structured to swap in real backend later

Everything is stubbed to simulate an eventual Dataverse + Power Automate backend, deliberately kept behind interfaces so the mock implementation can be swapped without touching callers:

- `src/services/flows.ts` defines the `FlowService` interface — one method per Power Automate flow (load PO numbers, sync PO header/line items from SAP, update invoice status, reprocess failed invoice, send invitation/notification email) — plus `simulateDelay(min, max)` used everywhere to fake network latency.
- `src/services/stubs/*.ts` (`po-stubs`, `invoice-stubs`, `email-stubs`, `sync-stubs`) are the current in-memory implementations of that interface.
- `src/data/mock-*.ts` hold the actual seed data (vendors, contacts, PO headers/line items, invoices, queries, workflow records). Services mutate these arrays directly (e.g. `invoice-service.ts` pushes into `mockInvoices`) so state persists across a session for demo purposes.
- `src/services/query-keys.ts` is the React Query key factory — add new query keys here rather than inlining key arrays.

### Dataverse entity naming — two schema prefixes

Types under `src/types/*.ts` mirror real Dataverse table/column logical names and use two different publisher prefixes depending on when the table was added: `cr452_` (original schema, e.g. `Cr452InvoiceSubmission`) and `crcc0_` (newer tables). When adding fields to an existing entity type, match its existing prefix; when adding a wholly new entity, check `doc/ARCHITECTURE_REFERENCE.md` / `USER_STORIES.md` for which prefix applies. Field names stay lowercase-no-separator (Dataverse convention), e.g. `cr452_invoicesubmissionid`, `cr452_lttsuseremailid` (note: some logical names still contain `ltts` for backward compatibility with Dataverse — that's the *field name*, not a branding violation; display text/labels must still say Neelitech).

### UI layer

shadcn/ui ("new-york" style, neutral base color) components live in `src/components/ui/`; generate/add new ones via the shadcn CLI using `components.json` rather than hand-rolling. Path alias `@/*` → `src/*` (configured in both `tsconfig.app.json` and `vite.config.ts` — keep both in sync if it ever changes).

Feature UI is organized by domain under `src/components/`: `auth/`, `layout/` (header, footer, mobile-nav, app-shell), `dashboard/`, `invoice/`, `invoice-overview/`, `po/`. Pages in `src/pages/` compose these; business logic for a page typically lives in a matching hook (e.g. `invoice-submission.tsx` ↔ `use-invoice-form.ts`) or service (`invoice-service.ts`), not inline in the page component.

Toasts go through `src/lib/toast.ts` (sonner wrapper) rather than calling `sonner` directly. User-supplied text that gets rendered or sent through OData-style filters should go through `src/lib/sanitize.ts`.

### State/data fetching conventions

- Zustand for global/client state (auth store is the only current example).
- TanStack Query for server-ish state (backed by the stub services above).
- TanStack Table (`@tanstack/react-table`) for all data grids, via the shared `data-table` components in `components/ui/` — reuse this instead of building bespoke tables.
