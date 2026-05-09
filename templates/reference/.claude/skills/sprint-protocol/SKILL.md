---
name: sprint-protocol
description: Protocole dual-file de gestion des sprints — backlog pérenne dans tasks.json et working state volatile dans scrum/sprintN.json. À consulter IMPÉRATIVEMENT par tout agent (scrum-master, lead-dev, architect, backend-dev, frontend-dev, code-reviewer, security-reviewer, qa-tester) avant toute lecture ou écriture de ces fichiers. Définit qui possède quel fichier, la machine à états des statuts, le format des entrées history, la règle de lecture-modification-écriture atomique, et le cycle de vie sprint → archive. Sans respect strict de ce protocole les agents s'écrasent mutuellement et le backlog diverge du working state.
---

# Protocole Sprint — architecture à deux fichiers

Le projet sépare strictement le **backlog** (source de vérité long terme) du **working state** (volatile, réécrit à chaque sprint).

```
ton-projet/
├── tasks.json                      ← BACKLOG : possédé par le scrum-master
└── scrum/
    ├── context.md                  ← CONTEXTE PRÉ-SPRINT : réécrit par scrum-master avant chaque sprint
    ├── memory.md                   ← MÉMOIRE INTER-SPRINTS : alimenté par scrum-master à la clôture
    ├── sprint3.json                ← SPRINT ACTIF : un seul à la fois
    └── archive/
        ├── sprint1.json
        └── sprint2.json            ← SPRINTS CLOS : en lecture seule
```

## 1. Responsabilités par fichier

### `tasks.json` (racine du projet)

- **Possédé exclusivement par le scrum-master.** Aucun autre agent ne l'écrit. Les autres agents peuvent le lire s'ils ont besoin du contexte global du projet, mais c'est rare.
- Contient le backlog brut du projet tel que produit par le PRD / task planner.
- Schéma d'une tâche (champs d'origine à préserver tels quels) :

```jsonc
{
  "id": "T001",
  "title": "Setup monorepo pnpm + workspaces",
  "description": "…",
  "phase": "Phase 0: Fondations",
  "priority": "P0 - Critical",
  "category": "infrastructure",
  "estimated_days": 1,
  "dependencies": [],
  "acceptance_criteria": ["…"],
  "tags": ["setup", "monorepo"],
  "status": "todo"                 // ajouté par le scrum-master
}
```

- Le **seul** champ ajouté/modifié par le scrum-master est `status`, qui prend **uniquement** ces trois valeurs :

```
todo → planned → done
```

- `todo` = pas encore sélectionnée pour un sprint
- `planned` = sélectionnée pour le sprint actif
- `done` = terminée (toutes les acceptance_criteria validées à la clôture du sprint)

- Une tâche bloquée en sprint retourne à `todo` dans tasks.json (elle retournera dans un prochain sprint) — elle n'a jamais le statut `blocked` dans tasks.json.

### `scrum/sprintN.json` (working state)

- **Un seul fichier sprint actif à la fois** dans `scrum/` (hors `archive/`).
- Créé par le scrum-master au début d'un sprint, archivé par le scrum-master à la fin.
- Modifié par **tous les agents** pendant le sprint selon la machine à états ci-dessous.
- Schéma du fichier :

```jsonc
{
  "sprint": 3,
  "created_at": "2026-04-09T10:00:00Z",
  "tasks": [
    {
      "id": "T-001",
      "title": "Endpoint POST /auth/login",
      "description": "...",
      "type": "backend",
      "priority": "high",
      "dependencies": [],
      "status": "todo",            // todo | in_progress | in_review | qa | done | blocked
      "assignee": null,            // backend-dev | frontend-dev | ...
      "atomic": false,             // passé à true par le lead-dev
      "subtasks": [],
      "acceptance_criteria": [],
      "artifacts": [],             // chemins des fichiers produits
      "history": []                // log des transitions
    }
  ]
}
```

- La correspondance avec `tasks.json` est assurée par l'`id`. Le scrum-master copie les champs pertinents de la tâche d'origine (title, description, dependencies, acceptance_criteria) et **ajoute** les champs d'orchestration (status, assignee, atomic, subtasks, artifacts, history). Le champ `category` de tasks.json est mappé vers `type` dans le sprint file, en préservant la valeur (`backend`, `frontend-web`, `frontend-mobile`, `design`, `architecture`, `infrastructure`).

