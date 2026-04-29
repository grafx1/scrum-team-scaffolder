---
name: clean-architecture
description: Conventions d'architecture en couches — séparation domain/application/infrastructure, dépendances unidirectionnelles, injection de dépendances, ports & adapters. À déclencher quand un agent structure un module, crée un service, définit un repository, arbitre où placer une logique, mentionne "clean architecture", "hexagonal", "ports adapters", "couche domain", "dépendance inversée", ou reçoit un rejet pour couplage excessif.
---

# clean-architecture

## 3 principes non-négociables

1. **La logique métier ne dépend pas du framework.** Le service ne connaît pas HTTP, ni le queue system. Il reçoit des paramètres typés, applique des règles, retourne un résultat.
2. **Les dépendances pointent vers l'intérieur.** Controller → Service → Domain → (rien). Jamais l'inverse.
3. **L'infrastructure est interchangeable.** Changer d'ORM, de queue system, ou de provider externe ne modifie que la couche infra, pas les services métier.

## Architecture en couches

```
┌─────────────────────────────────────────────────────┐
│  Couche Infrastructure (Adapters)                   │
│  Controllers / Webhooks / Processors / Cron         │
├─────────────────────────────────────────────────────┤
│  Couche Application (Use Cases / Services)          │
│  Orchestration, coordination, transactions          │
├─────────────────────────────────────────────────────┤
│  Couche Domain (Entities / Value Objects / Rules)    │
│  Logique métier pure, invariants                    │
├─────────────────────────────────────────────────────┤
│  Couche Infrastructure (Adapters)                   │
│  ORM, Redis, Storage, APIs externes                 │
└─────────────────────────────────────────────────────┘

Sens des dépendances : de l'extérieur vers l'intérieur ↓↓↓
```

### Mapping sur la structure de fichiers

```
modules/<feature>/
├── <feature>.controller.ts      ← Infrastructure (HTTP adapter)
├── <feature>.service.ts         ← Application (orchestrateur)
├── domain/                      ← Domain (si module complexe)
│   ├── rules/                  (fonctions pures de validation)
│   ├── value-objects/          (immuables, pas de dépendance framework)
│   └── events/                 (classes de données pures)
├── dto/                         ← Contrat d'entrée/sortie
├── jobs/                        ← Infrastructure (queue adapter)
└── __tests__/
```

**Pragmatisme** : le dossier `domain/` n'est obligatoire que pour les modules à logique métier complexe (≥ 3 règles métier). Les modules CRUD simples n'ont pas besoin de cette structuration.

## Règle de dépendance

### Autorisé

- Controller importe Service
- Service importe le wrapper DB (ex : `tenantDB`)
- Service importe un Service d'un autre module (via export)
- Processor importe Service pour déléguer la logique
- Domain value object n'importe rien du framework

### Interdit

| Pattern interdit | Alternative |
|---|---|
| Service importe Controller | Le controller appelle le service |
| Service importe les tables d'un **autre** bounded context | Appeler le service exporté de l'autre module |
| Controller avec logique métier (if/else business) | Déplacer dans le service |
| Domain entity importe le framework | Classes TypeScript/Python/Ruby pures |
| Service fait directement `fetch('https://api.vendor.com')` | Déléguer au service d'intégration |
| Job contient la logique métier | Le job appelle le service après avoir rejoué le contexte |

## Test mental d'interchangeabilité

Pour vérifier la clean-ness : **"si je remplace l'ORM X par l'ORM Y, quels fichiers changent ?"**

| Couche | Change ? |
|---|---|
| Controllers | ❌ Non |
| Services | ⚠️ Légèrement (l'API du wrapper DB change) |
| Domain (rules, value objects, events) | ❌ Non |
| DTOs | ❌ Non |
| `db/` (schema, migrations, context) | ✅ Entièrement (c'est l'infra) |

Si la réponse est "tout change" → l'architecture n'est pas clean.

## Ce qu'un service fait / ne fait pas

**Fait** : recevoir des DTO validés, ouvrir une transaction, appliquer les règles métier, persister, émettre des events, retourner le résultat.

**Ne fait pas** : parser des headers HTTP, formater une réponse HTTP, connaître le framework web, appeler directement une API externe, logger avec des détails HTTP.

## Quand ne PAS appliquer

- Modules CRUD simples (< 3 règles métier) : controller + service + table. Pas de `domain/`.
- Scripts one-shot (seed, migration manuelle) : code procédural direct.
- Endpoints de health check, métriques : pas de logique métier.

**Signal pour ajouter la structure clean** : quand un module a > 3 règles métier ou > 2 intégrations externes → créer `domain/`.

## Grep checks du code-reviewer

```bash
# Controller qui contient des règles métier
rg 'throw new Bad(Request|Conflict)Exception' <controllers> | grep -v 'validation'

# Service qui importe des tables d'un autre bounded context
rg "from '.*modules/(?!$(basename $MODULE))" <service_files> | grep -v 'Service'

# Domain qui dépend du framework
rg "@nestjs|from 'express'|from 'fastapi'" modules/*/domain/

# Service qui fait du fetch direct
rg "fetch\(|axios\.|this\.http\." <service_files> | grep -v 'integration'
```

## Checklist code-reviewer

- [ ] Controller = adapteur HTTP pur (parsing, appel service, retour réponse)
- [ ] Service ne connaît pas HTTP (pas de `@Req()`, pas de `Response`)
- [ ] Règles métier complexes dans `domain/rules/` (fonctions pures)
- [ ] Imports cross-bounded-context passent par les services exportés, jamais les tables
- [ ] APIs externes dans des services d'intégration dédiés
- [ ] Jobs délèguent au service, pas de logique propre
- [ ] Si module complexe → `domain/` existe
- [ ] Si module CRUD simple → absence de `domain/` acceptée
