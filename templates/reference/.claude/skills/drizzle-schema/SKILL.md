---
name: drizzle-schema
description: Skill central d'accès base de données avec Drizzle ORM pour le SaaS Éducation Sénégal. Définit les tables tenant multi-tenant, les policies RLS PostgreSQL déclaratives (pgPolicy, pgRole, enableRLS), le pattern AsyncLocalStorage + tenantDB() pour le contexte tenant, les migrations drizzle-kit, et l'intégration avec drizzle-zod pour les DTO. À déclencher dès qu'une tâche backend touche au schéma, aux repositories, aux requêtes SQL, à un service DB, ou mentionne "table", "migration", "Drizzle", "schema.ts", "tenantDB", "repository".
---

# drizzle-schema

## Objectif

Produire un accès base de données type-safe, multi-tenant, avec RLS comme ligne de défense finale. Toute requête métier passe par `tenantDB()` pour que `SET LOCAL app.current_school_id` soit actif, ce qui active automatiquement les policies RLS PostgreSQL.

## Stack

- **Drizzle ORM** (`drizzle-orm` + `drizzle-kit` + `pg`)
- **PostgreSQL 15** avec extensions `pgcrypto`, `pg_trgm`
- **AsyncLocalStorage** Node.js natif pour propager le contexte tenant
- **drizzle-zod** pour générer les schémas zod depuis le schéma Drizzle
- Intégration NestJS : `DatabaseModule` provide `db` (pool) et `tenantDB` (wrapper)

## Arborescence

```
apps/api/src/db/
├── schema.ts               # re-export de toutes les tables
├── schema/
│   ├── schools.ts         # école (tenant root)
│   ├── users.ts           # user → keycloak_user_id
│   ├── students.ts
│   ├── teachers.ts
│   ├── classes.ts
│   ├── subjects.ts
│   ├── grades.ts
│   ├── absences.ts
│   ├── payments.ts
│   └── ...
├── index.ts               # export db, tenantDB, adminDB, tenantContext
├── tenant-context.ts      # AsyncLocalStorage
└── migrations/            # généré par drizzle-kit
    └── <timestamp>_<slug>.sql
```

## Pattern : table tenant avec RLS déclaratif

```ts
// apps/api/src/db/schema/students.ts
import { pgTable, uuid, text, date, timestamp, index, pgPolicy } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';
import { schools } from './schools';

export const students = pgTable(
  'students',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    schoolId: uuid('school_id').notNull().references(() => schools.id, { onDelete: 'restrict' }),
    firstName: text('first_name').notNull(),
    lastName: text('last_name').notNull(),
    birthDate: date('birth_date').notNull(),
    createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    index('idx_students_school').on(t.schoolId),
    index('idx_students_school_name').on(t.schoolId, t.lastName, t.firstName),

    // Policy permissive de base : isolation par school_id
    pgPolicy('tenant_isolation', {
      as: 'permissive',
      for: 'all',
      to: 'public',
      using: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
      withCheck: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
    }),
  ],
).enableRLS();
```

