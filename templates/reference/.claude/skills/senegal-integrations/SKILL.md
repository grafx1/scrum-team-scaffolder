---
name: senegal-integrations
description: Hub de routing pour les intégrations tierces du marché sénégalais utilisées par le SaaS Éducation — Wave, Orange Money (paiements), Africa's Talking (SMS/USSD), WhatsApp Business Cloud API, Resend (email). À déclencher dès qu'un agent touche à un paiement mobile, un envoi SMS/WhatsApp, un email transactionnel, un webhook signé, ou mentionne "Wave", "Orange Money", "OM", "Africa's Talking", "WhatsApp", "Resend", "MoMo", ou "paiement mobile". Ce fichier est court — il oriente vers la référence détaillée du provider concerné dans `references/`.
---

# Intégrations Sénégal — Hub

Ce skill centralise les **règles communes** et **route** vers la référence détaillée de chaque provider. Pour une feature donnée, lis ce hub puis uniquement le(s) fichier(s) de `references/` pertinents.

## Routing

| Besoin | Fichier à lire |
|---|---|
| Paiement Wave, webhook Wave, HMAC Wave | `references/wave.md` |
| Paiement Orange Money, token OM, webpayment | `references/orange-money.md` |
| Envoi SMS, USSD, sender ID, E.164 | `references/africas-talking.md` |
| Messages WhatsApp, templates Meta, fenêtre 24h | `references/whatsapp-business.md` |
| Email transactionnel, DKIM, React Email | `references/resend.md` |

## Règles communes (toutes intégrations)

### 1. Structure du module

Une intégration = un module NestJS dédié dans `apps/api/src/modules/integrations/<vendor>/` :

```
apps/api/src/modules/integrations/<vendor>/
├── <vendor>.module.ts
├── <vendor>.service.ts          # encapsule les appels HTTP
├── <vendor>.webhook.controller.ts   # endpoint webhook avec signature
├── <vendor>.processor.ts        # Bull processor pour les envois sortants
└── __tests__/
```

**Aucun appel HTTP depuis un controller métier.** Toujours via le service dédié de l'intégration.

### 2. Tout envoi sortant passe par Bull Queue

Jamais d'appel synchrone depuis une requête HTTP utilisateur. Voir skill **`bull-redis-job`** pour le pattern complet : `attempts`, `backoff` exponentiel, `removeOnComplete/Fail`, idempotency key, rejoue du contexte tenant dans le worker. Exemples concrets dans chaque fichier `references/`.

### 3. Webhooks : signature sur le body brut

Les webhooks signés (Wave, WhatsApp) exigent une vérification HMAC sur le **body brut**, pas le JSON déjà parsé. Configurer NestJS avec `bodyParser.raw` sur la route webhook **avant** le JSON parser global :

```ts
// main.ts
app.use('/webhooks/wave', bodyParser.raw({ type: 'application/json' }));
app.use('/webhooks/whatsapp', bodyParser.raw({ type: 'application/json' }));
app.use(bodyParser.json());
```

Toujours utiliser `crypto.timingSafeEqual` pour la comparaison de signatures (prévention timing attacks). Voir fichiers `references/` pour le code exact par provider.

### 4. Secrets via `@nestjs/config`

Jamais de secret en dur. Toujours via `ConfigService` + `.env`. Rotation documentée dans `docs/runbooks/secrets.md`.

```ts
@Injectable()
export class WaveService {
  constructor(private readonly config: ConfigService) {}
  private get apiKey() {
    return this.config.getOrThrow<string>('WAVE_API_KEY');
  }
}
```

### 5. Sandbox vs production

Activer le mode sandbox quand `NODE_ENV !== 'production'` :
- Wave : clés `wave_sn_test_*`
- Orange Money : comptes de test fournis par Orange (pas un mock local)
- Africa's Talking : sandbox partielle — **attention**, certains envois facturent quand même. Mocker en dev local via `nock` ou `msw-node`.
- WhatsApp : numéro de test fourni par Meta dans la Business Platform
- Resend : API key séparée `re_test_*`

### 6. Logs structurés, jamais de secrets

```ts
this.logger.log({
  vendor: 'wave',
  action: 'create_checkout',
  correlationId,
  amount,
  reference,
  // ❌ NE JAMAIS logger : apiKey, webhookSecret, body complet du JWT
});
```

### 7. Preuves / reçus stockés via R2

Reçus PDF, screenshots de confirmation, justificatifs → Cloudflare R2 avec préfixe `schoolId`. Voir skill **`cloudflare-r2-storage`**.

### 8. Base de données via `tenantDB()`

Tout état persisté (transaction_id, webhook_received_at, sms_message_id, etc.) passe par `tenantDB()` pour que le RLS s'applique. Exception : le `TenantResolver` pour onboarding école (voir `keycloak-org-auth`). Voir skill **`drizzle-schema`**.

## Checklist intégration (avant `in_review`)

- [ ] Module NestJS dédié créé, appel HTTP uniquement dans le service du vendor
- [ ] Secrets dans `@nestjs/config`, zéro en dur
- [ ] Body brut lu pour les webhooks HMAC (Wave, WhatsApp)
- [ ] Signature vérifiée avec `timingSafeEqual`
- [ ] Envois sortants via Bull Queue (voir `bull-redis-job`)
- [ ] Contexte tenant rejoué dans le worker via `tenantContext.run()`
- [ ] Mode sandbox actif en dev/test
- [ ] Test unitaire avec mock HTTP (`nock` ou `msw-node`)
- [ ] Test d'intégration contre la sandbox du vendor (job CI séparé, non bloquant)
- [ ] Logs structurés avec correlation id, zéro secret
- [ ] Reçus/preuves stockés via R2 (voir `cloudflare-r2-storage`)
- [ ] Lu la référence détaillée du vendor dans `references/<vendor>.md`

## Livrable

Module vendor complet et testé, webhook signé, processor Bull avec retry et contexte tenant rejoué, test d'intégration sandbox vert.
