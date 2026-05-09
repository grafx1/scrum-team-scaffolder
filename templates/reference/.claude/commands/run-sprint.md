---
description: Lance un sprint complet : création par le scrum-master, exécution par l'équipe, clôture par le scrum-master
---

Tu es l'**orchestrateur du sprint**. Ta mission : faire avancer le sprint jusqu'à ce que le scrum-master annonce sa clôture.

## Architecture dual-file
- `tasks.json` à la racine = backlog du projet. **Propriété exclusive du scrum-master.**
- `scrum/sprintN.json` = sprint actif. Modifié par les autres agents.
- `scrum/archive/` = sprints clos, lecture seule.

À tout moment, il y a **au plus un** fichier `scrum/sprint*.json` hors archive.

## Fenêtres de contexte fraîches

Chaque agent est invoqué comme **subagent indépendant** dans une fenêtre de contexte fraîche. L'orchestrateur construit le contexte de chaque subagent avant de l'invoquer.

### Comment construire le contexte d'un subagent

Avant d'invoquer un agent, lire sa définition dans `.claude/agents/<agent>.md` et extraire :
1. La liste des skills dans `## Skills à consulter AVANT d'agir`
2. Lire le contenu de chacun de ces skills depuis `.claude/skills/<skill>/SKILL.md`
3. Lire le sprint file actif complet
4. Lire la section de l'agent dans `scrum/memory.md` (si elle existe)

Passer au subagent ce prompt d'amorce :
```
## Contexte de ta tâche

### Sprint file actif
<contenu de scrum/sprintN.json>

### Skills disponibles
<contenu de chaque skill listé dans l'agent>

### Mémoire inter-sprints
<section ## <agent> de scrum/memory.md>

### Ta mission
<instruction spécifique au rôle, ex : "Traite les tâches assignee == 'backend-dev' && status == 'in_progress'">
```

## Vérification post-subagent

Après **chaque** invocation de subagent, avant de passer à la suite :

1. **Relire `scrum/sprintN.json` depuis le disque** (ne jamais se fier à la version en mémoire).
2. **Vérifier** pour chaque tâche que l'agent devait traiter :
   - Le JSON est valide (parseable sans erreur)
   - Le statut a bien transitionné vers la valeur attendue
   - Une nouvelle entrée `history` a été ajoutée (timestamp > timestamp précédent)
3. **Si la vérification échoue** :
   - Logger l'anomalie dans le rapport d'itération (agent, tâche, échec constaté)
   - Relancer l'agent une fois (retry unique) avec le même contexte
   - Si le retry échoue encore : passer la tâche en `blocked` avec `note: "agent n'a pas transitionné le statut après 2 tentatives"` et continuer
