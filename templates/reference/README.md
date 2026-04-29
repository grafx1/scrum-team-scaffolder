# Équipe Scrum Claude Code

Équipe de 8 agents Claude Code pour exécuter automatiquement un backlog de projet via des sprints itératifs.

## Les 8 agents

| Agent | Rôle |
|---|---|
| `scrum-master` | Crée / observe / clôture les sprints. Seul à écrire `tasks.json` |
| `lead-dev` | Découpe en tâches atomiques + assigne |
| `architect` | ADR, schéma DB, contrats API |
| `backend-dev` | Implémente les modules backend |
| `frontend-dev` | Design brief (Phase 1) + code (Phase 2) |
| `code-reviewer` | Evaluator adversarial : relit et renvoie ou valide |
| `security-reviewer` | Audit de sécurité en parallèle du code-reviewer |
| `qa-tester` | Tests fonctionnels + non-régression |

## Les skills

### Transverses (toujours inclus)
- `sprint-protocol` — dual-file, machine à états, format history
- `adr-template` — format des Architecture Decision Records

### Méthodologiques (toujours inclus)
- `tdd-workflow` — Red/Green/Refactor, couches de test, test d'isolation tenant
- `ddd-modeling` — bounded contexts, agrégats, value objects, domain events, glossaire
- `clean-code` — nommage, fonctions courtes, types stricts, grep checks
- `clean-architecture` — séparation couches, dépendances unidirectionnelles

### Design (inclus si frontend)
- `design-spec-template` — format du design brief (11 sections)
- `design-system-tokens` — palette, typo, composants autorisés
- `ui-designer` — méthodologie design, anti-patterns

### Stack-specific (adaptés au projet)
Les skills techniques (ORM, auth, queue, storage, framework, intégrations) sont spécifiques à la stack du projet.

## Usage

```bash
cp -r .claude/ /chemin/vers/ton/projet/
cp tasks.json /chemin/vers/ton/projet/
cp -r scrum/ /chemin/vers/ton/projet/
cd /chemin/vers/ton/projet
git checkout -b sprint-1
claude
# puis : /run-sprint
```
