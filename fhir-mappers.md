# fhir-mappers

Pure TypeScript functions that map Prisma database records to FHIR R4 resources.

No side effects, no database access, no HTTP calls — input a Prisma record, get back a typed FHIR R4 object.
Uses `@types/fhir` for compile-time R4 type safety.

Imported as `@healthtech/fhir-mappers`.

---

## Functions

### `toFhirPatient`

```typescript
import { toFhirPatient } from '@healthtech/fhir-mappers';
import type { PrismaPatient } from '@healthtech/fhir-mappers';

const fhirPatient: fhir4.Patient = toFhirPatient(prismaPatient);
```

Maps to `fhir4.Patient`:

| Prisma field | FHIR field |
|---|---|
| `id` | `id` |
| `mrn` | `identifier[0].value` (system: `urn:oid:mrn`) |
| `firstName`, `lastName` | `name[0].given`, `name[0].family` |
| `dateOfBirth` | `birthDate` |
| `gender` (`M\|F\|U`) | `gender` (`male\|female\|unknown`) |

---

### `toFhirCoverage`

```typescript
import { toFhirCoverage } from '@healthtech/fhir-mappers';
import type { PrismaCoverage } from '@healthtech/fhir-mappers';

const fhirCoverage: fhir4.Coverage = toFhirCoverage(prismaCoverage);
```

Maps to `fhir4.Coverage`:

| Prisma field | FHIR field |
|---|---|
| `id` | `id` |
| `active` | `status` (`active` / `cancelled`) |
| `patientId` | `beneficiary.reference` (`Patient/:id`) |
| `payorName` | `payor[0].display` |
| `planId` | `class[0].value` (type: `plan`) |
| `periodStart`, `periodEnd` | `period.start`, `period.end` |

---

### `toFhirEpisodeOfCare`

```typescript
import { toFhirEpisodeOfCare } from '@healthtech/fhir-mappers';
import type { PrismaCase } from '@healthtech/fhir-mappers';

const fhirEpisode: fhir4.EpisodeOfCare = toFhirEpisodeOfCare(prismaCase);
```

Maps to `fhir4.EpisodeOfCare`:

| Prisma field | FHIR field |
|---|---|
| `id` | `id` |
| `status` (`OPEN\|IN_PROGRESS` → `active`, `CLOSED` → `finished`) | `status` |
| `patientId` | `patient.reference` (`Patient/:id`) |
| `category` | `type[0].text` |
| `openedAt`, `closedAt` | `period.start`, `period.end` |
| `assignedTo` | `careManager.display` |

---

## Usage in api-gateway

```typescript
import { toFhirPatient, toFhirCoverage, toFhirEpisodeOfCare } from '@healthtech/fhir-mappers';

// In a NestJS controller:
@Get('fhir/Patient/:id')
async fhirPatient(@Param('id') id: string) {
  const patient = await this.patientService.findOne(id);
  return toFhirPatient(patient);
}
```

---

## Testing

```bash
npx nx test fhir-mappers
```

13 unit tests across all three mappers — each mapper has ≥4 assertions covering resource type, field mapping, status mapping, and edge cases.

---

## Project layout

```
src/
  index.ts                    ← barrel export
  patient.mapper.ts           ← toFhirPatient + PrismaPatient interface
  patient.mapper.spec.ts
  coverage.mapper.ts          ← toFhirCoverage + PrismaCoverage interface
  coverage.mapper.spec.ts
  episode-of-care.mapper.ts   ← toFhirEpisodeOfCare + PrismaCase interface
  episode-of-care.mapper.spec.ts
```
