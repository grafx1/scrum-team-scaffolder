# WhatsApp Business Cloud API — référence technique

Canal privilégié pour les parents au Sénégal (WhatsApp très majoritaire). API officielle Meta.

**Docs** : https://developers.facebook.com/docs/whatsapp/cloud-api

## Prérequis

1. Compte Meta Business Manager validé
2. Numéro de téléphone WhatsApp Business (dédié, pas ton perso)
3. System User Token **permanent** (pas un user token qui expire)
4. Templates pré-approuvés par Meta (délai 2-24h par template)

## Authentification

System User Token en header `Authorization: Bearer <token>`. Le `phone_number_id` et le `business_account_id` sont dans le dashboard Meta.

```
WHATSAPP_TOKEN=EAAG...
WHATSAPP_PHONE_NUMBER_ID=123456789012345
WHATSAPP_BUSINESS_ACCOUNT_ID=123456789
WHATSAPP_WEBHOOK_VERIFY_TOKEN=your_verify_token
WHATSAPP_APP_SECRET=your_app_secret    # pour la signature webhook
```

## Envoyer un template (seul type autorisé hors fenêtre 24h)

```typescript
// whatsapp.service.ts
async sendTemplate(params: {
  to: string;              // sans le +, ex '221771234567'
  templateName: string;    // ex 'bulletin_ready'
  languageCode: string;    // 'fr' ou 'en_US'
  parameters: string[];    // remplace les {{1}}, {{2}}, ...
}) {
  const res = await this.http.post(
    `https://graph.facebook.com/v18.0/${this.phoneNumberId}/messages`,
    {
      messaging_product: 'whatsapp',
      to: params.to,
      type: 'template',
      template: {
        name: params.templateName,
        language: { code: params.languageCode },
        components: [
          {
            type: 'body',
            parameters: params.parameters.map((text) => ({ type: 'text', text })),
          },
        ],
      },
    },
    { headers: { Authorization: `Bearer ${this.token}` } },
  );
  return res.data; // { messages: [{ id: 'wamid.xxx' }] }
}
```

## Envoyer un message libre (fenêtre 24h uniquement)

Tu peux envoyer un message libre uniquement si le parent t'a **écrit dans les dernières 24h**. Hors fenêtre : template obligatoire.

```typescript
async sendFreeText(to: string, body: string) {
  // Vérifier d'abord que la dernière interaction inbound est < 24h
  const lastInbound = await this.tenantDB(async (tx) =>
    tx.select({ ts: messages.receivedAt })
      .from(messages)
      .where(and(eq(messages.from, to), eq(messages.direction, 'in')))
      .orderBy(desc(messages.receivedAt))
      .limit(1),
  );
  if (!lastInbound[0] || Date.now() - lastInbound[0].ts.getTime() > 24 * 3600 * 1000) {
    throw new Error('Outside 24h window — use template');
  }
  // ... envoi
}
```

## Webhook signé

Header : `X-Hub-Signature-256: sha256=<hmac>`

Deux cas sur le même endpoint :

```typescript
@Get('webhooks/whatsapp')
verify(
  @Query('hub.mode') mode: string,
  @Query('hub.verify_token') token: string,
  @Query('hub.challenge') challenge: string,
) {
  if (mode === 'subscribe' && token === this.config.get('WHATSAPP_WEBHOOK_VERIFY_TOKEN')) {
    return challenge; // Meta envoie un GET à la configuration du webhook
  }
  throw new ForbiddenException();
}

@Post('webhooks/whatsapp')
async handleWebhook(
  @Req() req: RawBodyRequest,
  @Headers('x-hub-signature-256') signature: string,
) {
  const raw = req.rawBody.toString('utf8');
  if (!this.verifySignature(raw, signature)) {
    throw new ForbiddenException('Invalid signature');
  }
  const body = JSON.parse(raw);
  // Dispatch selon body.entry[].changes[].field
  // - 'messages' : message inbound ou statut (sent/delivered/read/failed)
}

private verifySignature(rawBody: string, header: string): boolean {
  const sig = header?.replace(/^sha256=/, '');
  if (!sig) return false;
  const expected = crypto
    .createHmac('sha256', this.appSecret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(sig, 'hex'),
    Buffer.from(expected, 'hex'),
  );
}
```

## Templates : création et approbation

Un template se crée dans le dashboard Meta Business Manager (WhatsApp Manager → Message Templates) ou via l'API Graph. Structure type :

```
Nom : bulletin_ready
Catégorie : UTILITY
Langue : fr
Corps : Bonjour {{1}}, le bulletin de {{2}} pour le trimestre {{3}} est disponible.
        Connectez-vous sur {{4}} pour le consulter.
```

- Catégories : `UTILITY` (transactionnel, moins cher), `MARKETING` (promotion, plus cher et soumis à opt-in), `AUTHENTICATION` (OTP)
- Approbation par Meta : 2-24h, souvent rejeté la 1ère fois. Itérer.
- Liste des templates approuvés à versionner dans `docs/whatsapp-templates.md`.

## Pièges WhatsApp

- **Templates pré-approuvés obligatoires** hors fenêtre 24h. Impossible d'envoyer un message libre à un parent qui n'a pas écrit récemment.
- **Numéro sans `+` dans l'API Meta**, mais **avec `+`** en DB (format E.164). Convention : toujours stocker avec `+`, retirer juste avant l'appel Meta. Documenter dans la base de code.
- **Body brut** obligatoire pour la signature du webhook. Configurer `bodyParser.raw` sur la route et utiliser `@Req() req: RawBodyRequest` (NestJS).
- **Catégorie `UTILITY` vs `MARKETING`** : un template mal catégorisé en `MARKETING` coûte ~2× plus cher et exige un opt-in du parent. Notifications scolaires = UTILITY.
- **`wamid` pour idempotence** : le statut webhook renvoie le `wamid` du message original. Utiliser comme clé d'idempotence DB.
- **Rate limit dynamique** : Meta alloue un quota journalier qui augmente avec la qualité (taux de blocage parents). Commencer à 1000 msg/24h, monter progressivement.
- **Parents qui bloquent** : si > 3% des parents bloquent ton numéro, Meta baisse le quota et peut suspendre. Surveiller le `quality_rating` dans le webhook.
- **Pas de rich media en template** hors templates autorisés (header image/video) — et chaque type doit être approuvé séparément.

## Test

Utiliser le numéro de test fourni par Meta pour ton app (dans WhatsApp Manager). Envoyer à ton propre numéro avec le template de test `hello_world` (pré-approuvé par Meta). Mocker Graph API via `nock` pour les tests unitaires.

## Intégration Bull

```typescript
BullModule.registerQueue({
  name: 'whatsapp-outbound',
  limiter: { max: 80, duration: 1000 }, // 80/s conservateur
  defaultJobOptions: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 10000 },
    removeOnComplete: true,
    removeOnFail: 500,
  },
});
```

Stocker `wamid` à l'enqueue pour idempotence, et le `messages.id` DB pour retrouver l'entité métier.
