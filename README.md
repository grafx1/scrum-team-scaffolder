# scrum-team-scaffolder

**Meta-skill Claude Code qui génère une équipe scrum d'agents autonomes adaptée à n'importe quelle stack technique.**

Donne-lui ta stack (backend, ORM, auth, queue, frontend, intégrations...) et il produit un dossier `.claude/` complet — 8 agents, 15 à 23 skills, un backlog initial, une commande `/run-sprint` — prêt à exécuter des sprints automatiquement dans Claude Code.

---

## Installation

### Option A — Installation globale (recommandée)

```bash
cp -r scrum-team-scaffolder ~/.claude/skills/
```

Disponible dans toutes tes sessions Claude Code, sur tous tes projets.

### Option B — Installation par projet

```bash
mkdir -p mon-projet/.claude/skills
cp -r scrum-team-scaffolder mon-projet/.claude/skills/
```

Disponible uniquement dans ce projet.

---

## Utilisation

Depuis Claude Code, décris ta stack en langage naturel :

```
Scaffolde une équipe scrum pour mon projet.

Stack :
- Backend : FastAPI + Python + SQLAlchemy + PostgreSQL
- Multi-tenant : single-db-rls
- Auth : Auth0 Organizations
- Queue : Celery + Redis
- Storage : AWS S3 eu-west-1
- Frontend web : Next.js 14 App Router + shadcn/ui
- Mobile : Flutter
- Intégrations : Stripe, Twilio, SendGrid

Nom : healthtech-eu
Pitch : Plateforme de gestion médicale multi-établissements conforme RGPD.
```

Claude suit un workflow en 4 phases et produit le bundle complet.

---

## Ce que le scaffolder produit

```
<ton-projet>/
├── .claude/
│   ├── agents/                 # 8 agents
│   │   ├── scrum-master.md
│   │   ├── lead-dev.md
│   │   ├── architect.md
│   │   ├── backend-dev.md
│   │   ├── frontend-dev.md
│   │   ├── code-reviewer.md
│   │   ├── security-reviewer.md
│   │   └── qa-tester.md
│   │
│   ├── skills/                 # 15 à 23 skills selon la stack
│   │   │
│   │   │  # Invariants (copiés verbatim)
│   │   ├── sprint-protocol/
│   │   ├── adr-template/
│   │   │
│   │   │  # Méthodologie (adaptés au langage)
│   │   ├── tdd-workflow/
│   │   ├── ddd-modeling/
│   │   ├── clean-code/
│   │   ├── clean-architecture/
│   │   │
│   │   │  # Stack-specific (générés par extrapolation)
│   │   ├── <orm>-schema/
│   │   ├── <auth>-auth/
│   │   ├── <queue>-job/
│   │   ├── <storage>-storage/
│   │   ├── <framework>-module/
│   │   ├── <frontend-web>-*/
│   │   ├── <frontend-mobile>-*/
│   │   │
│   │   │  # Design (si frontend présent)
│   │   ├── design-spec-template/
│   │   ├── design-system-tokens/
│   │   ├── ui-designer/
│   │   │
│   │   │  # Intégrations (si spécifiées)
│   │   └── <integrations>-hub/ + references/*.md
│   │
│   └── commands/
│       └── run-sprint.md
│
├── scrum/
│   ├── memory.md               # Mémoire inter-sprints (alimentée par scrum-master)
│   ├── context.md              # Contexte pré-sprint (réécrit par scrum-master avant chaque sprint)
│   └── archive/
├── tasks.json                  # Backlog initial (8-12 tâches)
├── README.md
└── CHANGELOG.md
```

---

## Les 4 phases du scaffolder

### Phase 1 — Interview

Claude collecte les dimensions manquantes. **Toutes les questions sont posées dans un seul message** (pas de ping-pong). Si la description est complète, cette phase est silencieuse.

Dimensions obligatoires : nom, pitch, backend framework + langage, DB + ORM, stratégie multi-tenant, auth provider, queue, storage, frontend web, frontend mobile, intégrations.

