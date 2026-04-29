---
name: tdd-workflow
description: Workflow Test-Driven Development — cycle Red/Green/Refactor, écriture du test AVANT le code, structure des tests par couche (unit, integration, e2e), conventions de nommage, patterns de mock/stub, fixtures multi-tenant. À déclencher quand un agent écrit un test, implémente une feature, mentionne "TDD", "test first", "coverage", "red/green/refactor", "test d'isolation", ou reçoit un rejet pour manque de tests.
---

# tdd-workflow

## Principe fondamental

**On écrit le test qui échoue AVANT d'écrire le code qui le fait passer.** Le cycle :

1. **Red** — écrire un test qui échoue (le test exprime le comportement attendu)
2. **Green** — écrire le **minimum** de code pour que le test passe
3. **Refactor** — nettoyer sans changer le comportement (tous les tests restent verts)
4. Répéter

## Structure de test par couche

### Backend

| Couche | Ce qu'on teste | DB | Auth | Quand l'écrire |
|---|---|---|---|---|
| **Unit** | Logique métier du service en isolation | Mockée | Mockée | Chaque méthode publique du service |
| **Integration** | Service + vraie DB | Vraie DB | Mockée | Chaque nouvelle table ou policy |
| **E2E** | HTTP → Controller → Service → DB → réponse | Vraie DB | Token de test | Chaque endpoint |
| **Isolation** | Tenant A ne voit pas les données de tenant B | Vraie DB | 2 tokens distincts | Chaque nouvelle entité tenant (obligatoire si multi-tenant) |

### Frontend

| Couche | Ce qu'on teste | API | Quand l'écrire |
|---|---|---|---|
| **Unit** | Rendu, interactions, états (loading/empty/error/offline/success) | Mockée | Chaque composant avec logique |
| **Integration** | Composant + data fetching + mutations | Mockée (msw) | Chaque page ou flow |

## Conventions de nommage

```
describe('<Service>.<method>', () => {
  it('should <comportement> when <condition>', async () => { ... });
});
```

- `describe` = classe + méthode
- `it` = `should` + comportement
- Un `it` = un seul assert (ou groupe cohérent)
- Toujours `it()`, jamais `test()`

## Fixtures multi-tenant (si applicable)

```typescript
export const TENANT_A = { tenantId: 'uuid-a', userId: 'uuid-user-a', role: 'admin' };
export const TENANT_B = { tenantId: 'uuid-b', userId: 'uuid-user-b', role: 'admin' };

export function runAsTenant<T>(tenant: typeof TENANT_A, fn: () => Promise<T>) {
  return tenantContext.run(tenant, fn);
}
```

**Test d'isolation obligatoire** :
```typescript
it('should not return tenant A entities when querying as tenant B', async () => {
  const created = await runAsTenant(TENANT_A, () => service.create({ name: 'Test' }, TENANT_A.tenantId));
  const fromB = await runAsTenant(TENANT_B, () => service.findAll());
  expect(fromB.find((e) => e.id === created.id)).toBeUndefined();
});
```

## Indicateurs de mauvais tests (rejetés par le code-reviewer)

- `expect(true).toBe(true)` — test vide
- `expect(result).toBeDefined()` seul — ne vérifie rien
- Test qui mock tout et ne teste que des mocks
- Test qui appelle une méthode privée
- Nom sans comportement ("should work", "test 1")
- Dépendance inter-tests (ordre d'exécution)
- Test flaky, test sans cleanup
- 0 test d'isolation sur une entité tenant (si multi-tenant)

## Checklist avant `in_review`

- [ ] Tests écrits avant le code (pas après)
- [ ] Au moins 1 test par acceptance_criterion
- [ ] Nommage `should ... when ...`
- [ ] Test d'isolation tenant si entité multi-tenant
- [ ] 5 états frontend si tâche frontend
- [ ] Mock des APIs externes (jamais d'appel réseau en CI)
- [ ] Suite complète verte, zéro skip
- [ ] Cleanup propre (afterEach / afterAll)
