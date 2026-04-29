---
name: ui-designer
description: Conçoit et implémente le design d'interfaces frontend web et d'applications mobiles (React, Next.js, Vue, React Native, Flutter, SwiftUI). Déclencher ce skill dès qu'un agent dev front ou mobile doit créer, refondre ou améliorer des écrans, des composants, un design system, une landing page, un dashboard, une app iOS/Android, ou tout livrable visuel d'interface utilisateur. Mots-clés : "design l'écran", "crée la UI", "maquette", "refonte visuelle", "design system", "composant", "thème", "style l'app", "frontend", "mobile app", "écrans", "wireframe", "prototype", "UX/UI". Couvre le design thinking, le choix esthétique, le design system (tokens, typographie, couleurs, espacements), l'implémentation responsive et accessible, et les bonnes pratiques spécifiques web vs mobile.
---

# UI Designer — Frontend Web & Mobile

Ce skill guide la conception et l'implémentation d'interfaces utilisateurs distinctives, cohérentes et production-ready pour le web et le mobile. Il est destiné aux agents `dev-frontend`, `nextjs-frontend-developer`, `react-native-developer` et `flutter-developer` de Claude Code.

## 1. Cadrage avant de coder

Avant toute ligne de code, répondre explicitement à :

- **Plateforme cible** : Web responsive, PWA, app native iOS/Android, hybride (React Native / Flutter), desktop ?
- **Utilisateurs** : qui, quel contexte d'usage (mobile en mobilité, desktop pro, kiosque…), quelles contraintes d'accessibilité ?
- **Objectif de l'écran** : action principale, parcours, métriques de succès.
- **Contraintes techniques** : framework imposé, design system existant, performance, offline, dark mode.
- **Direction esthétique** : choisir UNE direction claire (minimal raffiné, brutaliste, éditorial, néo-skeuomorphe, playful, corporate sobre, glassmorphism, etc.). Pas de mélange tiède.

Écrire un court "design brief" (5–10 lignes) avant d'implémenter.

## 2. Design system d'abord

Toujours définir les tokens AVANT les écrans, dans un fichier dédié (`tokens.css`, `theme.ts`, `theme.dart`…) :

- **Couleurs** : palette primaire, secondaire, neutres (au moins 9 niveaux), sémantiques (success, warning, error, info), surfaces, fond, bordures. Toujours prévoir le mode sombre.
- **Typographie** : 1 police display + 1 police texte, échelle modulaire (ex. 12 / 14 / 16 / 18 / 20 / 24 / 32 / 48), line-height, letter-spacing. Éviter Arial/Inter par défaut — préférer des choix caractériels (Geist, Söhne, Inter Tight, Satoshi, Manrope, IBM Plex, Fraunces, etc.).
- **Espacements** : échelle 4px ou 8px stricte.
- **Rayons, ombres, bordures** : cohérents, peu nombreux (3–4 valeurs max chacun).
- **Motion** : durées et easings standardisés (ex. 150ms ease-out pour micro, 300ms cubic-bezier pour transitions).

## 3. Spécificités Web

- **Responsive mobile-first** : breakpoints sm/md/lg/xl, tester 360px, 768px, 1280px, 1536px.
- **Layout** : Grid CSS pour structures 2D, Flex pour 1D. Éviter les hauteurs fixes.
- **Accessibilité** : contraste AA minimum, focus visible, navigation clavier, ARIA correct, `prefers-reduced-motion`.
- **Performance** : images optimisées (AVIF/WebP), lazy loading, polices `font-display: swap`, CSS critique.
- **Stack recommandée** : Tailwind + shadcn/ui, ou CSS Modules + tokens. Framer Motion pour les animations React.
- **États** : toujours designer loading, empty, error, success — pas seulement le happy path.

## 4. Spécificités Mobile

- **Respect des plateformes** : Human Interface Guidelines (iOS) et Material Design 3 (Android). Ne pas forcer un look iOS sur Android et vice-versa, sauf direction de marque assumée.
- **Safe areas** : `SafeAreaView` (RN), `SafeArea` (Flutter), padding top/bottom des notchs et home indicator.
- **Tactile** : zones de tap ≥ 44×44pt (iOS) / 48×48dp (Android). Espacer les cibles.
- **Navigation** : tab bar (≤5 items), stack, drawer. Cohérent avec la plateforme.
- **Gestes** : swipe, pull-to-refresh, long-press si pertinent — jamais cachés sans affordance.
- **Feedback** : haptics légers sur actions clés, transitions fluides 60fps minimum.
- **États système** : keyboard avoiding, orientation, dark mode, dynamic type (tailles de police système).
- **Stack recommandée** : React Native + Expo + NativeWind ou Tamagui ; Flutter + Material 3 ; SwiftUI natif.

## 5. Composants

Construire en pensant **composant réutilisable** dès le premier écran :

- Props claires, variantes (`size`, `variant`, `tone`).
- Un seul composant par fichier, nommage explicite.
- Stories ou exemples d'usage en commentaire.
- Pas de styles inline magiques — tout passe par les tokens.

## 6. Checklist de qualité avant livraison

- [ ] Design brief écrit et direction esthétique assumée
- [ ] Tokens définis et utilisés partout (zéro valeur en dur)
- [ ] Mode sombre fonctionnel
- [ ] Responsive / safe areas validés
- [ ] États loading / empty / error / success traités
- [ ] Accessibilité : contraste, focus, ARIA / labels
- [ ] Animations subtiles, jamais gratuites
- [ ] Aucune dépendance UI inutile ajoutée
- [ ] Captures ou preview disponibles

## 7. Anti-patterns à éviter

- Gradients violet/bleu génériques "AI slop".
- Glassmorphism partout sans raison.
- Polices système par défaut sans intention.
- Padding/margin aléatoires hors échelle.
- Copier-coller de templates sans adaptation à la marque.
- Ignorer le dark mode "on verra plus tard".
- Designer uniquement le happy path.

## 8. Livrables attendus

Selon la demande :

1. **Design brief** (markdown court).
2. **Fichier de tokens / thème**.
3. **Composants** implémentés et typés.
4. **Écrans** assemblés avec données mockées réalistes (jamais "Lorem ipsum" — utiliser du contenu plausible métier).
5. **README** d'utilisation si design system.
