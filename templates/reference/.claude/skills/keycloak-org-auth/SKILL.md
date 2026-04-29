---
name: keycloak-org-auth
description: Implémente l'authentification et le provisionning multi-tenant Keycloak 24 Organizations pour le SaaS Éducation Sénégal — realm schoolapp, mapping org_id ↔ schools.keycloak_org_id, claims JWT, résolution cache Redis, pose du contexte tenant via AsyncLocalStorage + tenantDB() Drizzle. À déclencher quand l'utilisateur dit "onboard une école", "auth multi-tenant", "KC Guard", "TenantInterceptor", "CurrentTenant decorator", ou touche aux flows OIDC Keycloak.
---

# keycloak-org-auth

## Objectif

Assurer que toute requête authentifiée résout correctement son tenant et pose le contexte dans `tenantContext` (AsyncLocalStorage) **avant** que les services ne soient appelés. Les services utilisent ensuite `tenantDB()` qui lit ce contexte et émet les `SET LOCAL` dans la transaction Drizzle — ce qui active les policies RLS PostgreSQL.

## Stack

- **Keycloak 24+** avec feature **Organizations** activée
- Realm unique : `schoolapp`
- Une école = une Keycloak Organization
- JWT signé RS256, clé publique via JWKS (cache 1h avec `jose` + `createRemoteJWKSet`)
- **Redis** : cache `org_id → school_id` (TTL 5 min)
- **NestJS** : `KeycloakAuthGuard` + `TenantInterceptor` + `@CurrentTenant()` decorator
- **Drizzle** : `tenantContext` (AsyncLocalStorage) + `tenantDB()` wrapper (voir skill `drizzle-schema`)

## Flux d'authentification

1. Utilisateur → Keycloak OIDC (login ou SSO)
2. Keycloak retourne JWT avec claim `organizations: [{ id, roles: [...] }]`
3. `KeycloakAuthGuard` valide la signature via JWKS cache et attache `req.user`
4. `TenantInterceptor` résout `org_id → school_id` (Redis → fallback `adminDB()`) et démarre `tenantContext.run({schoolId, userId, role}, next)`
5. Le service appelle `tenantDB(async (tx) => ...)` qui lit `tenantContext.getStore()` et émet `SET LOCAL app.current_school_id`, `app.current_user_id`, `app.current_role` dans la transaction
6. PostgreSQL applique les policies RLS, filtrage automatique

## KeycloakAuthGuard

```ts
// apps/api/src/modules/auth/keycloak.guard.ts
import { Injectable, CanActivate, ExecutionContext, UnauthorizedException } from '@nestjs/common';
import { jwtVerify, createRemoteJWKSet } from 'jose';

@Injectable()
export class KeycloakGuard implements CanActivate {
  private readonly jwks = createRemoteJWKSet(
    new URL(`${process.env.KEYCLOAK_URL}/realms/schoolapp/protocol/openid-connect/certs`),
    { cacheMaxAge: 60 * 60 * 1000 }, // 1h
  );

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const auth = req.headers.authorization;
    if (!auth?.startsWith('Bearer ')) throw new UnauthorizedException('Missing bearer');

    try {
      const { payload } = await jwtVerify(auth.slice(7), this.jwks, {
        issuer: `${process.env.KEYCLOAK_URL}/realms/schoolapp`,
        audience: 'schoolapp-api',
      });
      if (!payload.organizations || !Array.isArray(payload.organizations) || payload.organizations.length === 0) {
        throw new UnauthorizedException('No organization claim');
      }
      req.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

## TenantInterceptor

```ts
// apps/api/src/modules/auth/tenant.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, ForbiddenException } from '@nestjs/common';
import { Observable, defer } from 'rxjs';
import { tenantContext } from '../../db';
import { TenantResolver } from './tenant.resolver';

@Injectable()
export class TenantInterceptor implements NestInterceptor {
  constructor(private readonly resolver: TenantResolver) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return defer(async () => {
      const req = ctx.switchToHttp().getRequest();
      const { sub, organizations } = req.user;

      // Pour multi-org (super_admin), supporte un header explicite X-School-Id
      const requestedOrgId = req.headers['x-school-id'] || organizations[0].id;
      const org = organizations.find((o: { id: string }) => o.id === requestedOrgId);
      if (!org) throw new ForbiddenException('User does not belong to this organization');

      const schoolId = await this.resolver.resolve(org.id);
      const role = org.roles[0] as 'super_admin' | 'director' | 'teacher' | 'parent' | 'student';

      return new Promise((resolve, reject) => {
        tenantContext.run({ schoolId, userId: sub, role }, () => {
          next.handle().toPromise().then(resolve).catch(reject);
        });
      });
    });
  }
}
```

## TenantResolver avec cache Redis

```ts
// apps/api/src/modules/auth/tenant.resolver.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { Redis } from 'ioredis';
import { adminDB } from '../../db';
import { schools } from '../../db/schema/schools';
import { eq } from 'drizzle-orm';

