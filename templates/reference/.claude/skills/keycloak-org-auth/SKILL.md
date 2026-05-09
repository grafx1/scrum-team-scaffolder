---
name: keycloak-org-auth
description: Implémente l'authentification et le provisionning multi-tenant Keycloak 24 Organizations pour le SaaS Éducation Sénégal — realm schoolapp, mapping org_id ↔ schools.keycloak_org_id, claims JWT, résolution cache Redis, pose du contexte tenant via AsyncLocalStorage + tenantDB() Drizzle. À déclencher quand l'utilisateur dit "onboard une école", "auth multi-tenant", "KC Guard", "TenantInterceptor", "CurrentTenant decorator", ou touche aux flows OIDC Keycloak.
---

# keycloak-org-auth

## Phases

1. **Guard** — valider signature JWT via JWKS (cache 1h), attacher `req.user`
2. **Interceptor** — résoudre `org_id → school_id` (Redis → fallback `adminDB()`), démarrer `tenantContext.run()`
3. **Decorator** — lire `tenantContext.getStore()` dans `@CurrentTenant()` et `@CurrentUser()`
4. **Register global** — `APP_INTERCEPTOR` dans `AppModule`, jamais par contrôleur
5. **Onboarding** — créer Org Keycloak + row `schools` + rôles KC + director initial

## Flux

```
Request → KeycloakGuard (vérifie JWT, attache req.user)
        → TenantInterceptor (résout org_id → school_id, tenantContext.run)
        → Controller (@CurrentTenant() lit le store)
        → Service (tenantDB() lit le store, SET LOCAL → RLS actif)
```

## Implémentations de référence

**KeycloakGuard** :
```typescript
@Injectable()
export class KeycloakGuard implements CanActivate {
  private readonly jwks = createRemoteJWKSet(
    new URL(`${process.env.KEYCLOAK_URL}/realms/schoolapp/protocol/openid-connect/certs`),
    { cacheMaxAge: 60 * 60 * 1000 },
  );
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const auth = req.headers.authorization;
    if (!auth?.startsWith('Bearer ')) throw new UnauthorizedException('Missing bearer');
    const { payload } = await jwtVerify(auth.slice(7), this.jwks, {
      issuer: `${process.env.KEYCLOAK_URL}/realms/schoolapp`,
      audience: 'schoolapp-api',
    });
    if (!payload.organizations?.length) throw new UnauthorizedException('No organization claim');
    req.user = payload;
    return true;
  }
}
```

**TenantInterceptor** :
```typescript
intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
  return defer(async () => {
    const { sub, organizations } = ctx.switchToHttp().getRequest().user;
    const requestedOrgId = req.headers['x-school-id'] || organizations[0].id;
    const org = organizations.find((o: { id: string }) => o.id === requestedOrgId);
    if (!org) throw new ForbiddenException('User does not belong to this organization');
    const schoolId = await this.resolver.resolve(org.id);
    const role = org.roles[0] as TenantContext['role'];
    return new Promise((resolve, reject) => {
      tenantContext.run({ schoolId, userId: sub, role }, () =>
        next.handle().toPromise().then(resolve).catch(reject));
    });
  });
}
```

**TenantResolver** (Redis → adminDB fallback) :
```typescript
async resolve(orgId: string): Promise<string> {
  const cached = await this.redis.get(`org:${orgId}`);
  if (cached) return cached;
  const [row] = await adminDB(async (tx) =>
    tx.select({ id: schools.id }).from(schools).where(eq(schools.keycloakOrgId, orgId)).limit(1));
  if (!row) throw new NotFoundException(`School not found for org ${orgId}`);
  await this.redis.setex(`org:${orgId}`, 300, row.id);
  return row.id;
}
```

**Décorateurs** :
```typescript
export const CurrentTenant = createParamDecorator((_: unknown, _ctx: ExecutionContext): string => {
  const store = tenantContext.getStore();
  if (!store) throw new Error('CurrentTenant used outside tenant context');
  return store.schoolId;
});
```

## Anti-patterns

- ❌ `TenantInterceptor` enregistré par contrôleur — utiliser `APP_INTERCEPTOR` global uniquement
- ❌ `tenantDB()` dans `TenantResolver` — boucle infinie, utiliser `adminDB()` là uniquement
- ❌ `SET` au lieu de `SET LOCAL` — fuite entre requêtes dans le pool pg
- ❌ Trust du claim `organizations` sans vérifier que l'org demandée appartient à l'user
- ❌ Cache Redis non invalidé après changement d'org ou de rôle → stale tenant
- ❌ Tenant dans l'URL — toujours dérivé du JWT, jamais d'un paramètre de route

## Checklist avant `in_review`

- [ ] `KeycloakGuard` vérifie signature JWKS + claim `organizations` non vide
- [ ] `TenantInterceptor` vérifie que l'org demandée appartient bien à l'user
- [ ] `TenantResolver` : Redis (TTL 5 min) → fallback `adminDB()`, invalidation exposée
- [ ] `APP_INTERCEPTOR` global dans `AppModule`, pas par contrôleur
- [ ] `@CurrentTenant()` throw si appelé hors contexte tenant
- [ ] Tests e2e avec 2 tenants distincts, zéro fuite cross-tenant observable
