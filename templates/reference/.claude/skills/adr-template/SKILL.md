---
name: adr-template
description: Format standard des Architecture Decision Records (ADR) du projet. Utiliser dès que l'architect (ou n'importe quel agent) doit documenter une décision structurante — choix de stack, pattern de modélisation, stratégie de sync, résolution d'un trade-off technique. Définit l'emplacement, la nomenclature, la structure markdown et la numérotation des ADR. Sans ce skill, chaque ADR aurait un format différent et serait illisible à 6 mois.
---

# Architecture Decision Records

Un ADR documente **une décision structurante** : un choix qui sera coûteux à défaire. Les micro-décisions de code (nom de variable, ordre des imports) n'en sont pas.

## 1. Quand écrire un ADR

Oui :
- Choix d'une lib qui crée une dépendance forte (Drizzle vs Prisma, Dexie vs Watermelon)
- Pattern de modélisation qui va se répéter (RLS multi-tenant, SyncQueue)
- Contrat entre deux systèmes (interface partagée web/mobile)
- Trade-off explicite entre deux options viables
- Revirement d'une décision précédente (nouvel ADR qui supersede l'ancien)

Non :
- "J'ai mis un `useState` ici plutôt qu'un `useReducer`"
- "J'ai nommé la fonction `getStudents` plutôt que `fetchStudents`"

Règle : si un dev junior peut comprendre le choix en lisant le code, pas d'ADR.

## 2. Emplacement et nomenclature

```
docs/adr/
├── README.md                                 # index auto-généré
├── ADR-0001-monorepo-pnpm-workspaces.md
├── ADR-0002-drizzle-over-prisma.md
├── ADR-0003-rls-postgres-multi-tenant.md
├── ADR-0004-sync-queue-contract.md
└── ADR-0005-keycloak-organizations.md
```

**Numéro** : 4 chiffres, incrémental, **jamais** réutilisé (même si l'ADR est supersédé).
**Slug** : minuscules, tirets, court et descriptif.

## 3. Template obligatoire

```markdown
# ADR-NNNN : <Titre court et affirmatif>

- **Statut** : proposed | accepted | deprecated | superseded by ADR-XXXX
- **Date** : YYYY-MM-DD
- **Auteur** : <agent ou humain>
- **Tâches liées** : T-001, T-042

## Contexte

Pourquoi se pose-t-on la question ? Quel problème essaie-t-on de résoudre ?
Quelles sont les contraintes (techniques, business, légales, équipe) ?

2-5 phrases. Si tu as besoin de plus, c'est que tu décris plusieurs décisions — découpe en plusieurs ADR.

## Options considérées

### Option A : <nom>
- ✅ Avantages
- ❌ Inconvénients

### Option B : <nom>
- ✅ Avantages
- ❌ Inconvénients

### Option C : <nom>
- ✅ Avantages
- ❌ Inconvénients

## Décision

**Option retenue : <X>**

En 1-3 phrases : pourquoi celle-ci et pas les autres, étant donné les contraintes du Contexte.

## Conséquences

### Positives
- Conséquence concrète 1
- Conséquence concrète 2

### Négatives
- Ce qu'on accepte comme coût / dette
- Ce qu'on renonce à faire

### Actions déclenchées
- T-XXX : tâche créée pour implémenter cette décision
- Skill `<name>` à créer/mettre à jour
- ADR-YYYY à réviser

## Notes

Liens externes, benchmarks, discussions, captures, ce que tu veux.
```

## 4. Cycle de vie

| Statut | Sens |
|---|---|
| `proposed` | Écrit mais pas encore validé — ne pas implémenter |
| `accepted` | Validé, implémenté ou en cours — source de vérité |
| `deprecated` | Plus pertinent mais pas remplacé (cas rare) |
| `superseded by ADR-XXXX` | Remplacé — pointer vers le nouvel ADR |

**On ne supprime JAMAIS un ADR.** On change son statut. L'historique des décisions est aussi important que les décisions en vigueur.

## 5. Index `docs/adr/README.md`

À mettre à jour à chaque nouvel ADR :

```markdown
# Architecture Decision Records

| # | Titre | Statut | Date |
|---|---|---|---|
| [0001](./ADR-0001-monorepo-pnpm-workspaces.md) | Monorepo pnpm workspaces | accepted | 2026-04-09 |
| [0002](./ADR-0002-drizzle-over-prisma.md) | Drizzle ORM plutôt que Prisma | accepted | 2026-04-09 |
```

## 6. Lien avec le sprint et le backlog

Quand un ADR génère du travail :
1. L'architect crée l'ADR.
2. Dans la section "Actions déclenchées", il liste les tâches à créer.
3. **Si ces tâches sont réalisables dans le sprint courant** : l'architect les ajoute directement comme subtasks dans `scrum/sprintN.json` (via le même mécanisme que le lead-dev, avec des ids `T-XXX.1`, `T-XXX.2`…). Le lead-dev les assignera au prochain tour.
4. **Si les tâches dépassent le sprint courant** : les lister dans la section "Actions déclenchées" de l'ADR. Le scrum-master les remontera à l'humain dans son rapport de clôture de sprint pour qu'elles soient ajoutées à `tasks.json`.
5. Les `artifacts` de la tâche architecture pointent vers le chemin de l'ADR.

## 7. Longueur cible

- **Minimum viable** : 20 lignes (Contexte + Décision + 3 conséquences)
- **Cible idéale** : 40-80 lignes
- **Trop long** : > 150 lignes → c'est probablement 2 ADR déguisés en un

Un ADR doit être lisible en 2 minutes. S'il prend plus, le lecteur de dans 6 mois ne le lira pas.

## 8. Checklist avant commit

- [ ] Numéro incrémental correct
- [ ] Template respecté (tous les h2 présents)
- [ ] Au moins 2 options considérées (sinon ce n'est pas une décision, c'est une imposition)
- [ ] Conséquences **négatives** explicites — pas de décision sans coût
- [ ] Index `README.md` mis à jour
- [ ] Tâches dérivées ajoutées au sprint file (ou listées dans l'ADR pour un futur sprint)
