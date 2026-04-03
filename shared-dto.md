# shared-dto

Zod schemas and TypeScript types shared across `apps/api-gateway` and `apps/patient-portal`.

Imported as `@healthtech/shared-dto` via the path alias defined in the root `tsconfig.base.json`.

---

## Installation

No separate install needed — resolved via workspace path alias:

```typescript
import { CreatePatientSchema, type PatientDto } from '@healthtech/shared-dto';
```

---

## Schemas and types

### Patient

```typescript
import { CreatePatientSchema, PatientSchema } from '@healthtech/shared-dto';
import type { CreatePatientDto, PatientDto } from '@healthtech/shared-dto';
```

| Schema | Fields |
|---|---|
| `CreatePatientSchema` | `mrn`, `firstName`, `lastName`, `dateOfBirth?`, `gender?` (`M\|F\|U`) |
| `PatientSchema` | extends Create + `id` (UUID), `createdAt` |

### Case

```typescript
import { CreateCaseSchema, CaseSchema, AddNoteSchema, CaseStatusSchema } from '@healthtech/shared-dto';
import type { CreateCaseDto, CaseDto, AddNoteDto, CaseStatus } from '@healthtech/shared-dto';
```

| Schema | Fields |
|---|---|
| `CaseStatusSchema` | `OPEN \| IN_PROGRESS \| CLOSED` |
| `CreateCaseSchema` | `patientId`, `category`, `priority` (`LOW\|MEDIUM\|HIGH`), `description?` |
| `CaseSchema` | extends Create + `id`, `status`, `assignedTo`, `openedAt`, `closedAt` |
| `AddNoteSchema` | `body` |

### Coverage

```typescript
import { CoverageSchema } from '@healthtech/shared-dto';
import type { CoverageDto } from '@healthtech/shared-dto';
```

| Schema | Fields |
|---|---|
| `CoverageSchema` | `id`, `patientId`, `planId`, `payorName`, `active`, `periodStart?`, `periodEnd?`, `eligibilityTransactionId?` |

### Auth

```typescript
import { LoginSchema, TokenResponseSchema } from '@healthtech/shared-dto';
import type { LoginDto, TokenResponse } from '@healthtech/shared-dto';
```

| Schema | Fields |
|---|---|
| `LoginSchema` | `username`, `password` |
| `TokenResponseSchema` | `access_token` |

---

## Validation example

```typescript
import { CreatePatientSchema } from '@healthtech/shared-dto';

const result = CreatePatientSchema.safeParse(input);
if (!result.success) {
  console.error(result.error.flatten());
}
```

---

## Testing

```bash
npx nx test shared-dto
```

---

## Rules

- No runtime dependencies beyond `zod`.
- Never import from `apps/` — this lib is consumed by apps, not the other way around.
- All DTO shapes used across the stack must be defined here.

---

## Project layout

```
src/
  index.ts           ← barrel export
  patient.schema.ts  ← CreatePatientSchema, PatientSchema
  case.schema.ts     ← CaseStatusSchema, CreateCaseSchema, CaseSchema, AddNoteSchema
  coverage.schema.ts ← CoverageSchema
  auth.schema.ts     ← LoginSchema, TokenResponseSchema
```
