# Africa's Talking — référence technique

SMS (et USSD) pour notifier les parents. Couverture Sénégal complète.

**Docs** : https://developers.africastalking.com

## Authentification

Headers HTTP `apiKey` + `username`. Pas d'OAuth.

```
AT_API_KEY=atsk_xxxxxxxx
AT_USERNAME=your_app
AT_SENDER_ID=SCHOOLAPP
```

## Envoi SMS

```typescript
// africas-talking.service.ts
async sendSms(params: {
  to: string[];            // format E.164 : ['+221771234567']
  message: string;
  senderId?: string;
}) {
  const body = new URLSearchParams({
    username: this.username,
    to: params.to.join(','),
    message: params.message,
    from: params.senderId ?? this.senderId,
  });

  const res = await this.http.post(
    'https://api.africastalking.com/version1/messaging',
    body,
    {
      headers: {
        apiKey: this.apiKey,
        'Content-Type': 'application/x-www-form-urlencoded',
        Accept: 'application/json',
      },
    },
  );

  // res.data.SMSMessageData.Recipients[] contient un statut par numéro
  return res.data.SMSMessageData;
}
```

## Normalisation E.164 (obligatoire)

Les numéros **doivent** être en E.164 : `+221771234567`. AT rejette silencieusement les autres formats (ou pire, envoie sans prévenir).

```typescript
// util/phone.ts
export function toE164Senegal(raw: string): string {
  const digits = raw.replace(/\D/g, '');
  if (digits.startsWith('221')) return `+${digits}`;
  if (digits.length === 9) return `+221${digits}`;     // 77 123 45 67
  if (digits.length === 10 && digits.startsWith('0'))  // 077 123 45 67
    return `+221${digits.slice(1)}`;
  throw new Error(`Invalid Senegalese phone: ${raw}`);
}
```

À normaliser **à la saisie** (formulaire parent côté web/mobile) ET à relire systématiquement côté backend avant l'envoi.

## Sender ID

Le `from` (sender ID) alphanumérique doit être **pré-approuvé par AT** pour le Sénégal :

1. Demande via le dashboard AT (délai 48h)
2. Nom max 11 caractères alphanumériques, pas d'espace
3. Sans ça, le SMS part avec un sender générique `AFRICASTK` ou échoue selon le pays

## Webhook delivery reports

AT peut POSTer les statuts de livraison sur un endpoint que tu configures dans le dashboard :

```typescript
@Post('webhooks/at/delivery')
async handleDelivery(@Body() body: AtDeliveryReport) {
  // body: { id, status: 'Success'|'Failed'|..., statusCode, phoneNumber, networkCode }
  await this.smsLogService.updateStatus(body.id, body.status);
}
```

Pas de signature native — si tu veux durcir, whitelist les IPs Africa's Talking (liste dans le dashboard).

## Pièges Africa's Talking

- **Format E.164 strict** : toujours `+221...`. Jamais de format local.
- **Sender ID alphanumérique** : pré-approbation 48h obligatoire pour le SN.
- **Coût réel en sandbox** : certains envois facturent même en sandbox partielle — toujours **mocker en dev local** via `nock` ou `msw-node`.
- **Retry avec doublons** : AT retourne un `messageId` par destinataire — stocker avant de retenter un envoi sinon doublons facturés.
- **Unicode** : les SMS avec des caractères non-ASCII sont comptés en unités de 70 caractères (GSM7 = 160). Un message de 80 caractères avec un accent = 2 SMS facturés. Prévenir l'utilisateur côté formulaire d'envoi en masse.
- **Limites de débit** : 600 SMS/min par défaut. Au-delà, AT rate-limite silencieusement. Pour les envois en masse (notifications parents d'une école entière), utiliser Bull avec `limiter: { max: 500, duration: 60000 }`.
- **Pas d'API de rappel** : pour savoir si un parent a lu le SMS, tu n'as que `delivered` (reçu sur le réseau), pas `read`.

## Test

```typescript
import * as nock from 'nock';

it('envoie un SMS avec le bon E.164', async () => {
  nock('https://api.africastalking.com')
    .post('/version1/messaging', (body) => {
      expect(body.toString()).toContain('to=%2B221771234567');
      return true;
    })
    .reply(201, { SMSMessageData: { Recipients: [{ status: 'Success', messageId: 'ATX_1' }] } });

  await service.sendSms({ to: ['+221771234567'], message: 'Hello' });
});
```

## Intégration Bull

Les envois SMS en masse (bulletins dispo, retard paiement) passent par un processor Bull avec rate limiter :

```typescript
// bull module config
BullModule.registerQueue({
  name: 'sms-outbound',
  limiter: { max: 500, duration: 60_000 }, // 500/min, marge sous la limite AT
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
    removeOnComplete: true,
    removeOnFail: 100,
  },
});
```