Dimensions optionnelles (défauts sensés) : testing, i18n, déploiement, package manager.

### Phase 2 — Composition

Claude résout quels skills et agents générer d'après les règles de `references/composition-rules.md`. Il identifie les templates à adapter et les patterns multi-tenant à appliquer.

### Phase 3 — Génération

1. **Skills invariants** copiés verbatim
2. **Skills méthodologiques** adaptés au langage (exemples, grep checks, fixtures)
3. **Skills stack-specific** générés par extrapolation depuis le bundle de référence
4. **Hub d'intégrations** + references par vendor
5. **8 agents** avec refs de skills adaptées
6. **`scrum/memory.md`** — fichier de mémoire inter-sprints vide avec structure initiale
7. **`scrum/context.md`** — fichier de contexte pré-sprint vide avec 4 sections fixes
8. **Fichiers racine** : `tasks.json`, `README.md`, `CHANGELOG.md`

### Phase 4 — Validation et rapport

Claude vérifie la cohérence (cross-références, pas de mélange de technos, 8 agents) et produit un rapport listant la stack détectée, les fichiers générés, et les points d'extrapolation à relire manuellement.

---

## Les 8 agents

| Agent | Rôle | Intervient quand |
|---|---|---|
| **scrum-master** | Crée/observe/clôture les sprints, seul à écrire `tasks.json` | Début et fin de sprint |
| **lead-dev** | Découpe en tâches atomiques, assigne aux devs | Tâches `status: todo` |
| **architect** | ADR, schéma DB, contrats API, choix de patterns | `type: architecture` |
| **backend-dev** | Implémentation backend (TDD, DDD, clean code) | `assignee: backend-dev` |
| **frontend-dev** | Design brief Phase 1 + code Phase 2 | `assignee: frontend-dev` |
| **code-reviewer** | Review qualité avec grading criteria et grep checks | `status: in_review` |
| **security-reviewer** | Audit sécurité en parallèle (tenant isolation, secrets, injections, webhooks) | `status: in_review` |
| **qa-tester** | Tests fonctionnels, isolation tenant, offline, non-régression | `status: qa` |

### Le security-reviewer

Intervient **en parallèle** du code-reviewer sur toutes les tâches en review. Domaines :

- Isolation multi-tenant (requêtes hors wrapper tenant, tenant ID depuis le body)
- Validation des inputs (endpoints sans validation, injection SQL, `eval()`)
- Secrets (credentials en dur, secrets dans les logs/URLs)
- Auth/authz (controllers sans guard, opérations sans check de rôle)
- Webhooks (signature sur body brut, `timingSafeEqual`)
- Données sensibles (PII en clair, champs non chiffrés)
- Supply chain (vulnérabilités connues, packages non lockés)
- XSS (si frontend)

Un finding **CRITIQUE** prime sur la décision du code-reviewer et renvoie la tâche au dev.

---

## Les 4 skills méthodologiques

Toujours inclus, quelle que soit la stack. Adaptés au langage cible. Format restructuré (phases + anti-patterns + checklists, ≤ 150 lignes chacun) pour maximiser l'application par les agents.

| Skill | Impose |
|---|---|
| **`tdd-workflow`** | Test AVANT le code. Cycle Red/Green/Refactor. 4 couches (unit, integration, e2e, isolation tenant). Convention `should ... when ...`. |
| **`ddd-modeling`** | Un module = un bounded context. Agrégats ≤ 3 tables. Value objects. Domain events via queue. Glossaire dans `docs/glossary.md`. |
| **`clean-code`** | Nommage intentionnel. Fonctions ≤ 20 lignes, ≤ 3 params. Early return. Zéro `any` / code mort / `console.log`. Grep checks. |
| **`clean-architecture`** | Logique métier dans le service, pas le controller. Dépendances vers l'intérieur. Infra interchangeable. CRUD simple = pas de sur-structuration. |

