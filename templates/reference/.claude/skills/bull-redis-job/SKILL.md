---
name: bull-redis-job
description: Crée un job asynchrone Bull Queue / Redis pour le SaaS Éducation Sénégal — envoi SMS Africa's Talking, WhatsApp Business, génération PDF bulletin, upload Cloudflare R2, relances impayés — avec retry, idempotence, backoff et observabilité. À déclencher quand l'utilisateur dit "ajoute un job async X", "envoie des SMS en masse", "génère les bulletins en batch", ou travaille sur src/queues/** / src/jobs/** dans l'app NestJS.
type: skill
---

# bull-redis-job

## Stack de référence

- **Bull Queue** (ou BullMQ) + **Redis**
- NestJS `@nestjs/bull` pour l'intégration (`@Processor`, `@Process`)
- Dimensionnable horizontalement (workers dédiés pour SMS/PDF massifs en fin de trimestre)
- Observabilité : logs structurés + métriques Prometheus

## Queues standard du projet

| Queue | Usage | Concurrence |
|---|---|---|
| `sms` | Notifications Africa's Talking (absences, relances) | 10 |
| `whatsapp` | WhatsApp Business via Twilio/360dialog | 5 |
| `pdf-bulletin` | Génération bulletins trimestriels | 3 |
| `r2-upload` | Upload fichiers vers Cloudflare R2 | 5 |
| `payment-webhook` | Traitement webhooks Wave / Orange Money | 10 |

## Squelette d'un processor

```ts
@Processor('sms')
export class SmsProcessor {
  private readonly logger = new Logger(SmsProcessor.name);

  @Process({ name: 'send', concurrency: 10 })
  async handleSend(job: Job<SendSmsPayload>) {
    const { schoolId, to, template, vars, idempotencyKey } = job.data;
    // 1. idempotence : noop si déjà envoyé
    if (await this.alreadySent(idempotencyKey)) return;
    // 2. appel provider
    await this.africasTalking.send({ to, message: render(template, vars) });
    // 3. persister le résultat dans notifications (RLS actif via tenant context)
    await this.notifications.markSent(idempotencyKey);
  }
}
```

## Configuration obligatoire

```ts
BullModule.registerQueue({
  name: 'sms',
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 }, // 2s, 4s, 8s, 16s, 32s
    removeOnComplete: { count: 1000 },
    removeOnFail: false,
  },
});
```

## Conventions

- **Idempotence obligatoire** : chaque job porte un `idempotencyKey` (UUID v4 ou hash métier stable)
- Payload minimal : stocker les IDs, pas les objets complets — le worker relit la DB
- Tenant : le worker doit re-positionner `SET LOCAL app.current_school_id` avant toute requête PG
- Logs : `{ jobId, queue, schoolId, attempt }` en JSON structuré
- Failed jobs : ne **pas** auto-supprimer → Dead Letter Queue inspectable

## Pièges à éviter

- Payload volumineux en Redis → passer des IDs, pas des blobs
- Oublier le contexte tenant dans le worker → RLS bloque ou bypass silencieux
- Pas de limite de concurrency → DoS du provider SMS
- Retry infini sur erreur métier (400) → filtrer les erreurs non-retryables
- Oublier `removeOnComplete` → Redis sature
- Pas de monitoring des queues en retard (`waiting > 1000`)

## Livrable

Queue déclarée + processor typé + producer dans le service métier + métriques exposées + test unitaire mock Redis.