## 2. Découverte du fichier sprint actif

Chaque agent, avant d'agir :

```bash
ls scrum/sprint*.json   # doit retourner exactement UN fichier
```

S'il y a 0 fichier → le scrum-master n'a pas encore créé le sprint. Attendre.
S'il y a 2+ fichiers → erreur critique, s'arrêter et alerter l'humain.

Le nom du fichier détermine le numéro du sprint (`sprint3.json` → sprint 3).

## 3. Lire-modifier-écrire atomique

Règle absolue : **relire le fichier sprint depuis le disque juste avant chaque écriture**. Ne jamais se fier à une version en mémoire datant de plus de quelques secondes.

```
1. Read(scrum/sprintN.json)      ← obligatoire, même si déjà lu plus tôt
2. Modifier uniquement les tâches qui relèvent de ton rôle
3. Write(scrum/sprintN.json)     ← en un seul appel, fichier entier
```

**Ne touche qu'aux tâches dont tu es responsable à cet instant.** Sinon, si un autre agent a modifié le fichier entre ta lecture et ton écriture, ses modifs d'autres tâches seraient écrasées.

## 4. Machine à états du sprint file

```
todo → in_progress → in_review → qa → done
          ↑_____________↓__________↓
                             blocked
```

Transitions **autorisées** :

| De | Vers | Par qui |
|---|---|---|
| `todo` | `in_progress` | lead-dev |
| `todo` | `todo` | lead-dev (lors du découpage en subtasks) |
| `in_progress` | `in_review` | architect / backend-dev / frontend-dev |
| `in_progress` | `blocked` | n'importe quel exécutant |
| `in_review` | `qa` | code-reviewer |
| `in_review` | `in_progress` | code-reviewer (renvoi avec changes_requested) |
| `in_review` | `blocked` | code-reviewer (après 3 renvois au même dev) |
| `in_review` | `in_progress` | security-reviewer (finding CRITIQUE — prime sur le code-reviewer) |
| `qa` | `in_progress` | security-reviewer (finding CRITIQUE détecté après passage en qa) |
| `qa` | `done` | qa-tester |
| `qa` | `in_progress` | qa-tester (test échoue) |
| `blocked` | `todo` | humain uniquement (en cours de sprint) ; sinon le scrum-master en fin de sprint |

Toute autre transition est **interdite**. Si tu te retrouves à vouloir en faire une, c'est un bug dans ton raisonnement — arrête-toi et réévalue.

## 5. Qui peut modifier quoi dans le sprint file

Chaque agent ne modifie que les tâches qui lui sont assignées **et** dont le statut correspond à son rôle :

| Agent | Filtre dans `scrum/sprintN.json` |
|---|---|
| scrum-master | **aucune modification** pendant le sprint — observe seulement |
| lead-dev | `status == "todo"` uniquement |
| architect | `assignee == "architect" && status == "in_progress"` |
| backend-dev | `assignee == "backend-dev" && status == "in_progress"` |
| frontend-dev | `assignee == "frontend-dev" && status == "in_progress"` |
| code-reviewer | `assignee == "code-reviewer" && status == "in_review"` |
| security-reviewer | `status == "in_review"` (intervient en parallèle du code-reviewer sur toutes les tâches) |
| qa-tester | `assignee == "qa-tester" && status == "qa"` |

## 6. Format d'une entrée history

Chaque modification d'une tâche DOIT ajouter une entrée à son tableau `history` :

```json
{
  "ts": "2026-04-09T14:32:10Z",
  "by": "backend-dev",
  "action": "in_progress → in_review",
  "note": "Module auth implémenté, 14 tests passants, coverage 87%"
}
```

- `ts` : ISO 8601 UTC, généré via `new Date().toISOString()` ou `date -u +%Y-%m-%dT%H:%M:%SZ`
- `by` : nom exact de l'agent
- `action` : transition de statut OU action hors transition (ex: `"split into subtasks"`)
- `note` : phrase courte et factuelle

## 7. Subtasks (lead-dev uniquement)

Quand le lead-dev découpe une tâche non-atomique en sous-tâches :

