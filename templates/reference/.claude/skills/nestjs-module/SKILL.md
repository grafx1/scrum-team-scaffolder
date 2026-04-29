---
name: nestjs-module
description: Structure standard d'un module NestJS pour le backend du projet — contrôleur, service Drizzle, DTO zod, guard Keycloak, processor Bull, tests. À utiliser systématiquement quand un agent (backend-dev, architect) crée un nouveau module métier, un controller, un service, un DTO ou un job async. Garantit la cohérence entre tous les modules du backend et l'usage correct de tenantDB().
---

# Module NestJS standard

Tout nouveau module backend suit cette structure. C'est la convention du projet — ne pas s'en écarter sans justification dans un ADR.

## 1. Arborescence

```
apps/api/src/modules/<feature>/
├── <feature>.module.ts
├── <feature>.controller.ts
├── <feature>.service.ts
├── dto/
│   ├── create-<entity>.dto.ts
│   └── update-<entity>.dto.ts
├── jobs/                   # handlers Bull Queue (processors)
└── __tests__/
    ├── <feature>.service.spec.ts
    └── <feature>.controller.e2e-spec.ts
```

## 2. Module

```typescript
// students.module.ts
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { StudentsController } from './students.controller';
import { StudentsService } from './students.service';
import { StudentsSyncProcessor } from './jobs/students-sync.processor';

@Module({
  imports: [BullModule.registerQueue({ name: 'students-sync' })],
  controllers: [StudentsController],
  providers: [StudentsService, StudentsSyncProcessor],
  exports: [StudentsService],
})
export class StudentsModule {}
```

## 3. Controller

```typescript
// students.controller.ts
import { Controller, Get, Post, Body, Headers, UseGuards, UseInterceptors } from '@nestjs/common';
import { KeycloakGuard } from '../auth/keycloak.guard';
import { CurrentTenant } from '../auth/current-tenant.decorator';
import { IdempotencyInterceptor } from '../common/idempotency.interceptor';
import { ZodValidationPipe } from '../common/zod-validation.pipe';
import { StudentsService } from './students.service';
import { createStudentSchema, type CreateStudentDto } from './dto/create-student.dto';

@Controller('students')
@UseGuards(KeycloakGuard)
export class StudentsController {
  constructor(private readonly service: StudentsService) {}

  @Get()
  list() {
    return this.service.findAll();
  }

  @Post()
  @UseInterceptors(IdempotencyInterceptor)
  create(
    @CurrentTenant() schoolId: string,
    @Headers('idempotency-key') idempotencyKey: string,
    @Body(new ZodValidationPipe(createStudentSchema)) dto: CreateStudentDto,
  ) {
    return this.service.create(dto, schoolId, idempotencyKey);
  }
}
```

**Règles** :
- `@UseGuards(KeycloakGuard)` au niveau controller, **jamais** oublié.
- Le `schoolId` vient TOUJOURS de `@CurrentTenant()` (décorateur qui lit l'`AsyncLocalStorage` `tenantContext`), **jamais** d'un param ou du body.
- Le service lit lui aussi le tenant depuis le contexte via `tenantDB()` — le `schoolId` passé en argument ici est pour l'insert (champ requis dans le INSERT).
- Les mutations exposées à un client offline utilisent `IdempotencyInterceptor` + header `idempotency-key` (voir skill `sync-queue-offline`).

## 4. Service — Drizzle via `tenantDB()`

```typescript
// students.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { tenantDB } from '../../db';
import { students } from '../../db/schema/students';
import { eq } from 'drizzle-orm';
import type { CreateStudentDto } from './dto/create-student.dto';

@Injectable()
export class StudentsService {
  findAll() {
    return tenantDB(async (tx) =>
      tx.select().from(students).orderBy(students.lastName),
    );
  }

  async findOne(id: string) {
    return tenantDB(async (tx) => {
      const [row] = await tx.select().from(students).where(eq(students.id, id));
      if (!row) throw new NotFoundException();
      return row;
    });
  }

  async create(dto: CreateStudentDto, schoolId: string, idempotencyKey: string) {
    return tenantDB(async (tx) => {
      const [row] = await tx
        .insert(students)
        .values({ ...dto, schoolId })
        .returning();
      return row;
    });
  }
}
```

**Règles** :
- **Toute** requête est wrappée dans `tenantDB(async (tx) => ...)`. Jamais d'import de `db` directement dans un service métier.
- `schoolId` passé explicitement dans le `values` de l'INSERT (WITH CHECK RLS l'exige).
- Pas besoin d'ajouter `where(eq(students.schoolId, ctx.schoolId))` aux SELECT — la policy RLS le fait automatiquement grâce à `SET LOCAL app.current_school_id`.
- Jamais de `any`. Les types viennent de l'inférence Drizzle.
- Voir skill `drizzle-schema` pour le pattern complet `tenantDB`.

