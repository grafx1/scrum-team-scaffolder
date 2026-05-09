---
name: nestjs-module
description: Structure standard d'un module NestJS pour le backend du projet — contrôleur, service Drizzle, DTO zod, guard Keycloak, processor Bull, tests. À utiliser systématiquement quand un agent (backend-dev, architect) crée un nouveau module métier, un controller, un service, un DTO ou un job async. Garantit la cohérence entre tous les modules du backend et l'usage correct de tenantDB().
---

# nestjs-module

## Phases

1. **Arborescence** — créer les fichiers dans `apps/api/src/modules/<feature>/`
2. **Module** — déclarer controller, service, Bull queue si async
3. **Controller** — poser `@UseGuards(KeycloakGuard)`, `@CurrentTenant()`, `ZodValidationPipe`, `IdempotencyInterceptor` sur mutations
4. **Service** — wrapper toutes les requêtes dans `tenantDB()`, passer `schoolId` dans `insert`
5. **DTO** — générer via `createInsertSchema().omit({ schoolId: true })`, exporter vers `packages/api-contract`
6. **Tests** — unit service + e2e controller + test d'isolation tenant (obligatoire)

## Arborescence

```
apps/api/src/modules/<feature>/
├── <feature>.module.ts
├── <feature>.controller.ts
├── <feature>.service.ts
├── dto/
│   ├── create-<entity>.dto.ts
│   └── update-<entity>.dto.ts
├── jobs/                        # uniquement si opération async
└── __tests__/
    ├── <feature>.service.spec.ts
    └── <feature>.controller.e2e-spec.ts
```

## Patterns de référence

**Controller** :
```typescript
@Controller('students')
@UseGuards(KeycloakGuard)
export class StudentsController {
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

**Service** :
```typescript
async create(dto: CreateStudentDto, schoolId: string, idempotencyKey: string) {
  return tenantDB(async (tx) => {
    const [row] = await tx.insert(students).values({ ...dto, schoolId }).returning();
    return row;
  });
}
```

**DTO** :
```typescript
export const createStudentSchema = createInsertSchema(students, {
  firstName: (s) => s.min(1).max(100),
}).omit({ id: true, schoolId: true, createdAt: true, updatedAt: true });
export type CreateStudentDto = z.infer<typeof createStudentSchema>;
```

**Job Bull** — contexte tenant obligatoire dans le processor :
```typescript
@Process('notify-parent')
async notifyParent(job: Job<{ schoolId: string; userId: string; role: string }>) {
  const { schoolId, userId, role } = job.data;
  return tenantContext.run({ schoolId, userId, role }, async () => { /* ... */ });
}
```

## Anti-patterns

- ❌ `schoolId` pris du body ou d'un param URL — doit venir de `@CurrentTenant()` uniquement
- ❌ Requête DB sans `tenantDB()` — retourne 0 ligne silencieusement (RLS fail-safe)
- ❌ `@UseGuards(KeycloakGuard)` oublié — endpoint public par défaut
- ❌ `schoolId` exposé dans le DTO client — faille d'injection de tenant
- ❌ Job Bull sans replay du `tenantContext.run()` dans le processor
- ❌ `adminDB()` dans un flow utilisateur — bypass RLS, rejet automatique en review

## Checklist avant `in_review`

- [ ] Controller avec `@UseGuards(KeycloakGuard)` au niveau classe
- [ ] `schoolId` vient de `@CurrentTenant()`, absent du DTO
- [ ] Toutes requêtes DB dans `tenantDB(async (tx) => ...)`
- [ ] DTO via `createInsertSchema().omit({ schoolId: true })` exporté vers `api-contract`
- [ ] Mutations avec `IdempotencyInterceptor` + header `idempotency-key`
- [ ] Jobs Bull : `attempts: 3`, backoff exponentiel, `tenantContext.run()` dans le processor
- [ ] Tests : unit service + e2e controller + isolation 2 schools présents
- [ ] `pnpm lint && pnpm test && pnpm build` passent
