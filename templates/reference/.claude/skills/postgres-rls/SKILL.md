---
name: postgres-rls
description: Génère les migrations PostgreSQL 15 pour le SaaS Éducation Sénégal avec Row Level Security multi-tenant — tables tenant (school_id NOT NULL), policies RLS par rôle Keycloak (director, teacher, parent, student), index, triggers _updated_at et extensions pgcrypto/pg_trgm. À déclencher quand l'utilisateur dit "ajoute la table X", "écris la migration", "crée les policies RLS", ou touche au schéma SQL du projet.
type: skill
---

# postgres-rls

## Objectif

Produire des migrations SQL robustes garantissant l'isolation multi-tenant au niveau base (défense en profondeur indépendante du code applicatif).

## Quand l'utiliser

- Nouvelle table tenant à créer
- Ajout d'une colonne ou d'un index
- Nouvelles policies RLS à écrire ou modifier
- Vue analytique pré-construite (`student_payment_summary`, etc.)

## Stack de référence

- **PostgreSQL 15**
- **Drizzle ORM** comme source de vérité du schéma (voir skill `drizzle-schema`). Les policies RLS sont **déclarées directement dans `pgTable()`** via `pgPolicy()` — elles apparaissent automatiquement dans les migrations générées par `drizzle-kit generate`.
- Extensions : `pgcrypto` (UUID), `pg_trgm` (recherche)
- Session variables posées par `tenantDB()` dans la transaction : `app.current_school_id`, `app.current_user_id`, `app.current_role`
- 5 rôles Keycloak : `super_admin`, `director`, `teacher`, `parent`, `student`

Ce skill décrit le SQL de référence que Drizzle doit produire. Si Drizzle ne génère pas une policy exotique dont tu as besoin (ex: recursive CTE), tu peux écrire la migration manuellement en SQL dans `apps/api/src/db/migrations/` — elle sera appliquée par `drizzle-kit migrate` comme n'importe quelle autre.

## Template de table tenant

```sql
CREATE TABLE <table> (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id uuid NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  -- colonnes métier
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  -- pour les entités offline-first uniquement
  _sync_status text CHECK (_sync_status IN ('pending','synced','conflict')),
  _updated_at bigint
);

CREATE INDEX idx_<table>_school ON <table>(school_id);

ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <table> FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON <table>
  USING (school_id = current_setting('app.current_school_id', true)::uuid)
  WITH CHECK (school_id = current_setting('app.current_school_id', true)::uuid);

CREATE TRIGGER trg_<table>_updated_at
  BEFORE UPDATE ON <table>
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

## Policies fines par rôle

Pour les tables sensibles (`grades`, `absences`, `payments`, `bulletins`), ajouter des policies additionnelles :

- **Parent** : ne voit que les lignes de ses propres enfants (`WHERE student_id IN (SELECT student_id FROM student_parents WHERE parent_user_id = current_setting('app.current_user_id')::uuid)`)
- **Teacher** : ne voit que les classes où il enseigne (`class_subjects.teacher_id = current_user`)
- **Director** : voit tout l'établissement (policy tenant basique suffit)
- **Student** : lecture seule sur ses propres données

## Workflow

1. **Définir la table dans `apps/api/src/db/schema/<n>.ts`** en suivant le skill `drizzle-schema` : `pgTable` + colonnes + index + `pgPolicy('tenant_isolation', ...)` + policies restrictives par rôle si pertinent + `.enableRLS()`
2. **Générer la migration** : `pnpm drizzle-kit generate --name <slug>` — le fichier SQL apparaît dans `apps/api/src/db/migrations/` avec `CREATE TABLE`, `ENABLE RLS`, `FORCE RLS`, `CREATE POLICY ...` tirés du schéma TS
3. **Relire la migration** manuellement : vérifier que toutes les policies attendues sont présentes, que les index sont là, que `FORCE` est bien émis
4. **Ajouter le trigger `set_updated_at`** manuellement dans le SQL si besoin (Drizzle ne gère pas les triggers nativement en 2026) — ou utiliser `$onUpdate(() => new Date())` dans la colonne Drizzle comme alternative applicative
5. **Appliquer** : `pnpm drizzle-kit migrate`
6. **Tester l'isolation** : insérer une ligne avec `school_id = A` via `tenantDB` contexte A, changer le contexte vers B, lancer le même SELECT, vérifier 0 ligne visible. Ce test est dans `__tests__/<feature>.controller.e2e-spec.ts`.

Pour une policy exotique que Drizzle ne sait pas exprimer (ex : recursive CTE pour une hiérarchie parent-enfant complexe), écris une migration SQL manuelle dans le même dossier `migrations/` — `drizzle-kit` les applique dans l'ordre alphabétique avec les siennes.

## Pièges à éviter

- **Oublier `FORCE ROW LEVEL SECURITY`** : sans ça, le owner de la table bypasse les policies
- Oublier le `WITH CHECK` : les UPDATE peuvent réécrire `school_id` vers un autre tenant
- Index manquant sur `school_id` : performance catastrophique
- `current_setting('app.current_school_id')` sans le `, true` : erreur si la variable n'existe pas
- Pas de policy → RLS activé bloque tout
- FK cross-tenant : toujours vérifier que la FK pointe sur une table du même tenant

## Livrable

Fichier SQL dans `apps/api/src/db/migrations/<timestamp>_<slug>.sql` (généré par `drizzle-kit generate`), relu manuellement, appliqué en local et staging, et couvert par un test d'isolation 2 schools dans les tests e2e du module concerné.
