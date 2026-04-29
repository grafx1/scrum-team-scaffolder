---
name: clean-code
description: Conventions Clean Code — nommage intentionnel, fonctions courtes, responsabilité unique, pas de code mort, pas de commentaires redondants, gestion d'erreurs explicite, DRY sans sur-abstraction. À déclencher quand un agent écrit ou review du code, reçoit un rejet pour lisibilité, mentionne "clean code", "refactor", "code smell", "dette technique", "naming", "readability".
---

# clean-code

## 1. Nommage

- **Les noms révèlent l'intention.** `getItemsByTenant` vs ~~`getData`~~
- **Boolean** : `is`, `has`, `can`, `should` — `isActive`, `hasPermission`, `canEdit`
- **Fonctions** : verbes — `createItem`, `sendNotification`, `validateInput`
- **Classes/types** : noms — `Order`, `PaymentService`, `NotificationProcessor`
- **Pas d'abréviations** sauf conventions universelles (`id`, `url`, `dto`, `tx`)
- **Constantes** : `SCREAMING_SNAKE_CASE`
- **Fichiers** : `kebab-case`

## 2. Fonctions

- **Une fonction fait UNE chose.** Si tu peux écrire "elle fait X **et** Y", c'est deux fonctions.
- **Maximum 20 lignes** (hors imports et types). Au-delà, extraire.
- **Maximum 3 paramètres.** Au-delà, regrouper dans un objet nommé.
- **Pas de paramètres boolean** (flag arguments). Préférer deux fonctions explicites.
- **Early return** plutôt que nesting profond :
  ```
  if (!entity) throw new NotFoundException();
  if (!entity.isActive) throw new ForbiddenException();
  return entity;
  ```

## 3. Commentaires

- **Autorisés** : `// WHY: <raison>`, `// TODO: <description> — <ticket>`, `// HACK: <explication>`, JSDoc sur interfaces publiques
- **Interdits** : paraphrase du code, `// Added by <name> on <date>`, code commenté, séparateurs visuels

## 4. Gestion d'erreurs

- **Jamais de catch vide.** Log structuré ou rethrow avec contexte.
- **Erreurs métier** = exceptions nommées (HttpException, custom Error subclass)
- **Pas de string d'erreur dans les comparaisons** : utiliser `instanceof`
- **Pas de `console.log`** — toujours le logger du framework avec contexte structuré

## 5. Types (langages typés)

- **Zéro `any`** (TypeScript) / **zéro `type: ignore`** (Python)
- **Zéro assertion de cast** non justifiée
- **Strict mode** activé
- **Inférence plutôt qu'annotation redondante**
- **Unions discriminées** plutôt que type + cast

## 6. DRY sans sur-abstraction

- **Règle de trois** : duplication acceptable si 2 occurrences, extraire à la 3ème
- **DRY au niveau logique, pas au niveau texte.** Deux fonctions avec le même if/else mais des sémantiques métier différentes ne doivent PAS être fusionnées.
- **Pas de fichier utilitaire fourre-tout** (`utils/helpers.ts` de 500 lignes → découper par domaine)
- **Schémas de validation partagés** via package commun, pas redéfinis dans chaque module

## 7. Structure de fichier

- **Un fichier = une responsabilité.** Controller + service + DTO dans le même fichier → 3 fichiers.
- **Maximum 200 lignes par fichier.** Au-delà, découper.
- **Imports ordonnés** : framework → tiers → internes absolus → relatifs

## 8. Code mort

- Pas de code commenté (git l'a en mémoire)
- Pas de fonctions non-appelées
- Pas de variables assignées jamais lues
- Pas de fichiers orphelins (modules non-importés)

## Grep checks du code-reviewer

```bash
# Variables d'une lettre
rg '\b(let|const|var)\s+[a-z]\s*=' <files> --glob '!*.spec.*' --glob '!*.test.*'

# any (TypeScript)
rg '\bany\b' <files> --glob '!*.d.ts' -c

# console.log/error
rg 'console\.(log|error|warn|info|debug)' <files> --glob '!*.spec.*' --glob '!*.test.*'

# Code commenté
rg '^\s*//\s*(const |let |var |import |export |return |if |for |while |await )' <files> --glob '!*.spec.*'

# TODO sans ticket
rg 'TODO(?!:.*[A-Z]+-\d)' <files>
```

## Checklist avant `in_review`

- [ ] Nommage intentionnel, pas d'abréviations cryptiques
- [ ] Fonctions ≤ 20 lignes, ≤ 3 params, une seule responsabilité
- [ ] Commentaires WHY/TODO(ticket)/HACK uniquement, zéro code commenté
- [ ] Erreurs : catch avec log ou rethrow, jamais vide
- [ ] Types : zéro `any`, strict mode
- [ ] DRY appliqué à la 3ème occurrence
- [ ] Fichiers ≤ 200 lignes, imports ordonnés
- [ ] Zéro code mort, zéro `console.log`
- [ ] Lint + typecheck sans warnings
