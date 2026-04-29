---
name: expo-router-screen
description: Conventions React Native + Expo Router pour l'app mobile du projet. Utiliser dès qu'un agent (frontend-dev, architect) crée un écran mobile, une navigation, un formulaire, une intégration expo-sqlite, ou un background task de sync. Couvre l'arborescence file-based d'Expo Router, les layouts Stack/Tabs, l'intégration SyncQueue mobile, les permissions, la gestion du réseau avec NetInfo, et les tests RNTL. Complémentaire au skill nextjs-app-router pour garder la cohérence entre web et mobile.
---

# Expo Router — conventions mobile

L'app mobile partage **les mêmes types, schémas zod et queryKeys** que le web (via `packages/`). Seules la couche UI et la couche stockage diffèrent.

## 1. Arborescence file-based

```
apps/mobile/app/
├── _layout.tsx                # root layout, providers TanStack Query + i18n
├── (auth)/
│   ├── _layout.tsx            # Stack
│   ├── login.tsx
│   └── callback.tsx
├── (app)/
│   ├── _layout.tsx            # Tabs ou Drawer, protégé
│   ├── index.tsx              # dashboard
│   ├── students/
│   │   ├── _layout.tsx        # Stack
│   │   ├── index.tsx          # liste
│   │   └── [id].tsx           # détail
│   └── attendance.tsx
└── +not-found.tsx
```

**Groupes** entre parenthèses (`(auth)`, `(app)`) = pas de segment dans l'URL, juste un regroupement logique.

## 2. Layout root

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/lib/query-client';
import { IntlProvider } from '@/lib/i18n';
import { SyncWorker } from '@/lib/sync/sync-worker';

export default function RootLayout() {
  return (
    <QueryClientProvider client={queryClient}>
      <IntlProvider>
        <SyncWorker />
        <Stack screenOptions={{ headerShown: false }} />
      </IntlProvider>
    </QueryClientProvider>
  );
}
```

`SyncWorker` est un composant invisible qui lance le tick de sync (voir skill `sync-queue-offline`).

## 3. Navigation typée

Expo Router génère les types des routes. Utilise-les :

```tsx
import { router, Link } from 'expo-router';

// Navigation impérative
router.push('/(app)/students/123');

// Navigation déclarative
<Link href="/(app)/students/123">Voir</Link>
```

Jamais de string concaténée à la main — passe toujours par `router.push` avec la chaîne littérale.

## 4. Écran standard

```tsx
// app/(app)/students/index.tsx
import { View, FlatList } from 'react-native';
import { useQuery } from '@tanstack/react-query';
import { queryKeys } from '@packages/query-keys';
import { api } from '@/lib/api';
import { StudentCard, Spinner, ErrorView } from '@/components';
import { useTranslation } from '@/lib/i18n';

export default function StudentsScreen() {
  const { t } = useTranslation();
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: queryKeys.students.all,
    queryFn: () => api.GET('/students'),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorView onRetry={refetch} />;

  return (
    <FlatList
      data={data}
      keyExtractor={(s) => s.id}
      renderItem={({ item }) => <StudentCard student={item} />}
      accessibilityLabel={t('students.listLabel')}
    />
  );
}
```

## 5. Stockage local : expo-sqlite

```ts
// lib/db/index.ts
import * as SQLite from 'expo-sqlite';

export const db = SQLite.openDatabaseSync('app.db');

db.execSync(`
  CREATE TABLE IF NOT EXISTS students (
    id TEXT PRIMARY KEY,
    school_id TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    sync_pending INTEGER DEFAULT 0
  );
  CREATE INDEX IF NOT EXISTS idx_students_school ON students(school_id);
`);
```

Le schéma local **reflète** celui du backend mais n'est pas généré automatiquement — toute modification passe par une migration locale (fichier `lib/db/migrations/NNN-description.ts`).

## 6. Détection réseau

```tsx
import NetInfo from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';

export function useIsOnline() {
  const [online, setOnline] = useState(true);
  useEffect(() => {
    return NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected && !!state.isInternetReachable);
    });
  }, []);
  return online;
}
```

Le worker de sync (skill `sync-queue-offline`) appelle `NetInfo.fetch()` avant chaque tick.

## 7. Background sync

Dans `app.json` :
```json
{
  "expo": {
    "plugins": [
      ["expo-background-fetch", { "minimumInterval": 900 }]
    ]
  }
}
```

Enregistrement dans `_layout.tsx` :
```tsx
import * as BackgroundFetch from 'expo-background-fetch';
import * as TaskManager from 'expo-task-manager';

TaskManager.defineTask('SYNC_TASK', async () => {
  await syncTick(syncQueue, api);
  return BackgroundFetch.BackgroundFetchResult.NewData;
});
```

## 8. Permissions

- Caméra (photos d'élèves) : `expo-image-picker`, demander au moment du premier usage, pas au lancement.
- Notifications : `expo-notifications`, opt-in explicite.
- **Jamais** de permission demandée sans contexte utilisateur clair.

## 9. Formulaires

Même stack que le web : `react-hook-form` + `zod` + schémas importés de `@packages/api-contract`. Composants custom `<FormField>` qui wrappent `TextInput` avec label et erreur.

## 10. i18n

Même namespace et même lib de clés que le web (`packages/i18n`). L'app web et l'app mobile partagent les mêmes fichiers `messages/fr.json`, `messages/wo.json`, `messages/en.json`.

## 11. Tests

- **Unitaires** : Jest + `@testing-library/react-native`
- Mock `expo-sqlite` via `jest-expo`
- Mock `NetInfo` pour tester les scénarios offline
- **Pas de Detox** pour le MVP (trop lourd à maintenir)

## 12. Checklist avant `in_review`

- [ ] Écran dans le bon groupe `(auth)`/`(app)`
- [ ] `openapi-fetch` typé partagé avec le web
- [ ] Mutations via `syncQueue` (expo-sqlite)
- [ ] `FlatList` pour les listes longues (pas `ScrollView` + map)
- [ ] États `isLoading` / `error` gérés avec composants communs
- [ ] Textes via i18n, pas en dur
- [ ] Accessibility labels présents
- [ ] Test RNTL du composant
- [ ] `pnpm lint && pnpm test` passent dans `apps/mobile`
