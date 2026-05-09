---
name: postgres-rls
description: Génère les migrations PostgreSQL 15 pour le SaaS Éducation Sénégal avec Row Level Security multi-tenant — tables tenant (school_id NOT NULL), policies RLS par rôle Keycloak (director, teacher, parent, student), index, triggers _updated_at et extensions pgcrypto/pg_trgm. À déclencher quand l'utilisateur dit "ajoute la table X", "écris la migration", "crée les policies RLS", ou touche au schéma SQL du projet.
type: skill
---

# postgres-rls

## Phases

1. **Schéma TS** — définir `pgTable` + `pgPolicy` + `.enableRLS()` dans `drizzle-schema` (source de vérité)
2. **Générer** — `pnpm drizzle-kit generate --name <slug>`, relire le SQL produit
3. **Vérifier** — `ENABLE RLS` + `FORCE RLS` + toutes les policies + index sur `school_id` présents
4. **Trigger** — ajouter `set_updated_at()` manuellement si Drizzle ne le génère pas
5. **Appliquer** — `pnpm drizzle-kit migrate` en local
6. **Tester** — test d'isolation 2 schools dans `__tests__/<feature>.controller.e2e-spec.ts`

## Template SQL de référence

```sql
CREATE TABLE <table> (
  id          uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id   uuid        NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  -- colonnes métier
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_<table>_school ON <table>(school_id);

ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <table> FORCE ROW LEVEL SECURITY;

-- Policy permissive : isolation tenant de base
CREATE POLICY tenant_isolation ON <table>
  USING      (school_id = current_setting('app.current_school_id', true)::uuid)
  WITH CHECK (school_id = current_setting('app.current_school_id', true)::uuid);

CREATE TRIGGER trg_<table>_updated_at
  BEFORE UPDATE ON <table>
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

Pour tables sensibles (`grades`, `absences`, `payments`, `bulletins`), ajouter policies **restrictives** par rôle :
- `parent` : `student_id IN (SELECT student_id FROM student_parents WHERE parent_user_id = current_setting('app.current_user_id', true)::uuid)`
- `teacher` : `EXISTS (SELECT 1 FROM class_subjects WHERE teacher_id = current_setting('app.current_user_id', true)::uuid AND ...)`
- `director` / `super_admin` : policy permissive de base suffit

## Anti-patterns

- ❌ `FORCE ROW LEVEL SECURITY` absent — le owner de la table bypasse les policies
- ❌ `WITH CHECK` absent — UPDATE peut réécrire `school_id` vers un autre tenant
- ❌ `current_setting('app.current_school_id')` sans `, true` — erreur si variable non posée
- ❌ Index manquant sur `school_id` — RLS ajoute un `WHERE school_id = ?` à chaque requête
- ❌ FK cross-tenant — vérifier que la FK pointe sur une entité du même `school_id`
- ❌ Éditer une migration déjà appliquée — créer une nouvelle migration

## Checklist avant `in_review`

- [ ] `ENABLE RLS` + `FORCE RLS` présents dans la migration
- [ ] `tenant_isolation` avec `USING` + `WITH CHECK`
- [ ] Index composite `(school_id, ...)` pour chaque requête fréquente
- [ ] Policies restrictives par rôle si table sensible
- [ ] Trigger `set_updated_at` présent
- [ ] Migration relue manuellement, appliquée en local
- [ ] Test d'isolation 2 schools présent dans les e2e
