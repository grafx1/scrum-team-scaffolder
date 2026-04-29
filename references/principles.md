# Principes architecturaux invariants

Ces 7 principes s'appliquent à **tout bundle** généré, quelle que soit la stack ou le domaine métier.

## 1 — Architecture dual-file

- **`tasks.json`** : backlog pérenne, propriété exclusive du `scrum-master` (todo → planned → done)
- **`scrum/sprintN.json`** : working state volatile, modifié par les 7 autres agents (todo → in_progress → in_review → qa → done + blocked)
- **`scrum/archive/`** : sprints clos, lecture seule

## 2 — Machine à états stricte avec 8 agents

**`tasks.json`** : `todo → planned → done` (retour `planned → todo` si sprint bloqué)

**`scrum/sprintN.json`** : `todo → in_progress → in_review → qa → done` + `blocked`

Les 8 agents : scrum-master, lead-dev, architect, backend-dev, frontend-dev, code-reviewer, security-reviewer, qa-tester. Chaque agent a un filtre d'entrée unique.

## 3 — Progressive disclosure des skills

Les skills de plus de 300 lignes doivent être découpés en hub + `references/`. Pattern officiel Anthropic : charge seulement ce qui est nécessaire.

## 4 — Single generator design+code pour le frontend

Pas d'agent designer séparé. Le `frontend-dev` produit le brief en Phase 1 et le code en Phase 2 (Anthropic Harness Design 2026). Les 3 skills design (`design-spec-template`, `design-system-tokens`, `ui-designer`) sont les grading criteria de l'evaluator.

## 5 — Evaluator adversarial explicite

Le `code-reviewer` a des **grading criteria formels** : grep checks sur anti-patterns, sections du brief complètes, test d'isolation tenant obligatoire, lint + build verts. Sans critères explicites, l'evaluator complimente au lieu de pousser.

## 6 — Security reviewer dédié

Le `security-reviewer` intervient **en parallèle** du code-reviewer sur toute tâche en `in_review`. Il ne review que la sécurité (isolation tenant, secrets, inputs, webhooks, permissions, injections). Un finding CRITIQUE prime sur le "OK" du code-reviewer.

## 7 — 4 skills méthodologiques toujours inclus

`tdd-workflow`, `ddd-modeling`, `clean-code`, `clean-architecture` sont inclus dans **tout** bundle. Ils sont adaptés au langage/framework de la stack mais leurs principes sont invariants. Le glossaire DDD est toujours réécrit pour le domaine du projet.

## Checklist de validation

| # | Invariant | Vérification |
|---|---|---|
| 1 | Dual-file | `tasks.json` + `scrum/` + `scrum/archive/` créés |
| 2 | 8 agents | sprint-protocol liste 8 agents, chaque agent a son filtre |
| 3 | Progressive disclosure | Aucun skill > 300 lignes sans hub + references |
| 4 | Single generator | Pas d'agent `ux-designer` |
| 5 | Evaluator explicite | code-reviewer a des grep checks inline |
| 6 | Security reviewer | security-reviewer.md présent, mentionné dans sprint-protocol et run-sprint |
| 7 | 4 skills méthodo | tdd-workflow + ddd-modeling + clean-code + clean-architecture présents |
