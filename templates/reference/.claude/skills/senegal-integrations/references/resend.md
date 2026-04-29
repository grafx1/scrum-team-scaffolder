# Resend — référence technique

Email transactionnel (reçus de paiement, bulletins PDF, rappels). Simple, moderne, support React Email natif.

**Docs** : https://resend.com/docs

## Authentification

API Key Bearer unique. Séparée dev/prod.

```
RESEND_API_KEY=re_xxxxxxxx
RESEND_FROM_ADDRESS=École <noreply@ton-domaine.sn>
```

## Envoyer un email (HTML ou React Email)

```typescript
// resend.service.ts
import { Resend } from 'resend';
import { BulletinEmail } from '../../../emails/bulletin-email';

@Injectable()
export class ResendService {
  private readonly client: Resend;
  constructor(private readonly config: ConfigService) {
    this.client = new Resend(config.getOrThrow('RESEND_API_KEY'));
  }

  async sendBulletinReady(params: {
    to: string;
    parentName: string;
    studentName: string;
    term: string;
    downloadUrl: string;
  }) {
    const { data, error } = await this.client.emails.send({
      from: this.config.getOrThrow('RESEND_FROM_ADDRESS'),
      to: params.to,
      subject: `Bulletin de ${params.studentName} — ${params.term}`,
      react: BulletinEmail(params),     // composant React Email
      tags: [
        { name: 'category', value: 'bulletin' },
        { name: 'term', value: params.term },
      ],
    });
    if (error) throw new Error(`Resend error: ${error.message}`);
    return data; // { id: 'email_xxx' }
  }
}
```

## React Email

Les emails sont des composants React dans `apps/api/src/emails/` :

```tsx
// emails/bulletin-email.tsx
import { Html, Head, Body, Container, Heading, Text, Button } from '@react-email/components';

export function BulletinEmail({ parentName, studentName, term, downloadUrl }: Props) {
  return (
    <Html>
      <Head />
      <Body style={{ fontFamily: 'Inter, sans-serif', backgroundColor: '#f6f9fc' }}>
        <Container>
          <Heading>Bonjour {parentName},</Heading>
          <Text>
            Le bulletin de {studentName} pour le trimestre {term} est disponible.
          </Text>
          <Button href={downloadUrl} style={{ background: '#00853e', color: '#fff', padding: '12px 24px', borderRadius: 8 }}>
            Consulter le bulletin
          </Button>
        </Container>
      </Body>
    </Html>
  );
}
```

Prévisualisation locale via `react-email dev` : `cd apps/api && pnpm react-email dev`.

## Domaine et DKIM (obligatoire)

**Le domaine d'envoi doit être vérifié par Resend (DKIM + SPF + DMARC)** — sinon les emails partent directement en spam, surtout chez les opérateurs locaux (Orange Sénégal, Free).

Setup (à faire une fois, à l'onboarding infra) :
1. Dashboard Resend → Domains → Add Domain → `ton-domaine.sn`
2. Ajouter les 3 enregistrements DNS fournis (TXT DKIM, TXT SPF, CNAME)
3. Attendre la propagation (jusqu'à 48h) puis cliquer "Verify"
4. Vérifier dans [mxtoolbox](https://mxtoolbox.com) que DKIM + SPF passent
5. Ajouter une policy DMARC progressive : `p=none` → `p=quarantine` → `p=reject` sur 3 mois

Runbook dans `docs/runbooks/email-domain-setup.md`.

## Webhooks (plan payant uniquement)

Les webhooks Resend (`email.sent`, `email.delivered`, `email.bounced`, `email.complained`) sont disponibles à partir du plan Pro. Pour le MVP gratuit, monitoring via le dashboard Resend.

Quand activés, Resend signe les webhooks avec `svix` — utiliser le SDK officiel :

```typescript
import { Webhook } from 'svix';

@Post('webhooks/resend')
async handleWebhook(@Req() req: RawBodyRequest, @Headers() headers: Record<string, string>) {
  const wh = new Webhook(this.config.getOrThrow('RESEND_WEBHOOK_SECRET'));
  const payload = wh.verify(req.rawBody.toString('utf8'), headers);
  // ... dispatch
}
```

## Pièces jointes

Pour les bulletins PDF, deux options :

1. **URL signée R2** dans le corps de l'email (recommandé) — pas de pièce jointe, respect des quotas, tracking possible
2. **Attachment direct** :
   ```typescript
   await resend.emails.send({
     // ...
     attachments: [{ filename: 'bulletin.pdf', content: pdfBuffer }],
   });
   ```
   Limite : 40 MB total par email, 22 MB recommandé (base64).

**Préférer l'URL signée R2** — voir skill `cloudflare-r2-storage`.

## Pièges Resend

- **DKIM non vérifié → spam garanti** chez les FAI sénégalais. À faire avant le premier envoi utilisateur.
- **Plan Free limité** : 100 emails/jour, 3000/mois. Le MVP scolaire dépasse vite si tu envoies bulletins + paiements + rappels. Prévoir le passage au plan Pro.
- **Pas de webhook delivery** en Free → impossible de tracker automatiquement bounces et plaintes. Monitoring dashboard uniquement.
- **Rate limit** : 10 emails/sec en Pro. Pour les envois en masse (rentrée, bulletins trimestriels), utiliser Bull avec `limiter`.
- **React Email version** : bien fixer la version exacte dans `package.json` — l'API change entre versions mineures.
- **Attachments base64** : 22 MB effectifs pour 40 MB base64. Toujours préférer R2.
- **From address** : le domaine du `from` doit matcher le domaine vérifié, sinon rejet.

## Test

```typescript
import { Resend } from 'resend';

jest.mock('resend', () => ({
  Resend: jest.fn().mockImplementation(() => ({
    emails: {
      send: jest.fn().mockResolvedValue({ data: { id: 'email_test' }, error: null }),
    },
  })),
}));

it('envoie le bulletin au parent', async () => {
  await service.sendBulletinReady({ to: 'parent@test.sn', /* ... */ });
  const mock = (Resend as jest.Mock).mock.results[0].value;
  expect(mock.emails.send).toHaveBeenCalledWith(expect.objectContaining({
    to: 'parent@test.sn',
    subject: expect.stringContaining('Bulletin'),
  }));
});
```

## Intégration Bull

```typescript
BullModule.registerQueue({
  name: 'email-outbound',
  limiter: { max: 8, duration: 1000 }, // 8/s sous la limite Pro
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 3000 },
    removeOnComplete: true,
    removeOnFail: 200,
  },
});
```