@Injectable()
export class TenantResolver {
  constructor(private readonly redis: Redis) {}

  async resolve(orgId: string): Promise<string> {
    const cacheKey = `org:${orgId}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return cached;

    // Fallback DB — utilise adminDB car on n'a PAS encore de contexte tenant
    const [row] = await adminDB(async (tx) =>
      tx.select({ id: schools.id }).from(schools).where(eq(schools.keycloakOrgId, orgId)).limit(1),
    );
    if (!row) throw new NotFoundException(`School not found for org ${orgId}`);

    await this.redis.setex(cacheKey, 300, row.id); // 5 min
    return row.id;
  }

  async invalidate(orgId: string) {
    await this.redis.del(`org:${orgId}`);
  }
}
```

**Règle critique** : `TenantResolver` est la **seule** utilisation légitime de `adminDB()` dans un flow utilisateur — parce que la résolution elle-même a besoin d'interroger la table `schools` avant d'avoir un contexte tenant. Aucun autre service ne doit utiliser `adminDB()` dans le chemin de requête.

## Décorateur `@CurrentTenant()`

```ts
// apps/api/src/modules/auth/current-tenant.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { tenantContext } from '../../db';

export const CurrentTenant = createParamDecorator(
  (_data: unknown, _ctx: ExecutionContext): string => {
    const store = tenantContext.getStore();
    if (!store) throw new Error('CurrentTenant used outside tenant context');
    return store.schoolId;
  },
);

export const CurrentUser = createParamDecorator(
  (_data: unknown, _ctx: ExecutionContext): { userId: string; role: string } => {
    const store = tenantContext.getStore();
    if (!store) throw new Error('CurrentUser used outside tenant context');
    return { userId: store.userId, role: store.role };
  },
);
```

## Onboarder une école (provisioning)

Flow côté admin uniquement, via `adminDB()` et Keycloak Admin API :

1. `POST ${KC}/admin/realms/schoolapp/organizations` avec `{ name, alias: slug, domains: [...] }` → récupère `org.id`
2. Créer l'école en DB : `adminDB(tx => tx.insert(schools).values({ name, slug, keycloakOrgId: org.id }))`
3. Créer les rôles d'organisation dans Keycloak : `director`, `teacher`, `parent`, `student` (via Admin API Organizations)
4. Créer l'utilisateur director initial + l'ajouter comme member avec rôle `director`
5. Invalider cache Redis `org:<org_id>` via `tenantResolver.invalidate(org.id)` (prévention si clé zombie)

## Conventions

- Toujours valider la signature JWT via JWKS, jamais lire `kid` manuellement
- Cache JWKS 1h, cache `org_id → school_id` 5 min
- `SET LOCAL` uniquement (fait automatiquement par `tenantDB`)
- Le `sub` du JWT = `users.keycloakUserId` en DB (colonne unique)
- Jamais de tenant dans l'URL — toujours dérivé du JWT
- Support multi-org via header `X-School-Id` réservé au rôle `super_admin`

## Pièges à éviter

- **Oublier `SET LOCAL app.current_user_id`** : les policies restrictives par rôle (parent/teacher) échouent silencieusement → fuite. `tenantDB()` le fait pour toi, mais vérifie que tu utilises bien `tenantDB()` partout.
- **Ne pas invalider Redis** après changement d'org ou rôle → stale cache, mauvaise résolution.
- **Utiliser `SET` au lieu de `SET LOCAL`** → fuite entre requêtes dans le pool pg. `tenantDB()` utilise `SET LOCAL` (ouf).
- **Trust aveugle du claim `organizations`** sans vérification → un user peut forger `organizations[0]` pour accéder à une autre école. La vérif se fait dans `TenantInterceptor` : `organizations.find((o) => o.id === requestedOrgId)`.
- **Ne pas gérer multi-org** pour `super_admin` → impossible de switcher entre écoles dans l'UI admin.
- **Appeler `tenantDB()` dans `TenantResolver`** → boucle infinie (pas de contexte → échec). Toujours `adminDB()` là.
- **Décorateur `@CurrentTenant()` utilisé dans un controller sans `TenantInterceptor`** → throw "outside tenant context". Vérifier l'enregistrement global de l'interceptor dans `AppModule`.
- **Register `TenantInterceptor` par contrôleur** au lieu du global → risque d'oubli. Register en global via `APP_INTERCEPTOR`.

## Livrable

`KeycloakGuard`, `TenantInterceptor`, `TenantResolver`, décorateurs `@CurrentTenant` / `@CurrentUser` fonctionnels, registered en global dans `AppModule`, couverts par tests e2e avec **deux tenants distincts**, aucune fuite cross-tenant observée dans les tests.
