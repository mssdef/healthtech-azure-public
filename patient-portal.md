# patient-portal

React SPA — patient dashboard and case management UI.

## Stack

| Layer | Technology |
|---|---|
| Framework | React 18 + React Router v6 |
| Server state | TanStack Query v5 |
| Client state | Zustand (auth token) |
| Forms | React Hook Form + Zod |
| Build | Vite 5 |
| Types | `@healthtech/shared-dto` (Zod schemas) |

## Local setup

**Prerequisites:** Node 20+, `apps/api-gateway` running on port 3000

```bash
# 1. Install dependencies (from repo root)
npm install

# 2. Start api-gateway (separate terminal)
npx nx serve api-gateway

# 3. Serve patient-portal
npx nx serve patient-portal
# → http://localhost:4200
```

Vite proxies all `/api` requests to `http://localhost:3000`, so no CORS config is needed locally.

## Demo credentials

| Username | Password | Role |
|---|---|---|
| `clinician` | `demo` | Clinician |
| `admin` | `demo` | Admin |

## Pages

| Route | Page | Description |
|---|---|---|
| `/login` | `LoginPage` | JWT login form |
| `/dashboard` | `DashboardPage` | Patient list with links to detail and cases |
| `/patients/:id` | `PatientDetailPage` | Demographics + active FHIR Coverage card |
| `/patients/:id/cases` | `PatientCasesPage` | Case list for a patient with status badges |
| `/cases/:id` | `CaseDetailPage` | Notes thread, status dropdown, document list |
| `/cases/new` | `NewCasePage` | Open a new case — patient search, category, priority |

All routes except `/login` require authentication. Unauthenticated requests are redirected to `/login`.

## State management

- **Auth** — Zustand store (`src/store/auth.store.ts`) with `persist` middleware; stores JWT token and username in `localStorage`.
- **Server state** — TanStack Query; typed query keys in `src/lib/queryKeys.ts`.
- **Forms** — React Hook Form + Zod resolvers using schemas from `@healthtech/shared-dto`.

## API client

All API calls go through `src/lib/api.ts` — a thin fetch wrapper that:
- Injects `Authorization: Bearer <token>` from the Zustand store
- Proxies to `/api` (Vite dev) or the deployed API Gateway URL (production)
- Throws on non-2xx responses

```typescript
import { api } from './lib/api';

const patients = await api.get<Patient[]>('/patients');
const newCase  = await api.post<Case>('/cases', { patientId, category, priority });
```

## Testing

```bash
npx nx test patient-portal
```

## Build

```bash
npx nx build patient-portal
# output → dist/apps/patient-portal/
```

The `dist/` output is deployed to Azure Static Web Apps via `.github/workflows/deploy.yml` on push to `main`.

## Project layout

```
index.html                   ← Vite entry
vite.config.ts               ← React plugin, port 4200, /api proxy
src/
  main.tsx                   ← React root, QueryClientProvider, BrowserRouter
  App.tsx                    ← Route tree, RequireAuth guard
  store/
    auth.store.ts            ← Zustand: token, username, setToken, logout
  lib/
    api.ts                   ← fetch wrapper with Bearer token injection
    queryKeys.ts             ← typed TanStack Query key factory
  components/
    Layout.tsx               ← sidebar nav, logout
  pages/
    LoginPage.tsx
    DashboardPage.tsx
    PatientDetailPage.tsx
    PatientCasesPage.tsx
    CaseDetailPage.tsx
    NewCasePage.tsx
project.json                 ← Nx targets: build, serve, test, lint
```