---

## Architecture du bundle généré

### Dual-file : backlog + sprint

Le bundle sépare le backlog pérenne (`tasks.json`, propriété du scrum-master) du working state du sprint actif (`scrum/sprintN.json`, modifié par tous les agents). Les sprints clos sont archivés dans `scrum/archive/`.

### Mémoire inter-sprints

`scrum/memory.md` accumule les apprentissages de chaque sprint. Le scrum-master l'alimente automatiquement à chaque clôture en extrayant du `history` des tâches :

| Tag | Origine |
|---|---|
| `[PATTERN]` | Tâche terminée sans rejet en < 2 itérations |
| `[PIÈGE]` | Tâche renvoyée ≥ 2 fois en `in_progress` |
| `[REJET]` | Motif de rejet récurrent du code-reviewer |
| `[BLOQUÉ]` | Raison de blocage non résolue |

Le fichier est structuré par agent (`## backend-dev`, `## frontend-dev`, `## code-reviewer`, `## architect`). Chaque agent lit uniquement sa section avant d'agir. Limite : 10 entrées par section (FIFO).

### Contexte pré-sprint

`scrum/context.md` capture l'état de connaissance de l'équipe avant chaque sprint. Le scrum-master le réécrit intégralement lors de la **Procédure 0** (Étape 0 du `/run-sprint`), depuis trois sources :

- `scrum/memory.md` — apprentissages des sprints passés
- `docs/adr/` — décisions d'architecture actives
- `tasks.json` — dépendances non résolues dans le backlog

Le fichier contient 4 sections fixes :

| Section | Lue par |
|---|---|
| `## Décisions d'architecture actives` | architect, backend-dev, frontend-dev, code-reviewer |
| `## Contraintes connues` | backend-dev, frontend-dev |
| `## Zones à risque` | backend-dev, frontend-dev |
| `## Dépendances non résolues` | lead-dev |

Chaque agent lit **uniquement la section pertinente à son rôle** avant d'agir. Ce pattern (inspiré du CONTEXT.md de [GSD](https://github.com/gsd-build/get-shit-done)) aligne l'équipe sur les décisions prises sans polluer le contexte des agents avec des informations non pertinentes.

### Machine à états

```
Backlog :  todo → planned → done

Sprint  :  todo → in_progress → in_review → qa → done
                      ↑              │        │
                      └──────────────┘────────┘
                      (code-reviewer, security-reviewer,
                       ou qa-tester renvoie)

                   blocked (en escape de n'importe quel état)
```

---

## Lancer un sprint

```bash
cd mon-projet
git checkout -b sprint-1
claude
# puis :
/run-sprint
```

L'orchestrateur boucle : lead-dev assigne → devs implémentent → reviewers reviewent (qualité + sécurité en parallèle) → qa teste → scrum-master clôture.

### Fenêtres de contexte fraîches

Chaque agent est invoqué dans un **subagent indépendant avec fenêtre de contexte fraîche**. L'orchestrateur construit le contexte minimal avant chaque invocation :

1. Lit la section `## Skills à consulter` de la définition de l'agent
2. Charge le contenu de chaque skill listé
3. Charge le sprint file actif complet
4. Charge la section `scrum/memory.md` de l'agent

L'agent reçoit exactement ce dont il a besoin — rien de plus. Ce pattern (inspiré de [GSD](https://github.com/gsd-build/get-shit-done)) élimine le *context rot* : la dégradation silencieuse de la qualité lorsque la fenêtre de contexte se sature au fil des itérations d'un sprint long.

---

## Personnalisation du bundle généré

| Action | Comment |
|---|---|
| Ajouter un agent | Nouveau `.md` dans `agents/` + mise à jour de `sprint-protocol` et `run-sprint` |
| Changer la stack | Éditer les skills dans `.claude/skills/` — les agents pointent vers les skills par nom |
| Ajouter une intégration | Nouveau `references/<vendor>.md` dans le hub d'intégrations + ligne dans le routing table |
| Ajuster le design system | Éditer `design-system-tokens/SKILL.md` (palette, typo, composants autorisés) |
| Enrichir le glossaire DDD | Éditer `docs/glossary.md` — le code-reviewer vérifie la conformité |

