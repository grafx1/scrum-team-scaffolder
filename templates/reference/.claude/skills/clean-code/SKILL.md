---
name: clean-code
description: Conventions Clean Code — nommage intentionnel, fonctions courtes, responsabilité unique, pas de code mort, pas de commentaires redondants, gestion d'erreurs explicite, DRY sans sur-abstraction. À déclencher quand un agent écrit ou review du code, reçoit un rejet pour lisibilité, mentionne "clean code", "refactor", "code smell", "dette technique", "naming", "readability".
---

# clean-code

## Règles non-négociables

**Nommage** : noms qui révèlent l'intention. Booléens : `is/has/can/should`. Fonctions : verbes. Classes : noms. Zéro abréviation cryptique. Constantes : `SCREAMING_SNAKE_CASE`. Fichiers : `kebab-case`.

**Fonctions** : une seule responsabilité. Maximum 20 lignes. Maximum 3 paramètres (au-delà → objet nommé). Pas de paramètre booléen. Early return plutôt que nesting.

**Types** : zéro `any`. Zéro cast non justifié. Strict mode activé. Inférence plutôt qu'annotation redondante.

**Erreurs** : jamais de `catch` vide. Jamais de `console.log` — logger du framework. Erreurs métier = exceptions nommées.

**DRY** : règle de trois (extraire à la 3ème occurrence). DRY au niveau sémantique, pas textuel. Pas de `utils.ts` fourre-tout.

**Fichiers** : une responsabilité par fichier. Maximum 200 lignes. Imports ordonnés : framework → tiers → internes → relatifs.

## Do / Don't

| Do | Don't |
|---|---|
| `getStudentsBySchool` | `getData`, `fetchStuff` |
| `isActive`, `hasPermission` | `active`, `perm` |
| `if (!entity) throw new NotFoundException()` | `if (entity) { if (entity.isActive) { ... } }` |
| `// WHY: RLS fail-safe retourne NULL si pas de contexte` | `// get all students` |
| `logger.error('create failed', { error, schoolId })` | `console.log(error)` |

## Grep checks (code-reviewer)

```bash
# Variables d'une lettre
rg '\b(let|const|var)\s+[a-z]\s*=' --glob '!*.spec.*' --glob '!*.test.*'
# any TypeScript
rg '\bany\b' --glob '!*.d.ts' -c
# console.log
rg 'console\.(log|error|warn|info|debug)' --glob '!*.spec.*' --glob '!*.test.*'
# Code commenté
rg '^\s*//\s*(const |let |var |import |return |if |for |await )' --glob '!*.spec.*'
# TODO sans ticket
rg 'TODO(?!:.*[A-Z]+-\d)'
```

## Anti-patterns

- ❌ Paramètre boolean dans une fonction — créer deux fonctions explicites
- ❌ Catch vide ou `catch (e) { console.log(e) }` sans rethrow
- ❌ Fichier `utils/helpers.ts` avec > 5 fonctions non liées
- ❌ Type `any` ou `as unknown as X` sans commentaire `// HACK:`
- ❌ Code commenté — git log conserve l'historique
- ❌ Duplication à la 3ème occurrence sans extraction

## Checklist avant `in_review`

- [ ] Nommage intentionnel, zéro abréviation cryptique
- [ ] Fonctions ≤ 20 lignes, ≤ 3 params, une responsabilité
- [ ] Zéro `any`, strict mode actif
- [ ] Zéro `console.log`, zéro code commenté, zéro variable morte
- [ ] Erreurs : catch avec log structuré ou rethrow, jamais vide
- [ ] Grep checks passent sans résultat inattendu
- [ ] `pnpm lint && pnpm typecheck` sans warnings
