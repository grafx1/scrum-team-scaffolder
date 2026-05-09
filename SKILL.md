---
name: scrum-team-scaffolder
description: Génère un bundle Claude Code complet (agents + skills + commands + backlog) pour une équipe scrum de 8 agents autonomes, adapté à la stack d'un projet donné. À déclencher quand l'utilisateur dit "scaffolder une équipe scrum", "initialiser Claude Code pour ce projet", "générer les agents et skills", "créer un bundle scrum-team", "bootstrap Claude Code", ou fournit une description de stack technique et demande un environnement Claude Code prêt à l'emploi.
---

# scrum-team-scaffolder

Génère un bundle Claude Code complet (`.claude/agents/`, `.claude/skills/`, `.claude/commands/` + `tasks.json` + `README.md`) pour une équipe scrum de **8 agents autonomes** adaptée à n'importe quelle stack technique.

## Philosophie : LLM-first

Le scaffolder contient des **principes** et une **référence complète**, pas une bibliothèque de templates par techno. Claude extrapole à partir de la référence pour produire un bundle customisé. Les parties stack-agnostiques sont copiées verbatim, les parties stack-specific sont adaptées.

## Workflow en 4 phases

### Phase 1 — Interview

1. Lire `references/stack-dimensions.md` pour les dimensions obligatoires.
2. Parser la description de l'utilisateur. Inférer ce qui est implicite.
3. Si des dimensions obligatoires manquent, poser **toutes les questions restantes dans une seule réponse** (voir `references/interview-script.md`).
4. Ne pas passer en Phase 2 tant que les dimensions obligatoires ne sont pas remplies.

### Phase 2 — Résolution

1. Lire `references/composition-rules.md`.
2. Pour chaque dimension de stack, déterminer le nom du skill cible et le fichier de référence à adapter.
3. Résoudre les interactions inter-dimensions (patterns tenant par langage, queue par framework, etc.).
4. Dresser la liste des fichiers à copier verbatim, adapter, ou générer.

### Phase 3 — Génération

**3.1 — Copies verbatim** (stack-agnostiques, toujours inclus) :
- `skills/sprint-protocol/SKILL.md`
- `skills/adr-template/SKILL.md`
- `commands/run-sprint.md`
- `agents/scrum-master.md`
- `scrum/memory.md` — fichier vide avec structure initiale (4 sections par agent)

**Skills méthodologiques** — toujours inclus, adaptés au langage cible :
- `skills/tdd-workflow/SKILL.md` — adapter mocking, fixtures multi-tenant, conventions test au langage
- `skills/ddd-modeling/SKILL.md` — adapter exemples code, events, **réécrire le glossaire** pour le domaine du projet
- `skills/clean-code/SKILL.md` — adapter grep checks, conventions nommage/fichiers au langage
- `skills/clean-architecture/SKILL.md` — adapter mapping couches→fichiers au framework

**Skills conditionnels** (inclus si frontend présent) :
- `skills/design-spec-template/SKILL.md`
- `skills/design-system-tokens/SKILL.md` (à customiser : palette, typo, composants du projet)
- `skills/ui-designer/SKILL.md`

**Conditionnel offline** : `skills/sync-queue-offline/SKILL.md` (si mobile ou PWA)

**3.2 — Skills stack-specific adaptés** : ORM, auth, queue, storage, framework backend, frontend web, frontend mobile — chacun adapté depuis le skill correspondant dans la référence.

**3.3 — Intégrations** : hub + `references/<vendor>.md` si des intégrations sont listées.

**3.4 — Agents customisés** : `lead-dev`, `architect`, `backend-dev`, `frontend-dev`, `code-reviewer`, `security-reviewer`, `qa-tester` — adapter les sections "Skills à consulter" et "Stack" aux noms des skills générés.

**3.5 — `tasks.json`** squelette de 8-12 tâches cohérentes avec la stack.

**3.6 — README.md et CHANGELOG.md**.

### Phase 4 — Validation

1. Lint cross-références : chaque skill mentionné dans un agent existe dans `skills/`.
2. Lint exemples de code : même stack partout, pas de mix.
3. Lint agents : 8 agents exactement, pas de fantôme `ux-designer`.
4. Rapport de scaffolding : stack détectée, fichiers générés, points d'extrapolation.

## Les 8 agents

| Agent | Rôle |
|---|---|
| `scrum-master` | Crée / observe / clôture les sprints. Seul à écrire `tasks.json`. |
| `lead-dev` | Découpe en tâches atomiques + assigne |
| `architect` | ADR, schéma DB, contrats API |
| `backend-dev` | Implémente les modules backend |
| `frontend-dev` | Design brief (Phase 1) + code (Phase 2) |
| `code-reviewer` | Evaluator adversarial : relit et renvoie ou valide |
| `security-reviewer` | Audit de sécurité en parallèle du code-reviewer |
| `qa-tester` | Tests fonctionnels + non-régression |

## Invariants (tout bundle)

1. **Dual-file** : `tasks.json` + `scrum/sprintN.json` + `scrum/archive/`
2. **Machine à états** stricte : 8 agents, statuts définis
3. **Progressive disclosure** pour les skills > 300 lignes
4. **Single generator design+code** dans `frontend-dev`
5. **Evaluator adversarial** avec grep checks dans `code-reviewer`
6. **Security reviewer** dédié en parallèle du code-reviewer
7. **4 skills méthodologiques** toujours inclus (TDD, DDD, Clean Code, Clean Architecture)
8. **Mémoire inter-sprints** : `scrum/memory.md` toujours inclus — alimenté par `scrum-master` à chaque clôture, lu par `backend-dev`, `frontend-dev`, `code-reviewer` avant chaque tâche

## Règles pour Claude

- **Ne jamais mélanger les technos** dans un même bundle
- **Nommage** : `<techno-qualifier>-<role>` en kebab-case
- **Toujours inclure** les 3 skills design si frontend présent, **jamais** sinon
- **Toujours exclure** `sync-queue-offline` si pas d'offline
- **Jamais d'agent `ux-designer`** — fusionné dans `frontend-dev`
- **Toujours 8 agents** — les 7 standards + `security-reviewer`
- **Toujours adapter le glossaire DDD** au domaine du projet, pas à un domaine exemple
- **Préserver la structure** des skills de référence lors de l'adaptation
