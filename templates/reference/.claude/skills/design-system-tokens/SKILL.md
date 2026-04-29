---
name: design-system-tokens
description: Système de design du projet — palette de couleurs, typographie, espacements, rayons, ombres, et catalogue des composants shadcn/ui autorisés. Source de vérité unique partagée entre frontend-dev (web et mobile) et code-reviewer. À consulter dès qu'un agent choisit une couleur, une taille de police, un espacement, un composant UI, ou valide un design brief. Toute valeur en dehors de ces tokens est un rejet automatique en review. Les tokens sont configurés dans tailwind.config.ts côté web et dans un fichier miroir côté mobile.
---

# Design System Tokens

Source de vérité unique pour tous les choix visuels. **Aucune valeur hardcodée en dehors de ce fichier.**

## 1. Philosophie

- Palette restreinte : moins de choix = plus de cohérence
- Contraste AA minimum sur tous les couples texte/fond
- Compatible dark mode dès le début (variables CSS inversables)
- Les tokens shadcn/ui par défaut sont la base ; on personnalise seulement la couleur primaire et les accents

## 2. Couleurs (Tailwind + CSS vars)

Tailwind est configuré avec les variables CSS de shadcn/ui. Dans `apps/web/src/app/globals.css` :

```css
:root {
  /* Neutrals */
  --background: 0 0% 100%;          /* blanc */
  --foreground: 222 47% 11%;        /* gris très foncé */
  --muted: 210 40% 96%;
  --muted-foreground: 215 16% 47%;
  --border: 214 32% 91%;
  --input: 214 32% 91%;

  /* Brand : vert sénégalais discret (référence drapeau, pas criard) */
  --primary: 142 71% 29%;           /* #0F7A3D */
  --primary-foreground: 0 0% 100%;

  /* Accent : jaune soleil, réservé au highlight */
  --accent: 45 93% 58%;             /* #F5C22C */
  --accent-foreground: 222 47% 11%;

  /* Feedback */
  --destructive: 0 72% 51%;         /* rouge erreur */
  --destructive-foreground: 0 0% 100%;
  --success: 142 71% 45%;
  --warning: 38 92% 50%;
  --info: 199 89% 48%;

  /* Card & popover */
  --card: 0 0% 100%;
  --card-foreground: 222 47% 11%;
  --popover: 0 0% 100%;
  --popover-foreground: 222 47% 11%;

  /* Radii */
  --radius: 0.5rem;
}

.dark {
  --background: 222 47% 11%;
  --foreground: 210 40% 98%;
  --muted: 217 33% 17%;
  --muted-foreground: 215 20% 65%;
  --border: 217 33% 17%;
  --input: 217 33% 17%;
  --primary: 142 71% 45%;
  --primary-foreground: 0 0% 100%;
  --accent: 45 93% 58%;
  --accent-foreground: 222 47% 11%;
  --destructive: 0 63% 31%;
  --destructive-foreground: 0 0% 98%;
  --card: 222 47% 11%;
  --card-foreground: 210 40% 98%;
  --popover: 222 47% 11%;
  --popover-foreground: 210 40% 98%;
}
```

**Classes Tailwind à utiliser** (jamais de hex inline) :
- `bg-background` / `text-foreground` — base
- `bg-muted` / `text-muted-foreground` — secondaire
- `bg-primary` / `text-primary-foreground` — CTA
- `bg-accent` / `text-accent-foreground` — highlight ponctuel
- `bg-destructive` / `text-destructive-foreground` — erreurs
- `border-border` — toutes les bordures

## 3. Typographie

Une seule famille : **Inter** (Google Fonts, self-hostée via `next/font`). Pas de fallback exotique.

| Token Tailwind | Usage | Taille | Line-height | Weight |
|---|---|---|---|---|
| `text-xs` | Labels, métadonnées | 12px | 16px | 400 |
| `text-sm` | Corps de texte compact, boutons | 14px | 20px | 400 |
| `text-base` | Corps de texte standard | 16px | 24px | 400 |
| `text-lg` | Sous-titres | 18px | 28px | 500 |
| `text-xl` | Titres d'écran | 20px | 28px | 600 |
| `text-2xl` | Titres de section importante | 24px | 32px | 600 |
| `text-3xl` | Hero (rare, landing uniquement) | 30px | 36px | 700 |

**Weights autorisés** : 400 (regular), 500 (medium), 600 (semibold), 700 (bold). Jamais 300 ou 800+.

## 4. Espacements

Échelle Tailwind par défaut, limitée à ces valeurs :

| Token | px | Usage |
|---|---|---|
| `gap-1` / `p-1` / `m-1` | 4px | micro-espacements (icône + texte) |
| `gap-2` | 8px | à l'intérieur d'un composant compact |
| `gap-3` | 12px | standard intra-composant |
| `gap-4` | 16px | entre composants proches |
| `gap-6` | 24px | entre sections |
| `gap-8` | 32px | grandes sections |
| `gap-12` | 48px | séparation majeure |

**Interdits** : `gap-5`, `gap-7`, `gap-9`, `gap-10`, `gap-11` (éviter la dispersion).

## 5. Rayons

| Token | Usage |
|---|---|
| `rounded-sm` (2px) | inputs très compacts |
| `rounded-md` (6px) | boutons, inputs standards |
| `rounded-lg` (8px) | cards, modales |
| `rounded-xl` (12px) | cards hero |
| `rounded-full` | avatars, badges, FAB |

## 6. Ombres