1. Créer les subtasks dans le **même tableau `tasks[]`** du sprint file, avec des ids `T-XXX.1`, `T-XXX.2`…
2. Chaque subtask a son propre `type`, `priority`, `acceptance_criteria`, `dependencies`.
3. La tâche parent :
   - `atomic: true`
   - `status: "todo"` (refermée quand toutes ses subtasks seront `done`)
   - `subtasks: ["T-XXX.1", "T-XXX.2"]`
   - `dependencies` complétées avec les ids des subtasks
4. Les subtasks démarrent en `status: "todo"` pour que le lead-dev les assigne dans la foulée.

**Important** : le découpage ne remonte PAS dans `tasks.json`. tasks.json garde la tâche originale. Seul le sprint file voit la décomposition.

## 8. Cycle de vie d'un sprint

### Création (scrum-master)

1. Vérifier qu'aucun fichier `scrum/sprint*.json` n'existe déjà.
2. Lire `tasks.json`. Identifier les tâches éligibles : `status == "todo"` dont toutes les `dependencies` sont `done`.
3. Trier par `priority` (P0 > P1 > P2 > …) puis par ordre du fichier.
4. Sélectionner les **5 à 10 premières** (ajustable selon la capacité de l'équipe).
5. Générer `scrum/sprint<N+1>.json` en transformant chaque tâche :
   - Copier id, title, description, dependencies, acceptance_criteria
   - Mapper `category` → `type`
   - Copier `priority` tel quel
   - Initialiser `status: "todo"`, `assignee: null`, `atomic: false`, `subtasks: []`, `artifacts: []`, `history: []`
6. Mettre à jour `tasks.json` : passer le `status` des tâches sélectionnées de `todo` à `planned`.
7. Écrire les deux fichiers (sprint file en premier, puis tasks.json).

### Pendant le sprint (tous les autres agents)

Les agents travaillent **exclusivement** sur `scrum/sprintN.json`. Ils ne lisent et ne modifient jamais `tasks.json`.

### Clôture (scrum-master)

1. Détecter que toutes les tâches du sprint file ont `status == "done"` OU `status == "blocked"`.
2. Pour chaque tâche du sprint avec `status == "done"` : passer la tâche correspondante dans `tasks.json` de `planned` à `done`.
3. Pour chaque tâche du sprint avec `status == "blocked"` : passer la tâche correspondante dans `tasks.json` de `planned` à `todo` (elle retournera dans un futur sprint) et préserver la raison du blocage en ajoutant un champ optionnel `last_blocked_reason` à la tâche dans tasks.json.
4. Déplacer `scrum/sprintN.json` → `scrum/archive/sprintN.json`.
5. Produire un rapport de clôture (tâches done, tâches blocked avec raison, commits associés).
6. Si le backlog contient encore des tâches `todo`, proposer à l'humain de lancer le sprint suivant.

## 9. Blocage d'une tâche en cours de sprint

Un agent passe une tâche en `blocked` quand :
- Une dépendance technique manque et n'est pas dans le sprint
- Une ambiguïté dans les `acceptance_criteria` nécessite une décision humaine
- Le code-reviewer a renvoyé la tâche 3 fois au même dev

Dans le sprint file :
```json
{
  "status": "blocked",
  "assignee": null,
  "history": [..., {
    "ts": "...",
    "by": "backend-dev",
    "action": "→ blocked",
    "note": "RAISON PRÉCISE — ce texte sera lu par un humain et remonté dans tasks.json à la clôture"
  }]
}
```

## 10. Champs à ne JAMAIS modifier

Dans le sprint file :
- `id` — immuable
- `description`, `acceptance_criteria`, `dependencies` — seul le lead-dev peut les préciser, et seulement lors du découpage en subtasks
- Le tableau `history` : on ajoute, on ne modifie/supprime jamais

Dans `tasks.json` :
- **Tout**, sauf `status` qui est la propriété exclusive du scrum-master

## 11. Checklist avant toute écriture

- [ ] Je sais quel fichier je modifie (tasks.json ou scrum/sprintN.json)
- [ ] J'ai le droit d'écrire ce fichier (tasks.json = scrum-master seul)
- [ ] J'ai relu le fichier juste avant
- [ ] Je ne modifie QUE les tâches qui relèvent de mon rôle et de mon filtre de statut
- [ ] Chaque transition de statut est autorisée par la machine à états
- [ ] Chaque tâche modifiée a une nouvelle entrée `history` (sauf tasks.json qui n'a pas d'history)
- [ ] Le JSON est valide (pas de virgule traînante, guillemets OK)