**Points clés** :
- `.enableRLS()` active ET force RLS (Drizzle émet `ENABLE + FORCE` dans la migration)
- `pgPolicy('tenant_isolation', ...)` est **versionné dans ton code TypeScript** — pas dans un fichier SQL séparé
- `using` filtre les SELECT/UPDATE/DELETE, `withCheck` empêche d'UPDATE/INSERT vers un autre tenant
- `current_setting('app.current_school_id', true)` — le `true` évite l'erreur si la variable n'est pas posée (retourne `NULL`, donc le WHERE est faux, donc rien n'est visible — fail-safe)

## Pattern : policies fines par rôle

Pour les tables sensibles (`grades`, `absences`, `payments`, `bulletins`), ajouter des policies **restrictives** par rôle Keycloak. Une policy restrictive s'ajoute à la permissive — les deux doivent passer.

```ts
// apps/api/src/db/schema/grades.ts
export const grades = pgTable(
  'grades',
  {
    id: uuid('id').primaryKey().defaultRandom(),
    schoolId: uuid('school_id').notNull().references(() => schools.id),
    studentId: uuid('student_id').notNull().references(() => students.id),
    subjectId: uuid('subject_id').notNull().references(() => subjects.id),
    teacherId: uuid('teacher_id').notNull().references(() => users.id),
    value: numeric('value', { precision: 4, scale: 2 }).notNull(),
    // ...
  },
  (t) => [
    index('idx_grades_school_student').on(t.schoolId, t.studentId),

    // permissive : tenant isolation
    pgPolicy('tenant_isolation', {
      as: 'permissive',
      for: 'all',
      to: 'public',
      using: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
      withCheck: sql`school_id = current_setting('app.current_school_id', true)::uuid`,
    }),

    // restrictive : un parent ne voit que les notes de ses enfants
    pgPolicy('parent_own_children_only', {
      as: 'restrictive',
      for: 'select',
      to: 'public',
      using: sql`
        current_setting('app.current_role', true) <> 'parent'
        OR student_id IN (
          SELECT student_id FROM student_parents
          WHERE parent_user_id = current_setting('app.current_user_id', true)::uuid
        )
      `,
    }),

    // restrictive : un teacher ne voit que les notes de ses propres cours
    pgPolicy('teacher_own_classes_only', {
      as: 'restrictive',
      for: 'select',
      to: 'public',
      using: sql`
        current_setting('app.current_role', true) <> 'teacher'
        OR EXISTS (
          SELECT 1 FROM class_subjects cs
          WHERE cs.teacher_id = current_setting('app.current_user_id', true)::uuid
            AND cs.subject_id = grades.subject_id
        )
      `,
    }),
  ],
).enableRLS();
```

**Règle** : `director` et `super_admin` ne passent par aucune policy restrictive — la permissive `tenant_isolation` leur suffit. Les restrictives ciblent `parent`, `teacher`, `student` uniquement.

## Pattern : `tenantDB()` avec AsyncLocalStorage

```ts
// apps/api/src/db/tenant-context.ts
import { AsyncLocalStorage } from 'node:async_hooks';

export type TenantContext = {
  schoolId: string;
  userId: string;
  role: 'super_admin' | 'director' | 'teacher' | 'parent' | 'student';
};

export const tenantContext = new AsyncLocalStorage<TenantContext>();
```

```ts
// apps/api/src/db/index.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { sql } from 'drizzle-orm';
import * as schema from './schema';
import { tenantContext } from './tenant-context';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
export { tenantContext };
export type DB = typeof db;
export type Tx = Parameters<Parameters<DB['transaction']>[0]>[0];

/**
 * Wrapper OBLIGATOIRE pour toutes les requêtes métier.
 * Injecte le contexte tenant dans la transaction PostgreSQL via SET LOCAL.
 * Les policies RLS s'activent automatiquement grâce aux session variables.
 */
export async function tenantDB<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  const ctx = tenantContext.getStore();
  if (!ctx) {
    throw new Error('tenantDB called without tenant context — missing TenantInterceptor?');
  }
  return db.transaction(async (tx) => {
    await tx.execute(sql`set local app.current_school_id = ${sql.raw(`'${ctx.schoolId}'`)}`);
    await tx.execute(sql`set local app.current_user_id = ${sql.raw(`'${ctx.userId}'`)}`);
    await tx.execute(sql`set local app.current_role = ${sql.raw(`'${ctx.role}'`)}`);
    return fn(tx);
  });
}

/**
 * Accès BYPASS RLS — réservé aux webhooks, jobs techniques, et tests.
 * JAMAIS dans un flow utilisateur. Code reviewer rejette en review si utilisé hors allowlist.
 */
export async function adminDB<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  return db.transaction(fn);
}
```

## Pattern : service qui utilise `tenantDB`

```ts
// apps/api/src/modules/students/students.service.ts
import { Injectable } from '@nestjs/common';
import { tenantDB } from '../../db';
import { students } from '../../db/schema/students';
import { eq, and } from 'drizzle-orm';
import type { CreateStudentDto } from './dto/create-student.dto';

@Injectable()
export class StudentsService {
  async findAll() {
    return tenantDB(async (tx) => tx.select().from(students).orderBy(students.lastName));
  }

  async findOne(id: string) {
    return tenantDB(async (tx) => {
      const [row] = await tx.select().from(students).where(eq(students.id, id));
      return row ?? null;
    });
  }

  async create(dto: CreateStudentDto, schoolId: string) {
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

**Notes** :
- Aucun `where(eq(students.schoolId, ...))` manuel : la RLS le fait. Mais on met `schoolId` dans le `insert` (WITH CHECK l'exige).
- Le service ne connaît pas `tenantContext` directement — l'interceptor NestJS l'a déjà posé.
- Si tu oublies `tenantDB()` et fais `db.select().from(students)` directement, PostgreSQL renvoie **zéro ligne** (RLS fail-safe) — le bug est immédiatement visible en test.

## Pattern : DTO via `drizzle-zod`

```ts
// apps/api/src/modules/students/dto/create-student.dto.ts
import { createInsertSchema } from 'drizzle-zod';
import { students } from '../../../db/schema/students';
import { z } from 'zod';

export const createStudentSchema = createInsertSchema(students, {
  firstName: (s) => s.min(1).max(100),
  lastName: (s) => s.min(1).max(100),
}).omit({ id: true, schoolId: true, createdAt: true, updatedAt: true });

export type CreateStudentDto = z.infer<typeof createStudentSchema>;
```

`schoolId` est omis du DTO parce qu'il vient du contexte tenant, jamais du client. Toute tâche qui expose `schoolId` dans un DTO est une faille — rejet en review.

## Migrations

```bash
# Générer depuis le schéma TypeScript
pnpm drizzle-kit generate --name add_students_table

# Le fichier SQL dans apps/api/src/db/migrations/<timestamp>_add_students_table.sql
# contient déjà : CREATE TABLE + ENABLE RLS + FORCE RLS + CREATE POLICY ...
# tiré directement du pgTable + pgPolicy. Aucun SQL manuel à ajouter.

# Appliquer
pnpm drizzle-kit migrate
```

**Règle** : ne JAMAIS éditer une migration après qu'elle soit appliquée en staging ou prod. Si besoin, nouvelle migration.

## Configuration `drizzle.config.ts`

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './apps/api/src/db/schema.ts',
  out: './apps/api/src/db/migrations',
  dbCredentials: { url: process.env.DATABASE_URL! },
  strict: true,       // détecte les renames ambigus → prompt interactif
  verbose: true,
});
```

## Checklist avant de livrer une nouvelle table tenant

- [ ] `schoolId: uuid('school_id').notNull().references(() => schools.id)` présent
- [ ] Index composite commençant par `schoolId` pour chaque requête fréquente
- [ ] `pgPolicy('tenant_isolation', ...)` permissive avec `using` + `withCheck`
- [ ] Policies restrictives par rôle si la table est sensible (grades, payments, absences)
- [ ] `.enableRLS()` appelé
- [ ] Triggers `updated_at` ajoutés (via SQL raw dans la migration ou via `$onUpdate` Drizzle)
- [ ] Schéma zod d'insert via `createInsertSchema().omit({ schoolId: true })`
- [ ] Service backend wrappe **toutes** les requêtes dans `tenantDB()`
- [ ] **Test d'isolation 2 schools obligatoire** : créer une ligne avec school A, changer le contexte vers school B, vérifier que la ligne est invisible
- [ ] Migration générée via `drizzle-kit generate` et relue manuellement
- [ ] Migration appliquée en local avec succès

## Pièges à éviter

- **Requête hors `tenantDB()`** → renvoie 0 ligne silencieusement (RLS fail-safe). Symptôme : "ma liste est vide en e2e alors que la DB a des données". Solution : wrapper la requête.
- **Oublier `withCheck`** : un UPDATE malicieux peut changer `school_id` et faire fuiter la ligne vers un autre tenant.
- **Oublier `.references(() => schools.id)`** sur `schoolId` : orphelins possibles si une école est supprimée.
- **FK cross-tenant** : une table enfant qui référence une entité d'un autre tenant. Toujours vérifier que la FK pointe sur une entité du même `school_id`.
- **`@@unique` sans `schoolId`** : collision entre tenants (ex : `email` unique global → deux écoles ne peuvent pas avoir le même email d'élève).
- **Utiliser `adminDB()` dans un flow user** : bypass RLS → fuite. Reviewer rejette automatiquement sauf si le fichier est dans `webhooks/`, `jobs/`, ou `test/`.
- **Oublier l'index sur `schoolId`** : perf catastrophique car RLS ajoute un `WHERE school_id = ...` à chaque requête.
- **`omit({ schoolId: true })` oublié dans le DTO** : client peut injecter un `schoolId` différent → sauvé par WITH CHECK côté DB mais faille en amont.

## Livrable

Nouvelle table dans `apps/api/src/db/schema/<name>.ts`, re-exportée depuis `schema.ts`, migration générée et appliquée, service backend utilisant `tenantDB()`, DTO zod via `createInsertSchema`, test d'isolation 2 schools.