Utilisation parcimonieuse, jamais plus d'un niveau d'ombre sur un écran.

| Token | Usage |
|---|---|
| `shadow-sm` | cards subtiles |
| `shadow` | cards interactives au hover |
| `shadow-lg` | popovers, modales |
| `shadow-xl` | drawer, sheet |

## 7. Breakpoints (web)

```
sm: 640px    (rarement utilisé)
md: 768px    (tablette — bascule mobile→desktop)
lg: 1024px   (desktop petit)
xl: 1280px   (desktop standard)
```

Règle : **mobile-first**. Tous les styles de base sont pour mobile, puis on surcharge avec `md:`, `lg:`.

## 8. Catalogue shadcn/ui autorisé

Composants installés et **autorisés**. Si tu as besoin d'un autre, demande d'abord s'il existe — ne réinvente pas.

| Composant | Installer avec | Usage |
|---|---|---|
| `Button` | `pnpm dlx shadcn@latest add button` | Tous les CTAs |
| `Input` | `... add input` | Champs de formulaire texte |
| `Textarea` | `... add textarea` | Champs multilignes |
| `Label` | `... add label` | Associé à tout input |
| `Select` | `... add select` | Listes déroulantes courtes |
| `Checkbox` | `... add checkbox` | Booléens |
| `RadioGroup` | `... add radio-group` | Choix unique parmi 2-5 options |
| `Switch` | `... add switch` | Booléens de préférence |
| `Card` | `... add card` | Conteneurs de contenu |
| `Dialog` | `... add dialog` | Modales |
| `Sheet` | `... add sheet` | Drawer latéral |
| `Toast` | `... add toast` + sonner | Notifications éphémères |
| `Alert` | `... add alert` | Messages d'état in-page |
| `Badge` | `... add badge` | Tags, statuts |
| `Avatar` | `... add avatar` | Photos de profil |
| `Skeleton` | `... add skeleton` | États de chargement |
| `Tabs` | `... add tabs` | Navigation secondaire |
| `DropdownMenu` | `... add dropdown-menu` | Menus contextuels |
| `Popover` | `... add popover` | Infos contextuelles |
| `Form` | `... add form` | Wrapper react-hook-form |
| `Table` | `... add table` | Tableaux de données (desktop uniquement) |

**Non autorisés pour le MVP** : Carousel, Calendar avec range, Command palette, Menubar, ContextMenu, Accordion (utiliser Tabs à la place).

## 9. Équivalents mobile (React Native)

Le mobile n'utilise pas shadcn/ui. À la place, un petit design system interne dans `packages/ui-mobile/` qui mappe les mêmes tokens :

| Composant mobile | Équivalent shadcn | Fichier |
|---|---|---|
| `<Button>` | `Button` | `packages/ui-mobile/src/Button.tsx` |
| `<TextField>` | `Input` + `Label` | `packages/ui-mobile/src/TextField.tsx` |
| `<Card>` | `Card` | `packages/ui-mobile/src/Card.tsx` |
| `<Sheet>` | `Sheet` | `packages/ui-mobile/src/Sheet.tsx` (via `@gorhom/bottom-sheet`) |
| `<Toast>` | `Toast` | `packages/ui-mobile/src/Toast.tsx` |
| `<Avatar>` | `Avatar` | `packages/ui-mobile/src/Avatar.tsx` |
| `<Skeleton>` | `Skeleton` | `packages/ui-mobile/src/Skeleton.tsx` |

Les tokens de couleur sont partagés via `packages/tokens/src/colors.ts` consommé par Tailwind (web) ET par les composants RN (mobile).

## 10. Icônes

**Une seule lib** : `lucide-react` (web) / `lucide-react-native` (mobile). Pas d'émoji en UI, pas de Heroicons, pas de Material Icons.

Tailles autorisées : 16px, 20px, 24px, 32px. Couleur via `currentColor` pour hériter du texte.

## 11. Vérificateur de contraste

Avant d'approuver une spec, le designer (et le code-reviewer en validation) doit vérifier :
- Tous les couples texte/fond atteignent **WCAG AA** (4.5:1 pour le texte normal, 3:1 pour le texte ≥18px bold)
- Vérifier avec https://webaim.org/resources/contrastchecker/ ou l'outil DevTools Chrome

Les couples pré-validés :
- `foreground` sur `background` : ✅ 15.8:1
- `primary-foreground` sur `primary` : ✅ 5.2:1
- `muted-foreground` sur `background` : ✅ 4.7:1
- `destructive-foreground` sur `destructive` : ✅ 4.9:1

## 12. Checklist usage des tokens

Pour le designer :
- [ ] Toutes les couleurs de la spec sont référencées par leur token (pas de hex)
- [ ] Toutes les tailles de texte sont dans l'échelle `text-*`
- [ ] Tous les espacements sont dans l'échelle autorisée (1/2/3/4/6/8/12)
- [ ] Tous les composants sont dans le catalogue shadcn autorisé

Pour le frontend-dev :
- [ ] Aucune valeur `#hex`, `rgb()`, `px` hardcodée dans le code (sauf exceptions justifiées en commentaire)
- [ ] Classes Tailwind utilisant les variables CSS (`bg-primary`, pas `bg-[#0F7A3D]`)
- [ ] Icônes lucide uniquement
- [ ] Pas de `font-family` custom

Pour le code-reviewer :
- Rechercher systématiquement `#`, `rgb(`, `px]`, `rem]`, `[0.` dans le diff. Toute occurrence = question.
