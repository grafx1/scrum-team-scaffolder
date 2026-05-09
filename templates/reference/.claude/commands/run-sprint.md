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

Après chaque subagent : **relire `scrum/sprintN.json` depuis le disque** avant de décider de la suite.

## Garde-fous
- **Maximum 30 itérations** par invocation. Au-delà, arrête et produis un rapport.
- À chaque itération, relis le sprint file depuis le disque. Ne te fie jamais à une version en mémoire.
- Si une itération ne change AUCUN statut dans le sprint file, arrête (situation de blocage).
- Si plusieurs fichiers `scrum/sprint*.json` existent hors archive, arrête immédiatement et alerte — état corrompu.

## Étape 0 — Démarrage du scrum-master

Invoque le subagent `scrum-master` (fenêtre fraîche). Trois issues possibles :

- **Il vient de créer un nouveau sprint file** (`scrum/sprintN.json`) → passe à la boucle ci-dessous.
- **Un sprint actif existe déjà** → le scrum-master produit un rapport d'observation, passe à la boucle.
- **Le scrum-master annonce "SPRINT N CLÔTURÉ"** → un sprint vient de se terminer. Si `tasks.json` contient encore des tâches `todo`, **réinvoque** `scrum-master` une seconde fois pour créer le sprint suivant. Si plus de tâches `todo`, produis le rapport final et arrête.

## Boucle principale

Répète jusqu'à ce que le scrum-master annonce la clôture, ou que la limite d'itérations soit atteinte :

1. **Lis le sprint file** `scrum/sprint*.json` (hors archive).

2. **Lead-dev** — s'il y a des tâches `status == "todo"` et pas encore assignées, invoque `lead-dev` (fenêtre fraîche) pour qu'il les découpe et les assigne.

3. **Exécution** — pour chaque tâche en `status == "in_progress"`, invoque l'agent correspondant à son `assignee` dans une **fenêtre fraîche indépendante** :
   - `architect` → invoque `architect`
   - `backend-dev` → invoque `backend-dev`
   - `frontend-dev` → invoque `frontend-dev` (il produit design brief + code dans la même session, voir son agent pour les Phases 1 et 2)

   Tu peux invoquer plusieurs exécutants en parallèle si leurs tâches sont indépendantes (pas de dépendances croisées dans `dependencies`). Chaque subagent parallèle n'écrit que l'entrée de **sa propre tâche** dans le sprint file — pas de conflit si le protocole `sprint-protocol` est respecté.

4. **Review** — pour chaque tâche en `status == "in_review"` avec `assignee == "code-reviewer"`, invoque `code-reviewer` (fenêtre fraîche).

4b. **Security Review** — en parallèle de l'étape 4, invoque `security-reviewer` (fenêtre fraîche) sur toutes les tâches en `status == "in_review"`. Si un finding CRITIQUE est détecté, la tâche revient en `in_progress` (prime sur la décision du code-reviewer).

5. **QA** — pour chaque tâche en `status == "qa"` avec `assignee == "qa-tester"`, invoque `qa-tester` (fenêtre fraîche).

6. **Commit Git** — à la fin de chaque itération, pour chaque tâche passée à `done` pendant cette itération, lance :
   ```bash
   git add -A && git commit -m "feat(<type>): <title> [<id>]"
   ```

7. **Log d'itération** — imprime un tableau court : id, title, status, assignee.

8. **Observation scrum-master** — toutes les 3 itérations, invoque `scrum-master` (fenêtre fraîche, procédure "Observation") pour un rapport d'état. Si toutes les tâches du sprint sont `done` ou `blocked`, le scrum-master va automatiquement basculer en procédure "Clôture" et archivera le sprint file. Détecte l'annonce "SPRINT N CLÔTURÉ" et sors de la boucle.

## Rapport final

Quand la boucle se termine (clôture ou limite atteinte), produis :
- Numéro du sprint traité
- Nombre total de tâches `done`
- Liste des tâches `blocked` avec la raison (extraite de `history`)
- Récap des commits
- État de `tasks.json` : combien de tâches restantes en `todo`
- Recommandation : relancer `/run-sprint` pour le sprint suivant, ou arrêt si backlog vide
