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
├── scrum/ + archive/
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
6. **Fichiers racine** : `tasks.json`, `README.md`, `CHANGELOG.md`

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

Toujours inclus, quelle que soit la stack. Adaptés au langage cible.

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

## Les 5 principes invariants

Tout bundle généré respecte ces 5 règles :

1. **Dual-file** — `tasks.json` + `scrum/sprintN.json` + `scrum/archive/`
2. **Machine à états stricte** — 8 agents avec filtres d'entrée documentés
3. **Progressive disclosure** — skills > 300 lignes ont un hub + `references/`
4. **Single generator design+code** — `frontend-dev` fait brief + code, pas d'agent designer séparé
5. **Evaluator adversarial** — `code-reviewer` avec grep checks + `security-reviewer` en parallèle

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
| v0.1 | 2026-04 | Version initiale. 1 référence. 8 agents (dont security-reviewer). 19 skills (dont 4 méthodologiques : TDD, DDD, Clean Code, Clean Architecture). |
