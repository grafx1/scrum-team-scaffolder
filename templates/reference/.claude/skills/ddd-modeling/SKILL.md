---
name: ddd-modeling
description: Conventions Domain-Driven Design — découpage en bounded contexts, agrégats, entités, value objects, domain events, repositories, application services, ubiquitous language. À déclencher quand un agent modélise une feature métier, crée un module, découpe un domain, définit des relations entre entités, mentionne "agrégat", "bounded context", "domain event", "value object", "repository", "ubiquitous language", ou conçoit un ADR de modélisation.
---

# ddd-modeling

## Phases

1. **Bounded contexts** — identifier les frontières métier, documenter dans un ADR, un module = un context
2. **Ubiquitous language** — créer/mettre à jour `docs/glossary.md`, aligner code + tests + tâches sur ce vocabulaire
3. **Agrégats** — identifier l'agrégat root, invariants dans le service, jamais dans le controller
4. **Domain events** — émettre après écriture réussie, via queue system, payload = IDs uniquement
5. **Value objects** — uniquement si validation non-triviale (ex: `Money`, `EmailAddress`) ; factory statique qui valide à la construction

## Règles structurantes

**Bounded context** : un module n'importe que les services exportés d'un autre module, jamais ses tables directement. Exception : vues SQL cross-context acceptables en lecture analytique uniquement.

**Agrégat** : une transaction = un agrégat. Pour la cohérence cross-agrégat → domain event. Viser 1-3 tables par agrégat.

**Domain event** — nommage `<Entity><PastParticiple>Event`, émission après commit, consommateur idempotent :
```typescript
// Émission (après insert réussi)
await this.queue.add('order-placed', { tenantId, orderId: order.id });

// Consommation dans un autre context
@Process('order-placed')
async handle(job: Job<{ tenantId: string; orderId: string }>) {
  return tenantContext.run({ tenantId: job.data.tenantId, ... }, async () => { /* ... */ });
}
```

## Anti-patterns

- ❌ **God aggregate** — service qui importe 5 tables de 3 contextes différents
- ❌ **Anemic model** — toute la logique dans le controller, service = CRUD pur
- ❌ **Cross-context write** — `service A` fait `tx.update(tableDeB)` directement
- ❌ **Validation métier dans le controller** — le controller valide le format, le service valide le métier
- ❌ **Over-engineering** — value object pour `firstName`, domain event pour chaque CRUD

## Checklist avant `in_review`

- [ ] Bounded contexts documentés dans un ADR (architect)
- [ ] Glossaire `docs/glossary.md` à jour, termes respectés dans le code et les tests
- [ ] Invariants métier dans le service, zéro règle métier dans le controller
- [ ] Aucun import de table d'un autre bounded context (seulement les services exportés)
- [ ] Domain events émis après écriture réussie, via queue, payload = IDs
- [ ] Consommateurs de domain events idempotents
