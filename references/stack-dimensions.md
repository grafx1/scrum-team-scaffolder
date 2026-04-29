# Taxonomie des dimensions de stack

## Dimensions obligatoires

### 1. Métadonnées projet
- `project.name` : slug-case
- `project.pitch` : 1-2 phrases décrivant le produit et le marché cible

### 2. Backend
- `backend.language` : typescript, python, ruby, java, go, rust, php, csharp
- `backend.framework` : TS: nestjs/express/fastify/hono · Python: fastapi/django/flask · Ruby: rails · Java: spring-boot · Go: echo/fiber/gin · PHP: laravel · C#: aspnet-core
- `backend.package-manager` : pnpm, npm, yarn, bun, uv, pip, poetry, bundler, cargo, maven, gradle (défaut inféré du framework)

### 3. Base de données et ORM
- `db.engine` : postgres, mysql, sqlite, mongodb, cockroach
- `db.orm` : TS: drizzle/prisma/typeorm/mikro-orm/kysely · Python: sqlalchemy/sqlmodel/django-orm · Ruby: activerecord · Java: jpa-hibernate/jooq · Go: gorm/sqlc/ent

### 4. Stratégie multi-tenant (critique)
- `single-db-rls` : une DB + tenant_id + policies RLS Postgres (recommandé)
- `single-db-shared` : une DB + tenant_id applicatif (pas de RLS)
- `schema-per-tenant` : un schéma Postgres par tenant
- `database-per-tenant` : une DB par tenant
- `none` : pas de multi-tenant

### 5. Auth
- `auth.provider` : keycloak, auth0, clerk, supabase-auth, firebase-auth, workos, custom-jwt, lucia, next-auth, devise (Rails), django-allauth
- `auth.multi-tenant-model` : organizations (natif KC/Auth0/Clerk), groups, custom
- `auth.token-type` : jwt-rs256, jwt-hs256, session-cookie, opaque-token

### 6. Queue / jobs async
- `queue.system` : TS: bull/bullmq · Python: celery/rq · Ruby: sidekiq/good_job · Java: quartz · Go: asynq · aws-sqs, rabbitmq, nats · `none`

### 7. Storage
- `storage.service` : cloudflare-r2, aws-s3, gcp-gcs, azure-blob, minio, local-fs, `none`

### 8. Frontend web
- `frontend-web.framework` : nextjs-14, nuxt3, sveltekit, remix, plain-react, astro, angular · `none`
- `frontend-web.styling` : tailwind, css-modules, styled-components, mui (défaut: tailwind)
- `frontend-web.components` : shadcn-ui, mui, chakra, mantine, radix, headlessui, custom
- `frontend-web.offline` : pwa, dexie, rxdb, `none`

### 9. Frontend mobile
- `frontend-mobile.framework` : expo, react-native-cli, flutter, native-ios, native-android, ionic · `none`
- `frontend-mobile.offline-db` : expo-sqlite, watermelondb, realm, drift (Flutter), `none`

### 10. Intégrations externes
Liste de `{ vendor, purpose }`. Si vide, aucun hub d'intégrations n'est généré.

Vendors courants (Claude peut générer fiablement) : Stripe, PayPal, Braintree, Adyen, Wave, Orange Money, Flutterwave, Paystack, Twilio, Africa's Talking, MessageBird, Resend, SendGrid, Postmark, Mailgun, SES, WhatsApp Business, FCM, APNS, PostHog, Mixpanel, Sentry, Plaid, DocuSign, Google Maps, Mapbox.

## Dimensions optionnelles (défauts sensés)

| Dimension | Défaut |
|---|---|
| `testing` | Jest si TS, pytest si Python, RSpec si Ruby |
| `i18n` | next-intl si Next.js, react-i18next sinon |
| `deploy` | Railway/Fly pour MVP |
| `forms` | react-hook-form + zod si TS |
| `monorepo` | pnpm-workspaces si frontend + backend + packages |
| `api-contract` | openapi si REST, trpc si full-stack TS |
| `ci` | github-actions |

## Ordre de collecte recommandé

1. Nom + pitch
2. Backend (framework + langage)
3. DB + ORM + multi-tenant (interdépendants)
4. Auth
5. Queue + Storage + Intégrations
6. Frontend web + mobile + offline

## Fallback pour technos inconnues

Si une techno n'est pas dans les listes : demander à l'utilisateur le "voisin connu" le plus proche. Adapter depuis le voisin, signaler en Phase 4.
