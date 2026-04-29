---
name: design-spec-template
description: Format obligatoire du design brief markdown produit par le frontend-dev avant d'écrire le code et évalué par le code-reviewer. Sert de grading criteria pour l'evaluator dans l'architecture Generator/Evaluator adversariale. À déclencher quand un agent mentionne "design brief", "spec d'écran", "docs/design/", "format de spec", ou commence une tâche frontend.
---

# design-spec-template

## Pourquoi

Le `frontend-dev` est le Generator (design + code), le `code-reviewer` est l'Evaluator adversarial. Pour que l'Evaluator fonctionne, il a besoin de **grading criteria explicites**. Ce skill définit le **format obligatoire** du brief et sert de checklist formelle en review.

## Emplacement

`docs/design/<slug>.md` — le chemin est ajouté au champ `artifacts[]` de la tâche dans `scrum/sprintN.json`.

## Format obligatoire (11 sections)

```markdown
# Design brief — <titre de la feature>

**Tâche** : T-XXX
**Plateforme(s)** : web | mobile | both
**Auteur** : frontend-dev
**Date** : YYYY-MM-DD

## 1. Contexte et objectif
<Problème utilisateur, rôle concerné, résultat attendu>

## 2. User flow
<Séquence d'écrans et d'actions, liste numérotée>

## 3. Wireframe ASCII
<Structure hiérarchique de chaque écran clé>

## 4. Les 5 états obligatoires
### 4.1 Loading
### 4.2 Empty
### 4.3 Error
### 4.4 Offline
### 4.5 Success

## 5. Composants UI utilisés
<Du catalogue autorisé du design system>

## 6. Tokens appliqués
<Couleurs, typo, espacements — uniquement les tokens du design system, jamais de valeurs en dur>

## 7. Copywriting (avec clés i18n)
| Clé | Valeur |
|---|---|
| `feature.title` | ... |

## 8. Accessibilité (a11y AA minimum)
- [ ] Contraste AA ≥ 4.5:1
- [ ] Labels explicites ou aria-label
- [ ] Ordre de tab logique
- [ ] Rôles ARIA sur zones dynamiques
- [ ] Boutons d'icône avec aria-label
- [ ] Navigation clavier complète
- [ ] Focus visible
- [ ] Texte redimensionnable 200%
- [ ] Messages d'erreur annoncés (aria-live)

## 9. Responsive / safe areas
<Breakpoints web ou safe areas mobile>

## 10. Anti-patterns à éviter (checklist du reviewer)
- [ ] Aucune animation gratuite
- [ ] Aucune valeur en dur (#hex, [Npx], rgb())
- [ ] Aucun texte en dur (tout via i18n)
- [ ] Aucun composant custom si équivalent dans le design system
- [ ] Aucune mutation sans passer par le sync queue (si offline)
- [ ] Dark mode fonctionnel

## 11. Justifications des exceptions
<Si un anti-pattern est volontaire, justifier ici>
```

## Utilisation par le frontend-dev (Generator)

1. Créer `docs/design/<slug>.md` à partir de ce template AVANT d'écrire une ligne de code
2. Remplir les **11 sections** — aucune section vide
3. Ajouter le chemin du brief dans `artifacts[]` de la tâche
4. Implémenter le code en suivant strictement le brief

## Utilisation par le code-reviewer (Evaluator)

1. Vérifier que les **11 sections** sont toutes remplies (rejet si section vide)
2. Confronter le code au brief — tout écart sans justification en section 11 = renvoi
3. Grep checks anti-patterns de la section 10 :
   ```bash
   rg '#[0-9a-fA-F]{3,8}' <fichiers>
   rg '\[[0-9]+(?:\.[0-9]+)?(?:px|rem|em)\]' <fichiers>
   ```
4. Vérifier tokens, composants du catalogue, clés i18n, a11y
