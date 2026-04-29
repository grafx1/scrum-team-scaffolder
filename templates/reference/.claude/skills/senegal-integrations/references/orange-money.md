# Orange Money — référence technique

Deuxième moyen de paiement mobile au Sénégal. API officielle via le portail développeur Orange.

**Docs** : https://developer.orange.com/apis/om-webpay

**Note** : compte développeur validé par Orange requis (délai 1-2 semaines). Ne pas démarrer sans.

## Authentification — OAuth 2.0

Client credentials flow, token à rafraîchir toutes les heures. **Cacher impérativement dans Redis** (TTL 55 min) pour éviter de rate-limiter le endpoint OAuth.

```typescript
// orange-money.service.ts
async getAccessToken(): Promise<string> {
  const cached = await this.redis.get('om:token');
  if (cached) return cached;

  const basic = Buffer
    .from(`${this.clientId}:${this.clientSecret}`)
    .toString('base64');

  const res = await this.http.post(
    'https://api.orange.com/oauth/v3/token',
    'grant_type=client_credentials',
    {
      headers: {
        Authorization: `Basic ${basic}`,
        'Content-Type': 'application/x-www-form-urlencoded',
      },
    },
  );

  await this.redis.setex('om:token', 3300, res.data.access_token); // 55min
  return res.data.access_token;
}
```

## Initier un paiement Web Payment

```typescript
async initWebPayment(params: {
  amount: number;          // en XOF entier
  orderId: string;
  returnUrl: string;
  cancelUrl: string;
  notifUrl: string;        // webhook
}) {
  const token = await this.getAccessToken();

  const res = await this.http.post(
    'https://api.orange.com/orange-money-webpay/dev/v1/webpayment',
    {
      merchant_key: this.merchantKey,
      currency: 'OUV',         // OUV = XOF en sandbox ; XOF en prod
      order_id: params.orderId,
      amount: params.amount,
      return_url: params.returnUrl,
      cancel_url: params.cancelUrl,
      notif_url: params.notifUrl,
      lang: 'fr',
      reference: params.orderId,
    },
    { headers: { Authorization: `Bearer ${token}` } },
  );

  return res.data; // { status, message, pay_token, payment_url, notif_token }
}
```

Rediriger l'utilisateur vers `payment_url` — il saisit son numéro OM, reçoit un code USSD, valide.

## Webhook — PAS de signature HMAC

Orange ne signe **pas** ses webhooks. La sécurité repose sur un aller-retour de validation :

```typescript
@Post('webhooks/orange-money')
async handleWebhook(@Body() body: OmNotification) {
  // 1. Body reçu — ne JAMAIS faire confiance aux valeurs
  const { pay_token, status } = body;

  // 2. Re-valider auprès de l'API Orange avec le token
  const token = await this.omService.getAccessToken();
  const verification = await this.http.get(
    `https://api.orange.com/orange-money-webpay/dev/v1/transactionstatus`,
    {
      params: { order_id: body.order_id, amount: body.amount, pay_token },
      headers: { Authorization: `Bearer ${token}` },
    },
  );

  // 3. Si le statut officiel correspond, alors créditer
  if (verification.data.status === 'SUCCESS') {
    await this.tenantDB(async (tx) => {
      // update transaction status in DB
    });
  }
  return { ok: true };
}
```

**Sans cette re-validation**, n'importe qui peut POSTer sur ton endpoint webhook et déclencher un crédit frauduleux.

## Pièges Orange Money

- **Pas de signature webhook** → re-validation obligatoire, toujours.
- **Montants en XOF entiers**, jamais de décimales.
- **Currency** : `OUV` en sandbox, `XOF` en prod. Variable d'environnement.
- **Délai de validation** : compte développeur validé par Orange en 1-2 semaines. Planifier en amont.
- **Sandbox en ligne** : comptes de test factices fournis par Orange, pas de mock local possible. Toujours ajouter un `nock` pour les tests unitaires.
- **Token OAuth rate-limité** : impératif de le cacher dans Redis sinon blocage rapide.
- **`notif_url` doit être public HTTPS** : impossible de tester en local sans tunnel (ngrok ou cloudflared).
- **`order_id` unique** : Orange rejette les doublons. Générer un UUID interne.
- **Devise `OUV`** en sandbox peut confondre un reviewer → commenter explicitement : `// 'OUV' = alias sandbox pour XOF`.

## Test

Utiliser les numéros OM de test fournis par Orange dans la doc du portail développeur. Pour les tests unitaires, mocker l'endpoint OAuth et l'endpoint webpayment via `nock`.
