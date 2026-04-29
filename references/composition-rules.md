# Règles de composition

Playbook pour adapter la référence à une autre stack.

## Règles générales

**G1 — Préserver la structure, adapter la substance.** Garder les sections, checklists, pièges. Adapter les exemples de code, noms, paths, outils CLI.

**G2 — Cohérence globale.** Un bundle ne mélange jamais deux ORMs, deux auth providers, deux queue systems.

**G3 — Nommage.** `<techno-qualifier>-<role>` en kebab-case : `drizzle-schema`, `prisma-schema`, `keycloak-org-auth`, `auth0-multi-tenant`, `bull-redis-job`, `sidekiq-job`, `nextjs-app-router`, `nuxt3-page`.

**G4 — Cross-références.** Si un skill B est renommé en B', mettre à jour toutes les références dans les agents et les autres skills.

**G5 — Descriptions.** Mentionner la techno spécifique dans le frontmatter YAML pour maximiser l'activation.

## Adaptations par dimension

### ORM

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| Prisma | drizzle-schema | `pgTable` → `model`, policies RLS dans migration SQL manuelle via `--create-only`, `tenantDB` → `prisma.$transaction` + `$executeRawUnsafe('SET LOCAL')` |
| TypeORM | drizzle-schema | `@Entity` decorators, `dataSource.transaction` + manager.query, migrations manuelles |
| SQLAlchemy | drizzle-schema | `Mapped/mapped_column`, tenant via **contextvars** + `SET LOCAL`, migrations Alembic, DTOs Pydantic |
| ActiveRecord | drizzle-schema | `class X < ApplicationRecord`, tenant via **CurrentAttributes**, migrations Rails + `execute "CREATE POLICY"` |
| GORM | drizzle-schema | Structs Go + tags gorm, tenant via **context.Context**, migrations golang-migrate |

### Auth

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| Auth0 | keycloak-org-auth | Auth0 Organizations, `org_id` top-level JWT, Management API |
| Clerk | keycloak-org-auth | SDK Clerk, `org_id/org_role` dans JWT, webhooks `organization.created` |
| Supabase Auth | keycloak-org-auth | `auth.uid()` dans les RLS, claim custom `tenant_id`, Edge Functions |
| Custom JWT | keycloak-org-auth | Endpoint login interne, JWKS auto-hébergé, `tenant_id` dans le claim |

### Queue

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| BullMQ | bull-redis-job | API `Worker` au lieu de `Queue.process` |
| Sidekiq | bull-redis-job | `include Sidekiq::Worker`, `perform_async`, tenant via `Current.set` |
| Celery | bull-redis-job | `@celery_app.task(bind=True)`, tenant via `contextvars` |
| SQS | bull-redis-job | DLQ native, `MessageDeduplicationId` pour idempotence |

### Storage

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| AWS S3 | cloudflare-r2-storage | Endpoint S3 standard, région explicite, `@aws-sdk/s3-request-presigner` |
| GCS | cloudflare-r2-storage | `@google-cloud/storage`, Service Account JSON |

### Backend framework

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| FastAPI | nestjs-module | Routers + `Depends()`, Pydantic DTOs, pytest + httpx, middleware contextvars |
| Express | nestjs-module | Middleware au lieu de Guards, DI manuel, validation zod custom |
| Rails | nestjs-module | Controllers + service objects, strong parameters, RSpec, `Current` attributes |
| Spring Boot | nestjs-module | Controllers + Services + Repositories, Jakarta Validation, JUnit + Testcontainers |

### Frontend web

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| Nuxt 3 | nextjs-app-router | Nuxt pages + `useFetch`, `@nuxtjs/i18n`, `shadcn-vue` |
| SvelteKit | nextjs-app-router | `+page.svelte` + `load` functions, `shadcn-svelte` |
| Remix | nextjs-app-router | Routes imbriquées, `loader/action` functions |

### Frontend mobile

| Cible | Adapter depuis | Changements clés |
|---|---|---|
| Flutter | expo-router-screen | `go_router`, `sqflite/drift`, `flutter_riverpod`, `flutter_test` |
| RN CLI | expo-router-screen | `react-navigation`, `react-native-sqlite-storage`, `fastlane` |

### Skills méthodologiques

**Toujours inclus, jamais exclus.** Adaptations :

| Skill | Ce qui s'adapte |
|---|---|
| `tdd-workflow` | Exemples test (Jest→pytest→RSpec), mocking (nock→responses→webmock), fixtures tenant |
| `ddd-modeling` | Exemples code, patterns events (Bull→Celery→Sidekiq), **glossaire réécrit pour le domaine** |
| `clean-code` | Exemples, grep checks, conventions nommage/fichiers au langage |
| `clean-architecture` | Mapping couches→fichiers au framework, grep checks |

### Intégrations

- ≥ 2 intégrations → hub + `references/<vendor>.md` (progressive disclosure)
- 1 seule → skill simple `<vendor>-integration/SKILL.md`
- Nommage hub : `<theme>-integrations` (ex : `payment-integrations`, `messaging-integrations`)

## Matrice de décision

| Stack a... | Inclure | Exclure |
|---|---|---|
| Multi-tenant RLS | ORM skill + auth skill + postgres-rls | — |
| Pas de multi-tenant | ORM simplifié, auth simplifié | postgres-rls, tests isolation |
| Frontend (web ou mobile) | 3 skills design + frontend-dev | — |
| Pas de frontend | — | 3 skills design, frontend-dev agent |
| Offline (mobile ou PWA) | sync-queue-offline | — |
| Pas d'offline | — | sync-queue-offline |
| Queue system | queue skill | — |
| Intégrations | hub d'intégrations + references | — |
| **Toute stack** | **tdd, ddd, clean-code, clean-archi** | **Jamais exclus** |
| **Toute stack** | **security-reviewer agent** | **Jamais exclus** |

## Fallback

Si combinaison inédite : adapter depuis le voisin le plus proche, documenter dans le rapport Phase 4. Ne jamais abandonner.