## 5. DTO avec `drizzle-zod`

```typescript
// dto/create-student.dto.ts
import { createInsertSchema } from 'drizzle-zod';
import { students } from '../../../db/schema/students';
import { z } from 'zod';

export const createStudentSchema = createInsertSchema(students, {
  firstName: (s) => s.min(1).max(100),
  lastName: (s) => s.min(1).max(100),
}).omit({ id: true, schoolId: true, createdAt: true, updatedAt: true });

export type CreateStudentDto = z.infer<typeof createStudentSchema>;
```

- `omit({ schoolId: true })` est **obligatoire** — le tenant vient du JWT, pas du client.
- Le schéma zod est **aussi** exporté vers `packages/api-contract` pour être réutilisé côté frontend (react-hook-form + `zodResolver`).

## 6. Job Bull Queue (si async)

```typescript
// jobs/students-sync.processor.ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';
import { tenantContext } from '../../../db';

type NotifyPayload = {
  schoolId: string;
  userId: string;
  role: 'parent';
  studentId: string;
  message: string;
};

@Processor('students-sync')
export class StudentsSyncProcessor {
  @Process('notify-parent')
  async notifyParent(job: Job<NotifyPayload>) {
    const { schoolId, userId, role, studentId, message } = job.data;
    // Rejouer le contexte tenant dans le worker — obligatoire
    return tenantContext.run({ schoolId, userId, role }, async () => {
      // logique métier, ici le service utilise tenantDB() normalement
    });
  }
}
```

**Règles jobs** : voir skill `bull-redis-job`.
- `attempts: 3`, `backoff: { type: 'exponential', delay: 2000 }`
- `removeOnComplete: true`, `removeOnFail: 100`
- Idempotence obligatoire — clé dans le payload
- Contexte tenant rejoué dans `tenantContext.run()` dans le handler

## 7. Tests (obligatoires)

**Unitaire (service)** :
```typescript
// __tests__/students.service.spec.ts
describe('StudentsService', () => {
  it('crée un student avec le bon schoolId', async () => { /* ... */ });
  it('findAll retourne uniquement les students du tenant courant', async () => { /* ... */ });
});
```

**E2E (controller)** avec vraie Postgres :
```typescript
// __tests__/students.controller.e2e-spec.ts
describe('POST /students', () => {
  it('401 sans JWT', async () => { /* ... */ });
  it('201 avec JWT valide et idempotency-key', async () => { /* ... */ });
  it('renvoie la même réponse si idempotency-key réutilisée', async () => { /* ... */ });
  it('isole strictement entre deux schools', async () => {
    // Créer un student dans school A
    // Changer le JWT pour school B
    // GET /students → ne doit pas voir le student de A
  });
});
```

Le test d'isolation tenant est **obligatoire** pour toute table tenant (cf `drizzle-schema` et `postgres-rls`).

## 8. Checklist avant `in_review`

- [ ] Controller protégé par `KeycloakGuard`
- [ ] `schoolId` vient de `@CurrentTenant()`, jamais du client
- [ ] Toutes les requêtes DB wrappées dans `tenantDB()`
- [ ] DTO zod avec `createInsertSchema` + `.omit({ schoolId: true })`
- [ ] Schéma zod exporté vers `packages/api-contract`
- [ ] Endpoints de mutation supportent `Idempotency-Key`
- [ ] Jobs Bull avec `attempts`, `backoff`, `removeOnComplete`, contexte tenant rejoué
- [ ] Tests unitaires + e2e + test d'isolation tenant présents
- [ ] `pnpm lint && pnpm test && pnpm build` passent