---

## Structure du meta-skill

```
scrum-team-scaffolder/
├── SKILL.md                          # Hub : workflow 4 phases
├── README.md                         # Ce fichier
├── references/
│   ├── principles.md                 # 5 invariants architecturaux
│   ├── stack-dimensions.md           # Taxonomie des dimensions de stack
│   ├── composition-rules.md          # Playbook d'adaptation par techno
│   ├── interview-script.md           # Format des questions Phase 1
│   └── integration-template-format.md # Format des references d'intégrations
├── templates/
│   └── reference/                    # Bundle de référence complet
│       └── .claude/ (8 agents + 19 skills + run-sprint)
└── examples/
    └── senegal-edtech.yaml           # Exemple de stack YAML
```

---

## Enrichir le scaffolder

1. **Après chaque utilisation** : ajoute le YAML de la stack dans `examples/`
2. **Si Claude a mal extrapolé** : ajoute une règle dans `composition-rules.md`
3. **Si une stack devient fréquente** : ajoute-la comme référence secondaire dans `templates/`

---

## Les 11 invariants

Tout bundle généré respecte ces règles sans exception :

1. **Dual-file** — `tasks.json` + `scrum/sprintN.json` + `scrum/archive/`
2. **Machine à états stricte** — 8 agents avec filtres d'entrée documentés
3. **Progressive disclosure** — skills > 300 lignes ont un hub + `references/`
4. **Single generator design+code** — `frontend-dev` fait brief + code, pas d'agent designer séparé
5. **Evaluator adversarial** — `code-reviewer` avec grep checks + `security-reviewer` en parallèle
6. **4 skills méthodologiques** — TDD, DDD, Clean Code, Clean Architecture toujours inclus
7. **Skills concis** — ≤ 150 lignes, format phases + anti-patterns + checklists
8. **Mémoire inter-sprints** — `scrum/memory.md` toujours inclus, alimenté par `scrum-master` à chaque clôture
9. **Fenêtres de contexte fraîches** — chaque agent est un subagent isolé, contexte minimal injecté par l'orchestrateur
10. **TDD non-négociable** — aucune ligne d'implémentation sans test rouge d'abord, veto du code-reviewer sinon
11. **Contexte pré-sprint** — `scrum/context.md` réécrit par `scrum-master` avant chaque sprint, 5 agents lisent leurs sections pertinentes

---

## Limitations v0.1

- **Une référence concrète** (TypeScript + NestJS + Drizzle). Qualité excellente pour les stacks TS voisines, inférée pour Python/Ruby/Java/Go.
- **Pas de CLI `npx`** — tout dans Claude Code.
- **Pas de validation automatique** — le rapport Phase 4 est informatif, pas bloquant.
- **Intégrations exotiques inférées** — qualité proportionnelle à la notoriété du vendor.

---

## Versions

| Version | Date | Contenu |
|---|---|---|
| v0.4 | 2026-05 | Contexte pré-sprint (`scrum/context.md` + Procédure 0 scrum-master). 5 agents lisent leurs sections pertinentes. 11 invariants. |
| v0.3 | 2026-05 | Fenêtres de contexte fraîches par agent (anti-context-rot). Règle de concision dans les 8 agents. 9 invariants. |
| v0.2 | 2026-05 | Restructuration des skills (≤ 150 lignes, phases + anti-patterns + checklists). Checklists migrées vers les agents. Mémoire inter-sprints (`scrum/memory.md`). TDD non-négociable. 8 invariants. |
| v0.1 | 2026-04 | Version initiale. 1 référence. 8 agents (dont security-reviewer). 19 skills (dont 4 méthodologiques : TDD, DDD, Clean Code, Clean Architecture). |
