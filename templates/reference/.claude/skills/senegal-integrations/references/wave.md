# Wave — référence technique

Paiements mobiles dominants au Sénégal. Intégration via API REST.

**Docs** : https://docs.wave.com/business

## Authentification

API Key en header `Authorization: Bearer <key>`.

- Sandbox : `wave_sn_test_*`
- Production : `wave_sn_prod_*`

Les clés sont dans `.env` :
```
WAVE_API_KEY=wave_sn_test_xxxxxx
WAVE_WEBHOOK_SECRET=whsec_xxxxxx
WAVE_BASE_URL=https://api.wave.com/v1
```

## Créer un checkout

```typescript
// wave.service.ts
async createCheckout(params: {
  amount: number;          // en XOF (franc CFA), unité (pas centimes)
  reference: string;       // ton id interne, unique
  successUrl: string;
  errorUrl: string;
}) {
  const res = await this.http.post(
    `${this.baseUrl}/checkout/sessions`,
    {
      amount: params.amount.toString(),   // ⚠️ string, pas number
      currency: 'XOF',
      error_url: params.errorUrl,
      success_url: params.successUrl,
      client_reference: params.reference,
    },
    { headers: { Authorization: `Bearer ${this.apiKey}` } },
  );
  return res.data; // { id, wave_launch_url, checkout_status, ... }
}
```

Rediriger l'utilisateur vers `wave_launch_url` (app Wave ou navigateur mobile).

## Webhook signé

Header : `Wave-Signature: t=<timestamp>,v1=<hmac>`

Vérification HMAC **sur le body brut** (configurer `bodyParser.raw` sur la route) :

```typescript
import * as crypto from 'crypto';

verifySignature(rawBody: string, header: string): boolean {
  const parts = header.split(',').reduce((acc, p) => {
    const [k, v] = p.split('=');
    acc[k] = v;
    return acc;
  }, {} as Record<string, string>);

  const ts = parts.t;
  const sig = parts.v1;
  if (!ts || !sig) return false;

  // Prévention replay : rejeter les webhooks > 5 min
  if (Math.abs(Date.now() / 1000 - parseInt(ts, 10)) > 300) return false;

  const expected = crypto
    .createHmac('sha256', this.webhookSecret)
    .update(`${ts}.${rawBody}`)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(sig, 'hex'),
    Buffer.from(expected, 'hex'),
  );
}
```

## Statuts à gérer

- `checkout_session.completed` — paiement réussi
- `checkout_session.expired` — utilisateur n'a pas payé à temps
- `checkout_session.cancelled` — utilisateur a annulé

Après réception d'un webhook `completed` signé :
1. Vérifier idempotence via le `checkout.id` (peut arriver 2× en cas de retry Wave)
2. Mettre à jour le statut de la transaction en DB via `tenantDB()`
3. Enqueue un job Bull pour envoyer la confirmation (SMS + email)

## Pièges Wave

- **Body brut obligatoire** pour la signature — un JSON parsé donne un hash différent.
- **Webhooks avant la réponse HTTP du checkout** : Wave peut envoyer un webhook `completed` avant même que ton `POST /checkout/sessions` initial soit terminé. Gérer via une table `wave_transactions` avec UPSERT sur `wave_checkout_id`.
- **Pas de remboursement API** pour le MVP — via dashboard Wave uniquement. Documenter dans le runbook.
- **Montants en XOF unité**, jamais en centimes. `1000` = 1 000 CFA.
- **Environnement test** : utilise des numéros de téléphone Wave de test fournis par le support commercial.
- **Retry webhooks** : Wave retente jusqu'à 24h en cas de 5xx. Toujours renvoyer 200 rapidement, traiter en async via Bull.

## Test

```typescript
// wave.service.spec.ts
import * as nock from 'nock';

describe('WaveService.createCheckout', () => {
  it('crée un checkout avec le bon payload', async () => {
    nock('https://api.wave.com')
      .post('/v1/checkout/sessions', (body) => {
        expect(body.amount).toBe('5000');
        expect(body.currency).toBe('XOF');
        return true;
      })
      .reply(201, { id: 'cos_test', wave_launch_url: 'https://...' });

    const res = await service.createCheckout({
      amount: 5000, reference: 'ref-1',
      successUrl: '/ok', errorUrl: '/err',
    });
    expect(res.id).toBe('cos_test');
  });
});
```