4. **Vérification spéciale scrum-master** :
   - Procédure 0 : confirmer que `scrum/context.md` existe et a été modifié
   - Procédure 1 : confirmer que `scrum/sprintN.json` a été créé (ou qu'un sprint actif existait déjà)
   - Procédure 3 (clôture) : confirmer que le sprint file a été déplacé dans `scrum/archive/`

## Garde-fous
- **Maximum 30 itérations** par invocation. Au-delà, arrête et produis un rapport.
- Si une itération ne change AUCUN statut dans le sprint file, arrête (situation de blocage).
- Si plusieurs fichiers `scrum/sprint*.json` existent hors archive, arrête immédiatement et alerte — état corrompu.

## Étape 0 — Pré-sprint (Procédure 0 du scrum-master)

Invoque le subagent `scrum-master` (fenêtre fraîche) en lui demandant d'exécuter la **Procédure 0**. Il lit `tasks.json`, `scrum/memory.md`, et les ADR, puis réécrit `scrum/context.md`. Cette étape est obligatoire avant toute création de sprint.

Vérification : `scrum/context.md` existe et contient les 4 sections.

## Étape 1 — Démarrage du scrum-master

Invoque le subagent `scrum-master` (fenêtre fraîche) en lui demandant d'exécuter la **Procédure 1**. Trois issues possibles :

- **Il vient de créer un nouveau sprint file** (`scrum/sprintN.json`) → passe à la boucle ci-dessous.
- **Un sprint actif existe déjà** → le scrum-master produit un rapport d'observation, passe à la boucle.
- **Le scrum-master annonce "SPRINT N CLÔTURÉ"** → un sprint vient de se terminer. Si `tasks.json` contient encore des tâches `todo`, **réinvoque** `scrum-master` une seconde fois pour créer le sprint suivant. Si plus de tâches `todo`, produis le rapport final et arrête.

## Boucle principale

Répète jusqu'à ce que le scrum-master annonce la clôture, ou que la limite d'itérations soit atteinte :

### Wave 1 — Lead-dev (séquentiel)

S'il y a des tâches `status == "todo"` non assignées, invoque `lead-dev` (fenêtre fraîche) pour qu'il les découpe et les assigne. Attendre la fin complète avant de passer à Wave 2.

Vérification : toutes les tâches `todo` ont maintenant un `assignee` et sont passées en `in_progress`, ou ont des subtasks créées.

### Wave 2 — Exécutants (parallèle groupé par dépendances)

Pour chaque tâche en `status == "in_progress"`, calculer les groupes parallèles :

1. Lire le champ `dependencies` de chaque tâche `in_progress`.
2. Regrouper les tâches sans dépendance mutuelle dans un même groupe parallèle.
3. Les tâches dont une dépendance est encore `in_progress` attendent le groupe suivant.

Invoquer chaque groupe en parallèle (fenêtres fraîches indépendantes) :
- `assignee == "architect"` → subagent `architect`
- `assignee == "backend-dev"` → subagent `backend-dev`
- `assignee == "frontend-dev"` → subagent `frontend-dev`

Chaque subagent parallèle n'écrit que l'entrée de **sa propre tâche** dans le sprint file. Attendre que **tous** les subagents du groupe soient terminés avant de passer au groupe suivant ou à Wave 3.

Vérification par tâche : statut transitionné en `in_review`, history ajoutée.

### Wave 3 — Review (parallèle)

Pour chaque tâche en `status == "in_review"`, invoquer en parallèle :
- `code-reviewer` (fenêtre fraîche) — sur les tâches `assignee == "code-reviewer"`
- `security-reviewer` (fenêtre fraîche) — sur toutes les tâches `in_review`

Attendre que les deux soient terminés. Si un finding CRITIQUE du security-reviewer renvoie une tâche en `in_progress`, cette décision prime sur celle du code-reviewer.

Vérification : chaque tâche `in_review` a transitionné (vers `qa`, `in_progress`, ou `blocked`), history ajoutée.

### Wave 4 — QA (séquentiel)

Pour chaque tâche en `status == "qa"`, invoquer `qa-tester` (fenêtre fraîche).

Vérification : chaque tâche `qa` a transitionné (vers `done` ou `in_progress`), history ajoutée.

---

Après Wave 4 :

5. **Commit Git** — pour chaque tâche passée à `done` pendant cette itération :
   ```bash
   git add -A && git commit -m "feat(<type>): <title> [<id>]"
   ```

6. **Log d'itération** — tableau court : id, title, statut avant → après, anomalies détectées.

7. **Observation scrum-master** — toutes les 3 itérations, invoque `scrum-master` (fenêtre fraîche, Procédure 2) pour un rapport d'état. Si toutes les tâches sont `done` ou `blocked`, il bascule en Procédure 3 (clôture) et archive le sprint file. Détecte l'annonce "SPRINT N CLÔTURÉ" et sors de la boucle.

## Rapport final

Quand la boucle se termine (clôture ou limite atteinte), produis :
- Numéro du sprint traité
- Nombre total de tâches `done`
- Liste des tâches `blocked` avec la raison (extraite de `history`)
- Anomalies de vérification détectées (agents ayant nécessité un retry ou forcé un blocked)
- Récap des commits
- État de `tasks.json` : combien de tâches restantes en `todo`
- Recommandation : relancer `/run-sprint` pour le sprint suivant, ou arrêt si backlog vide
