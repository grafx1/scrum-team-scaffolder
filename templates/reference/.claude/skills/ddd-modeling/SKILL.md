---
name: ddd-modeling
description: Conventions Domain-Driven Design — découpage en bounded contexts, agrégats, entités, value objects, domain events, repositories, application services, ubiquitous language. À déclencher quand un agent modélise une feature métier, crée un module, découpe un domain, définit des relations entre entités, mentionne "agrégat", "bounded context", "domain event", "value object", "repository", "ubiquitous language", ou conçoit un ADR de modélisation.
---

# ddd-modeling

## Pourquoi DDD

DDD aide à nommer les choses correctement, découper proprement (un module = un bounded context = une responsabilité métier), protéger les invariants (un agrégat valide ses propres règles), et faire évoluer sans casser (les domain events découplent les modules).

Ce skill impose le **vocabulaire** et les **patterns de découpage**, pas une implémentation DDD puriste (pas d'event sourcing obligatoire, pas de CQRS systématique).

## Bounded contexts

Chaque bounded context correspond à un **module backend** et à une zone de schéma DB. Un module ne doit importer que les **services exportés** d'un autre module, jamais ses tables directement.

**Exception** : les vues SQL cross-context sont acceptables pour les lectures analytiques (dashboard), mais toute **écriture** cross-context passe par un domain event.

L'architect doit documenter les bounded contexts du projet dans un ADR dédié au démarrage.

## Ubiquitous language (glossaire)

Le code, les tests, les ADR, les tâches et les design briefs utilisent les mêmes termes. L'architect crée et maintient un glossaire dans `docs/glossary.md` dès le sprint 0.

Format du glossaire :

| Terme métier | Terme technique | À ne pas utiliser |
|---|---|---|
| <concept du domaine> | `<nom_code>` | ~~<alternatives confuses>~~ |

**Règle** : si tu hésites sur un nom de table, variable, ou endpoint, vérifie le glossaire. Si le terme n'y est pas, propose-le dans un ADR ou dans le rapport de ton sprint.

## Agrégat — règles de design

Un **agrégat** est un groupe d'entités qui changent toujours ensemble et garantissent leurs propres invariants. L'agrégat root est la seule entité accessible de l'extérieur.

### Règles strictes

1. **Un agrégat = une transaction.** Jamais de transaction qui modifie deux agrégats différents. Pour la cohérence cross-agrégat, utiliser un domain event.
2. **L'agrégat root est le point d'entrée.** Pour modifier une entité enfant, passer par le service de l'agrégat root.
3. **Les invariants sont dans le service de l'agrégat, pas dans le controller.** Le controller valide le format, le service valide le métier.
4. **Préférer les petits agrégats.** Viser 1-3 tables par agrégat. Un agrégat de 10 tables = mauvais découpage.

## Value Objects

Un value object est un objet sans identité propre, défini par ses attributs, immuable. Exemples génériques :

```typescript
// domain/value-objects/money.ts
export class Money {
  private constructor(readonly amount: number, readonly currency: string) {}
  static of(amount: number, currency: string): Money {
    if (!Number.isInteger(amount) || amount < 0) throw new Error(`Invalid amount: ${amount}`);
    return new Money(amount, currency);
  }
  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }
}

// domain/value-objects/email-address.ts
export class EmailAddress {
  private constructor(readonly value: string) {}
  static from(raw: string): EmailAddress {
    const normalized = raw.trim().toLowerCase();
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(normalized)) throw new Error(`Invalid email: ${raw}`);
    return new EmailAddress(normalized);
  }
}
```

**Règle** : un value object a une factory statique qui valide à la construction. Ne pas sur-abstraire — `firstName: string` n'a pas besoin d'être un value object.

## Domain Events

Un domain event exprime un fait qui s'est produit dans un bounded context et qui intéresse d'autres contextes.

### Conventions

- Nommage : `<Entity><PastParticiple>Event` (ex : `OrderPlacedEvent`, `PaymentReceivedEvent`)
- Émission **après** l'écriture réussie (pas avant)
- Payload : les **ids** seulement, pas les entités entières
- Transport via le queue system du projet (pas in-memory — survit aux restarts)
- Le consommateur traite de manière **idempotente**

```typescript
// Émission
async placeOrder(dto: PlaceOrderDto, tenantId: string) {
  const order = await tenantDB(async (tx) => {
    // ... insert order ...
    return created;
  });
  await this.orderQueue.add('order-placed', { tenantId, orderId: order.id, placedAt: new Date() });
  return order;
}

// Consommation dans un autre bounded context
@Process('order-placed')
async handleOrderPlaced(job: Job<OrderPlacedPayload>) {
  return tenantContext.run({ tenantId: job.data.tenantId, ... }, async () => {
    // envoyer notification, mettre à jour stock, etc.
  });
}
```

## Anti-aggregate patterns (le code-reviewer rejette)

1. **God aggregate** : un service qui importe 5 tables de 3 contextes → découper
2. **Anemic domain model** : toute la logique dans le controller, le service ne fait que CRUD → déplacer les règles dans le service
3. **Cross-context write** : le service A fait directement `tx.update(tableDeB)` → émettre un domain event
4. **Validation dans le controller** : le controller fait des vérifications métier → déplacer dans le service
5. **Over-engineering** : créer des value objects pour `firstName` ou un event pour chaque CRUD → pragmatisme

## Checklists par agent

### Architect
- [ ] Bounded contexts identifiés et documentés (ADR)
- [ ] Glossaire ubiquitous language dans `docs/glossary.md`
- [ ] Agrégat root identifié pour chaque module complexe
- [ ] Domain events cross-context listés (émetteur → consommateur)

### Backend-dev
- [ ] Invariants dans le service, pas le controller
- [ ] Pas d'import de tables d'un autre bounded context
- [ ] Domain events émis après écriture réussie, via le queue system
- [ ] Glossaire respecté dans le nommage

### Code-reviewer
- [ ] Aucune transaction cross-agrégat
- [ ] Aucun import cross-context de tables (seulement des services exportés)
- [ ] Glossaire respecté dans les noms
- [ ] Invariants dans les services, pas les controllers
