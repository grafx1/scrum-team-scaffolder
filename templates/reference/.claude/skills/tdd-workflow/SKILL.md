---
name: tdd-workflow
description: Workflow Test-Driven Development — cycle Red/Green/Refactor, écriture du test AVANT le code, structure des tests par couche (unit, integration, e2e), conventions de nommage, patterns de mock/stub, fixtures multi-tenant. À déclencher quand un agent écrit un test, implémente une feature, mentionne "TDD", "test first", "coverage", "red/green/refactor", "test d'isolation", ou reçoit un rejet pour manque de tests.
---

# tdd-workflow

> **Non-négociable.** Le TDD est impératif sur ce projet. Aucune ligne de code d'implémentation sans un test qui échoue d'abord. Pas d'exception, pas de dérogation. Écrire le code avant le test est un motif de rejet automatique en review.

## Phases

1. **Red** — écrire un test qui échoue décrivant le comportement attendu via l'interface publique
2. **Green** — écrire le minimum de code pour que le test passe, rien de plus
3. **Refactor** — éliminer la duplication, tous les tests restent verts
4. **Répéter** — un comportement à la fois, jamais deux tests en attente simultanément

## Structure par couche

| Couche | Ce qu'on teste | DB | Quand |
|---|---|---|---|
| **Unit** | Logique métier du service | Mockée | Chaque méthode publique |
| **Integration** | Service + vraie DB | Réelle | Chaque nouvelle table ou policy |
| **E2E** | HTTP → Controller → DB | Réelle | Chaque endpoint |
| **Isolation** | Tenant A invisible de Tenant B | Réelle | Chaque entité tenant (obligatoire) |

Frontend : Unit (rendu + états) et Integration (composant + msw) uniquement.

## Convention de nommage

```typescript
describe('StudentsService.create', () => {
  it('should return created student when dto is valid', async () => { ... });
});
```

`describe` = classe + méthode. `it` = `should <comportement> when <condition>`. Toujours `it()`, jamais `test()`.

## Fixture multi-tenant (copier tel quel)

```typescript
export const TENANT_A = { schoolId: 'uuid-a', userId: 'uuid-user-a', role: 'director' as const };
export const TENANT_B = { schoolId: 'uuid-b', userId: 'uuid-user-b', role: 'director' as const };
export const runAs = <T>(t: typeof TENANT_A, fn: () => Promise<T>) => tenantContext.run(t, fn);

it('should not expose tenant A data to tenant B', async () => {
  const created = await runAs(TENANT_A, () => service.create(dto, TENANT_A.schoolId));
  const fromB = await runAs(TENANT_B, () => service.findAll());
  expect(fromB.find((e) => e.id === created.id)).toBeUndefined();
});
```

## Anti-patterns

- ❌ `expect(true).toBe(true)` — test vide
- ❌ `expect(result).toBeDefined()` seul — ne vérifie rien de métier
- ❌ Mock de tous les collaborateurs internes — on teste des mocks, pas du code
- ❌ Appel de méthode privée — signe que l'interface publique est mal conçue
- ❌ Nom sans comportement — `"should work"`, `"test 1"`, `"creates student"`
- ❌ 0 test d'isolation sur une entité tenant — rejet automatique en review

## Checklist avant `in_review`

- [ ] Tests écrits avant le code
- [ ] 1 test par acceptance_criterion minimum
- [ ] Nommage `should … when …` sur tous les `it()`
- [ ] Test d'isolation 2 tenants présent si entité multi-tenant
- [ ] 5 états UI couverts si tâche frontend (loading / empty / error / offline / success)
- [ ] Aucun appel réseau réel en CI (mocks ou msw)
- [ ] Suite complète verte, zéro `.skip`
- [ ] `afterEach` / `afterAll` cleanup présent
