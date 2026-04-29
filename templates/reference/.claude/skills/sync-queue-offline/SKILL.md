---
name: sync-queue-offline
description: Contrat commun de file de synchronisation offline-first pour le web (Dexie.js/IndexedDB) et le mobile (expo-sqlite). Utiliser IMPÉRATIVEMENT dès qu'un agent (frontend-dev, architect, backend-dev) implémente une mutation utilisateur, un formulaire, une action qui doit survivre à une perte de réseau, ou tout ce qui touche aux entités offline (élèves, présences, notes, paiements). Couvre le format des mutations en attente, la stratégie de retry, la résolution de conflits last-write-wins avec timestamp serveur, et l'idempotency key. Sans respect strict de ce contrat, le web et le mobile divergeront et la sync sera cassée.
---

# SyncQueue offline-first

Contexte : connectivité irrégulière au Sénégal → toute l'app doit fonctionner offline. Les mutations utilisateur sont mises en file locale et rejouées quand la connexion revient. **Le contrat DOIT être identique entre web et mobile** pour que l'équipe backend n'ait qu'une seule surface à gérer.

## 1. Principe

```
UI → optimistic update → SyncQueue (local) → [network] → API → réconciliation → local DB
```

- La mutation est immédiatement appliquée à la base locale (Dexie ou SQLite) pour que l'UI soit réactive.
- Elle est aussi poussée dans `sync_queue`, une table locale dédiée.
- Un worker (web: `setInterval` + `navigator.onLine` ; mobile: `NetInfo` + background task) dépile et envoie au backend.
- Le backend renvoie l'état canonique (id serveur, timestamps), le client réconcilie.

## 2. Schéma de la table `sync_queue` (identique web & mobile)

```typescript
interface SyncQueueEntry {
  id: string;                    // uuid v4 généré côté client
  idempotencyKey: string;        // = id, envoyé au backend pour dédup
  entity: string;                // "student" | "attendance" | "grade" | ...
  operation: 'create' | 'update' | 'delete';
  localEntityId: string;         // id local (temporaire si create)
  serverEntityId: string | null; // rempli après première sync réussie
  payload: Record<string, unknown>;
  createdAt: string;             // ISO, horloge client
  attempts: number;              // nombre de tentatives
  lastAttemptAt: string | null;
  lastError: string | null;
  status: 'pending' | 'in_flight' | 'acked' | 'failed' | 'conflict';
}
```

Ce type est déclaré **une seule fois** dans `packages/types/src/sync.ts` et importé par les deux apps.

## 3. Interface SyncQueue (contrat obligatoire)

```typescript
// packages/types/src/sync.ts
export interface SyncQueue {
  enqueue(entry: Omit<SyncQueueEntry, 'id' | 'idempotencyKey' | 'createdAt' | 'attempts' | 'status' | 'lastAttemptAt' | 'lastError' | 'serverEntityId'>): Promise<string>;
  pending(limit?: number): Promise<SyncQueueEntry[]>;
  markInFlight(id: string): Promise<void>;
  markAcked(id: string, serverEntityId: string): Promise<void>;
  markFailed(id: string, error: string): Promise<void>;
  markConflict(id: string, serverState: unknown): Promise<void>;
  purgeAcked(olderThanMs: number): Promise<number>;
}
```

Implémentations :
- **Web** : `apps/web/src/lib/sync/dexie-queue.ts` utilisant Dexie.js
- **Mobile** : `apps/mobile/src/lib/sync/sqlite-queue.ts` utilisant expo-sqlite

Les deux implémentent `SyncQueue` et passent la **même** suite de tests dans `packages/types/test/sync-queue.contract.ts`. Si tu modifies l'interface, tu modifies les deux implémentations **et** les tests.

## 4. Idempotency key

Chaque mutation a un `idempotencyKey` = uuid v4 généré au moment de `enqueue`. Le backend le reçoit via le header `Idempotency-Key`.

Côté backend NestJS, un interceptor Redis stocke `idempotencyKey → response` pendant 24h. Si la même clé revient, il renvoie la réponse précédente sans rejouer la mutation. C'est ce qui rend les retries sûrs.

