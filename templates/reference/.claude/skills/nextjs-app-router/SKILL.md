---
name: nextjs-app-router
description: Conventions Next.js 14 App Router pour l'app web du projet. Utiliser dès qu'un agent (frontend-dev, architect) crée une nouvelle route, page, layout, Server Component, Client Component, ou intègre TanStack Query, shadcn/ui, react-hook-form ou Dexie. Couvre la structure des routes, la séparation Server/Client Components, le pattern TanStack Query avec openapi-fetch typé, les formulaires zod, les états loading/error, l'i18n et l'accessibilité. Garantit la cohérence de tous les écrans web.
---

# Next.js 14 App Router — conventions du projet

## 1. Structure d'une route

```
apps/web/src/app/(app)/students/
├── page.tsx              # Server Component par défaut
├── loading.tsx           # skeleton shadcn pendant le fetch
├── error.tsx             # Client Component, boundary d'erreur
├── layout.tsx            # optionnel, si sous-navigation
└── [id]/
    ├── page.tsx
    ├── loading.tsx
    └── edit/
        └── page.tsx      # formulaire
```

Groupes de routes :
- `(auth)` — login, callback Keycloak
- `(app)` — layout authentifié avec sidebar, protégé par middleware
- `(public)` — landing, mentions légales

## 2. Server vs Client Components

**Règle par défaut : Server Component.** N'ajoute `'use client'` que si le composant utilise :
- `useState`, `useEffect`, `useRef`, autres hooks
- Event handlers (`onClick`, `onChange`)
- APIs navigateur (`localStorage`, `IndexedDB`, `navigator`)
- TanStack Query côté client

**Pattern recommandé** : Server Component qui fait le premier fetch + hydrate un Client Component :

```tsx
// app/(app)/students/page.tsx  (Server Component)
import { StudentsList } from './students-list';
import { getQueryClient } from '@/lib/query-client';
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { api } from '@/lib/api';

export default async function StudentsPage() {
  const qc = getQueryClient();
  await qc.prefetchQuery({
    queryKey: ['students'],
    queryFn: () => api.GET('/students'),
  });

  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <StudentsList />
    </HydrationBoundary>
  );
}
```

```tsx
// app/(app)/students/students-list.tsx  (Client Component)
'use client';
import { useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api';

export function StudentsList() {
  const { data, isLoading } = useQuery({
    queryKey: ['students'],
    queryFn: () => api.GET('/students'),
  });
  // ...
}
```

## 3. Data fetching : openapi-fetch + TanStack Query

**Jamais** de `fetch` direct. Toujours via le client typé :

```tsx
// lib/api.ts
import createClient from 'openapi-fetch';
import type { paths } from '@packages/api-contract';

export const api = createClient<paths>({
  baseUrl: process.env.NEXT_PUBLIC_API_URL,
});
```

Les types `paths` sont générés depuis l'OpenAPI produit par NestJS (`@nestjs/swagger`) dans le CI.

**Clés de query typées** dans `lib/query-keys.ts` :

```ts
export const queryKeys = {
  students: {
    all: ['students'] as const,
    detail: (id: string) => ['students', id] as const,
  },
} as const;
```

## 4. Mutations : TOUJOURS via SyncQueue

Voir skill `sync-queue-offline`. **Jamais** d'appel direct à l'API pour une mutation :

```tsx
'use client';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { syncQueue } from '@/lib/sync/dexie-queue';

export function useCreateStudent() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (data: CreateStudentDto) => {
      // optimistic update local DB
      const localId = `local-${crypto.randomUUID()}`;
      await db.students.add({ id: localId, ...data, syncPending: true });
      // enqueue
      await syncQueue.enqueue({
        entity: 'student',
        operation: 'create',
        localEntityId: localId,
        payload: data,
      });
      return localId;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: queryKeys.students.all }),
  });
}
```

## 5. Formulaires : react-hook-form + zod

```tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createStudentSchema, type CreateStudentDto } from '@packages/api-contract';
import { Button, Input, Label } from '@/components/ui';  // shadcn

export function StudentForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<CreateStudentDto>({
    resolver: zodResolver(createStudentSchema),
  });
  const createStudent = useCreateStudent();

  return (
    <form onSubmit={handleSubmit((d) => createStudent.mutate(d))}>
      <Label htmlFor="firstName">{t('student.firstName')}</Label>
      <Input id="firstName" {...register('firstName')} aria-invalid={!!errors.firstName} />
      {errors.firstName && <p role="alert">{errors.firstName.message}</p>}
      {/* ... */}
      <Button type="submit" disabled={createStudent.isPending}>
        {t('common.save')}
      </Button>
    </form>
  );
}
```

**Le schéma zod est importé depuis `packages/api-contract`** — même source de vérité que le backend.

## 6. Composants UI : shadcn/ui uniquement

- Ajout via `pnpm dlx shadcn-ui@latest add <component>` — les fichiers atterrissent dans `src/components/ui/`.
- **Ne pas** installer d'autres libs de composants (pas de Material UI, pas de Chakra, pas de Mantine).
- Tailwind pour le styling custom, classes utilitaires uniquement. Pas de CSS modules.

## 7. i18n

- Lib : `next-intl`
- Clés dans `messages/fr.json`, `messages/wo.json` (wolof), `messages/en.json`
- **Jamais** de texte en dur dans le JSX. Toujours `t('namespace.key')`.
- Par défaut : `fr`.

## 8. Accessibilité

- Tout `<input>` a un `<Label htmlFor>` associé.
- Erreurs avec `role="alert"` et `aria-invalid`.
- Focus visible (Tailwind `focus-visible:ring-2`).
- Navigation clavier testée.
- Contraste minimum AA (Tailwind `text-foreground` sur `bg-background`).

## 9. Tests

- **Unitaires** : Vitest + React Testing Library dans `<component>.test.tsx` à côté du composant.
- **E2E** : Playwright pour les parcours critiques (login, création élève, présence).
- Mocker `api` via MSW, pas via stub manuel.

## 10. Checklist avant `in_review`

- [ ] Server Component par défaut, `'use client'` uniquement si nécessaire
- [ ] `openapi-fetch` typé, jamais de `fetch` direct
- [ ] Mutations passent par `syncQueue`
- [ ] Formulaire avec zodResolver et schéma partagé
- [ ] Textes via `next-intl`, pas en dur
- [ ] shadcn/ui + Tailwind, pas d'autre lib UI
- [ ] `loading.tsx` et `error.tsx` présents pour la route
- [ ] Test RTL du composant + test Playwright si parcours critique
- [ ] `pnpm lint && pnpm test && pnpm build` passent
