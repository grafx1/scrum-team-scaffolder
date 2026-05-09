---
name: drizzle-schema
description: Skill central d'accès base de données avec Drizzle ORM pour le SaaS Éducation Sénégal. Définit les tables tenant multi-tenant, les policies RLS PostgreSQL déclaratives (pgPolicy, pgRole, enableRLS), le pattern AsyncLocalStorage + tenantDB() pour le contexte tenant, les migrations drizzle-kit, et l'intégration avec drizzle-zod pour les DTO. À déclencher dès qu'une tâche backend touche au schéma, aux repositories, aux requêtes SQL, à un service DB, ou mentionne "table", "migration", "Drizzle", "schema.ts", "tenantDB", "repository".
---

# drizzle-schema

## Phases

1. **Schéma** — créer `apps/api/src/db/schema/<name>.ts` avec `pgTable` + `pgPolicy` + `.enableRLS()`
2. **Re-export** — ajouter dans `apps/api/src/db/schema.ts`
3. **Migration** — `pnpm drizzle-kit generate --name add_<name>_table` puis relire le SQL généré
4. **Appliquer** — `pnpm drizzle-kit migrate` en local
5. **Service** — wrapper toutes les requêtes dans `tenantDB()`, `schoolId` explicite dans `insert`
6. **DTO** — `createInsertSchema().omit({ schoolId: true })`

## Pattern : table tenant

```typescript
// apps/api/src/db/schema/students.ts
export const students = pgTable('students', {
  id: uuid('id').primaryKey().defaultRandom(),
  schoolId: uuid('school_id').notNull().references(() => schools.id, { onDelete: 'restrict' }),
  firstName: text('first_name').notNull(),
  lastName: text('last_name').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('idx_students_school').on(t.schoolId),
  index('idx_students_school_name').on(t.schoolId, t.lastName, t.firstName),
  pgPolicy('tenant_isolation', {
    as: 'permissive', for: 'all', to: 'public',
    using: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
    withCheck: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
  }),
]).enableRLS();
```

Pour tables sensibles (`grades`, `payments`, `absences`) : ajouter des `pgPolicy` **restrictives** par rôle (`parent`, `teacher`). `director` et `super_admin` passent uniquement par la permissive.

## Pattern : `tenantDB()` (ne pas modifier)

```typescript
// apps/api/src/db/index.ts
export async function tenantDB<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  const ctx = tenantContext.getStore();
  if (!ctx) throw new Error('tenantDB called without tenant context — missing TenantInterceptor?');
  return db.transaction(async (tx) => {
    await tx.execute(sql`set local app.current_school_id = ${sql.raw(`'${ctx.schoolId}'`)}`);
    await tx.execute(sql`set local app.current_user_id = ${sql.raw(`'${ctx.userId}'`)}`);
    await tx.execute(sql`set local app.current_role = ${sql.raw(`'${ctx.role}'`)}`);
    return fn(tx);
  });
}

// adminDB — bypass RLS. UNIQUEMENT dans webhooks/, jobs/, tests/.
export async function adminDB<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  return db.transaction(fn);
}
```

## Anti-patterns

- ❌ `db.select().from(table)` sans `tenantDB()` — retourne 0 ligne silencieusement (RLS fail-safe)
- ❌ `withCheck` oublié — UPDATE peut changer `school_id` et faire fuiter la ligne
- ❌ `schoolId` exposé dans le DTO — `omit({ schoolId: true })` obligatoire
- ❌ `@@unique` sans `schoolId` — collision cross-tenant possible
- ❌ `adminDB()` dans un flow utilisateur — bypass RLS, rejet automatique en review
- ❌ Éditer une migration déjà appliquée en staging/prod — créer une nouvelle migration

## Checklist avant `in_review`

- [ ] `schoolId uuid NOT NULL REFERENCES schools(id)` présent
- [ ] Index composite commençant par `schoolId` pour chaque requête fréquente
- [ ] `pgPolicy('tenant_isolation')` avec `using` + `withCheck`
- [ ] Policies restrictives par rôle si table sensible
- [ ] `.enableRLS()` appelé
- [ ] `createInsertSchema().omit({ schoolId: true })` pour le DTO
- [ ] Toutes requêtes service dans `tenantDB()`
- [ ] Test d'isolation 2 schools présent (créer avec A, vérifier invisible depuis B)
- [ ] Migration générée + relue + appliquée en local
