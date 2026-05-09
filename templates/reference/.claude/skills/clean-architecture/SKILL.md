---
name: clean-architecture
description: Conventions d'architecture en couches — séparation domain/application/infrastructure, dépendances unidirectionnelles, injection de dépendances, ports & adapters. À déclencher quand un agent structure un module, crée un service, définit un repository, arbitre où placer une logique, mentionne "clean architecture", "hexagonal", "ports adapters", "couche domain", "dépendance inversée", ou reçoit un rejet pour couplage excessif.
---

# clean-architecture

## Phases

1. **Évaluer la complexité** — < 3 règles métier → controller + service suffit, pas de `domain/`. ≥ 3 règles → créer `domain/rules/`.
2. **Placer la logique** — HTTP parsing dans le controller, orchestration dans le service, règles pures dans `domain/`.
3. **Vérifier les dépendances** — Controller → Service → DB. Jamais l'inverse. Jamais cross-context sur les tables.
4. **Vérifier l'interchangeabilité** — "si je change l'ORM, quels fichiers changent ?" → uniquement `db/`.

## Règles de dépendance

| Autorisé | Interdit |
|---|---|
| Controller importe Service | Service importe Controller |
| Service importe `tenantDB` | Controller contient du `if/else` métier |
| Service importe Service d'un autre module (via export) | Service importe tables d'un autre bounded context |
| Processor délègue au Service | Job contient la logique métier |
| Domain n'importe rien du framework | Domain entity dépend de NestJS / Express |

**Ce qu'un service fait** : recevoir des DTO validés, ouvrir une transaction, appliquer les règles, persister, émettre des events, retourner.

**Ce qu'un service ne fait pas** : parser des headers HTTP, formatter une réponse, appeler directement `fetch()` vers une API externe.

## Grep checks (code-reviewer)

```bash
# Règles métier dans le controller
rg 'throw new Bad(Request|Conflict)Exception' <controllers> | grep -v 'validation'
# Import cross-context de tables (pas de services)
rg "from '.*modules/(?!<current>)" <service_files> | grep -v 'Service'
# Domain dépendant du framework
rg "@nestjs|from 'express'" modules/*/domain/
# Fetch direct dans un service
rg "fetch\(|axios\.\|this\.http\." <service_files> | grep -v 'integration'
```

## Anti-patterns

- ❌ Controller avec `if/else` métier — déplacer dans le service
- ❌ Service qui importe `@Req()` ou `Response` de NestJS
- ❌ Service qui appelle `fetch()` directement — déléguer au service d'intégration
- ❌ `domain/` créé pour un module CRUD sans règle métier — sur-ingénierie
- ❌ Job processor avec logique propre — le job appelle le service après `tenantContext.run()`

## Checklist avant `in_review`

- [ ] Controller = adapteur HTTP pur (parsing + appel service + retour)
- [ ] Service sans référence à HTTP (`@Req`, `Response`, headers)
- [ ] Règles métier complexes dans `domain/rules/` si module ≥ 3 règles
- [ ] Imports cross-bounded-context via services exportés uniquement
- [ ] APIs externes dans des services d'intégration dédiés
- [ ] Grep checks passent sans résultat inattendu
