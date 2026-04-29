# Script d'interview

## Règle d'or : une seule réponse, toutes les questions

Jamais de ping-pong question par question. Poser toutes les questions manquantes dans un seul message. Utiliser `ask_user_input_v0` pour les choix fermés (max 3 questions par appel).

## Format texte brut

```markdown
J'ai besoin de quelques précisions sur ta stack avant de générer le bundle.
Réponds en bloc — un mot ou deux par point suffit.

**Projet**
1. Nom (slug-case) :
2. Pitch (1 phrase) :

**Backend**
3. Framework + langage :
4. Package manager :

**Base de données**
5. Engine + version :
6. ORM :
7. Multi-tenant : single-db-rls | schema-per-tenant | database-per-tenant | single-db-shared | none

**Auth**
8. Provider :

**Infrastructure**
9. Queue system (ou "aucun") :
10. Storage fichiers (ou "aucun") :

**Intégrations**
11. Liste des APIs externes (ou "aucune") :

**Frontend**
12. Web (ou "aucun") :
13. Mobile (ou "aucun") :
14. Offline-first ? (oui/non)

**Sortie**
15. Dossier cible (défaut ./):
```

## Réduire les questions

Si la description initiale contient des indices, **inférer sans demander** :

- "App NestJS" → TS, pnpm
- "SaaS Next.js + Supabase" → Next.js 14, Postgres, supabase-auth
- "App Rails" → Ruby, ActiveRecord, Bundler
- "FastAPI + SQLAlchemy" → Python, uv

Pour chaque inférence, signaler : "J'ai déduit X, Y, Z. Confirme ou corrige."

## Réponse ambiguë

Si l'utilisateur est hésitant sur un point critique (ex : multi-tenant), poser **une seule** question de clarification ciblée, pas reposer tout le bloc.

## Ne pas sur-interviewer

Si une décision a un défaut sensé (testing framework, CI, logging), ne pas demander. Les défauts sont dans `stack-dimensions.md`.
