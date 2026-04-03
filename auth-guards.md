# auth-guards

Reusable NestJS JWT authentication guard and role-based access control (RBAC) decorator.

Imported as `@healthtech/auth-guards`.

---

## Exports

| Export | Type | Description |
|---|---|---|
| `JwtAuthGuard` | Guard | Validates Bearer JWT on incoming requests |
| `RolesGuard` | Guard | Enforces `@Roles()` requirements on a handler or controller |
| `Roles(...roles)` | Decorator | Declares which roles are allowed access |
| `UserPayload` | Interface | Shape of the decoded JWT payload |

---

## Usage

### Protect a route with JWT

```typescript
import { JwtAuthGuard } from '@healthtech/auth-guards';
import { UseGuards } from '@nestjs/common';

@UseGuards(JwtAuthGuard)
@Get('patients')
findAll() { ... }
```

### Restrict to specific roles

```typescript
import { JwtAuthGuard, RolesGuard, Roles } from '@healthtech/auth-guards';

@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Delete('patients/:id')
remove(@Param('id') id: string) { ... }
```

### Access the authenticated user

```typescript
import type { UserPayload } from '@healthtech/auth-guards';

@Post('cases')
create(@Body() dto: CreateCaseDto, @Request() req: { user: UserPayload }) {
  return this.caseService.create(dto, req.user.sub);
}
```

### `UserPayload` interface

```typescript
interface UserPayload {
  sub: string;               // user ID / username
  username: string;
  role: 'clinician' | 'admin';
}
```

---

## Setup in a NestJS module

`JwtAuthGuard` delegates to Passport's JWT strategy — the strategy must be configured in your app's `AuthModule`:

```typescript
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    PassportModule,
    JwtModule.register({ secret: process.env.JWT_SECRET, signOptions: { expiresIn: '8h' } }),
  ],
})
export class AuthModule {}
```

`RolesGuard` requires `Reflector` from `@nestjs/core`, which NestJS provides automatically when the guard is registered as a provider.

---

## Testing

```bash
npx nx test auth-guards
```

Unit tests cover:

- `RolesGuard` allows access when no roles are required
- `RolesGuard` allows access for a matching role
- `RolesGuard` denies access for a non-matching role

---

## Project layout

```
src/
  index.ts                   ← barrel export
  user-payload.interface.ts  ← UserPayload
  jwt-auth.guard.ts          ← JwtAuthGuard
  roles.decorator.ts         ← Roles() + ROLES_KEY
  roles.guard.ts             ← RolesGuard
  roles.guard.spec.ts
```