## 5. Résolution de conflits : last-write-wins avec timestamp serveur

Stratégie par défaut pour le MVP :

1. Chaque entité a un champ `updatedAt` géré **par le serveur** (pas par le client).
2. Quand le client envoie un `update`, il inclut le `updatedAt` qu'il avait au moment du changement dans `If-Unmodified-Since` ou un champ `expectedUpdatedAt` du payload.
3. Si le serveur voit que son `updatedAt` actuel est plus récent → **409 Conflict** + état serveur dans la réponse.
4. Le client marque l'entrée `status: "conflict"` et l'UI affiche un écran de résolution manuelle (pour le MVP : "garder version locale" / "garder version serveur").

Pour `create` : pas de conflit possible grâce à l'idempotency key.

Pour `delete` : toujours gagne (tombstone côté serveur avec `deletedAt`).

## 6. Optimistic update + réconciliation d'ids

Problème : quand l'utilisateur crée une entité offline, le client génère un id local (`local-<uuid>`). Le serveur en renverra un différent.

Solution :
1. À la création offline, insère en base locale avec `id: "local-<uuid>"`.
2. Enqueue une mutation `create` avec `localEntityId: "local-<uuid>"`.
3. Quand le serveur répond, `markAcked(queueId, serverId)` met à jour **toutes** les lignes locales qui référençaient `local-<uuid>` (l'entité elle-même + les FK des autres entités créées offline qui pointaient vers elle).
4. L'UI re-render automatiquement (TanStack Query invalide la query).

⚠️ Les autres mutations offline qui référencent `localEntityId` doivent rester en file **dans l'ordre** et ne partir qu'après l'ack du create parent. Le worker doit donc respecter les dépendances :

```typescript
// Bloque une mutation si son payload référence un localEntityId pas encore acked
function canSend(entry: SyncQueueEntry, queue: SyncQueueEntry[]): boolean {
  const refs = extractLocalRefs(entry.payload);
  return refs.every(localId =>
    queue.find(e => e.localEntityId === localId)?.status === 'acked'
  );
}
```

## 7. Worker de sync (pattern commun)

```typescript
async function syncTick(queue: SyncQueue, api: ApiClient) {
  if (!isOnline()) return;
  const batch = await queue.pending(10);
  for (const entry of batch) {
    if (!canSend(entry, batch)) continue;
    await queue.markInFlight(entry.id);
    try {
      const res = await api.send(entry);
      await queue.markAcked(entry.id, res.serverEntityId);
    } catch (err) {
      if (err.status === 409) {
        await queue.markConflict(entry.id, err.body);
      } else if (entry.attempts >= 5) {
        await queue.markFailed(entry.id, String(err));
      } else {
        // back-off exponentiel : 2^attempts secondes, cap 5 min
      }
    }
  }
}
```

- Web : appelé toutes les 30s et sur `online` event.
- Mobile : appelé toutes les 30s en foreground, + background task Expo (`expo-background-fetch`) toutes les 15 min.

## 8. Tests de contrat (obligatoires)

`packages/types/test/sync-queue.contract.ts` exporte une fonction `runContractTests(createQueue: () => SyncQueue)` que chaque implémentation lance. Elle couvre :

- Enqueue → pending retourne l'entry
- markInFlight puis markAcked : l'entry disparaît de pending
- markFailed avec attempts=5 → status failed
- Ordre FIFO respecté
- Purge acked par âge

Le frontend-dev **ne peut pas** considérer une implémentation SyncQueue comme terminée sans que ces tests passent.

## 9. Checklist pour ajouter une nouvelle entité synchronisable

- [ ] Type partagé dans `packages/types/src/entities/<entity>.ts`
- [ ] `create` / `update` / `delete` passent par `syncQueue.enqueue`, jamais d'appel direct à l'API depuis l'UI
- [ ] L'entité a `updatedAt` géré serveur
- [ ] Endpoint backend lit le header `Idempotency-Key`
- [ ] Endpoint backend retourne 409 avec état serveur en cas de conflit
- [ ] Test e2e offline → reco → sync → cohérent
- [ ] UI gère l'état `conflict` de la sync_queue (même si c'est juste un toast pour le MVP)
