---
name: cloudflare-r2-storage
description: Gère le stockage de fichiers sur Cloudflare R2 pour le SaaS Éducation Sénégal — bulletins PDF, photos élèves, pièces jointes messagerie — avec SDK compatible S3, signed URLs, lifecycle, région EU pour conformité données mineurs. À déclencher quand l'utilisateur dit "upload sur R2", "signed URL", "stocker les bulletins", ou travaille sur le module storage / fichiers.
type: skill
---

# cloudflare-r2-storage

## Stack de référence

- **Cloudflare R2** (zéro frais d'egress)
- **SDK AWS S3** compatible (`@aws-sdk/client-s3` + `@aws-sdk/s3-request-presigner`)
- Région **EU** imposée pour les données de mineurs (conformité RGPD/CNIL)
- Buckets :
  - `bamtare-bulletins` — bulletins PDF trimestriels
  - `bamtare-photos` — photos d'élèves
  - `bamtare-attachments` — pièces jointes messagerie
- CDN Cloudflare devant les buckets publics (photos uniquement)

## Conventions de nommage

```
bulletins/<school_id>/<academic_year>/<term_id>/<student_id>.pdf
photos/<school_id>/students/<student_id>.jpg
attachments/<school_id>/<thread_id>/<message_id>/<filename>
```

Le préfixe `<school_id>` permet un audit et un purge par tenant.

## Upload côté serveur

```ts
await s3.send(new PutObjectCommand({
  Bucket: 'bamtare-bulletins',
  Key: `bulletins/${schoolId}/${year}/${termId}/${studentId}.pdf`,
  Body: pdfBuffer,
  ContentType: 'application/pdf',
  Metadata: { schoolId, studentId, termId },
  ServerSideEncryption: 'AES256',
}));
```

## Signed URL pour le parent

```ts
const url = await getSignedUrl(s3, new GetObjectCommand({
  Bucket: 'bamtare-bulletins',
  Key: key,
}), { expiresIn: 900 }); // 15 min
```

La validation de l'accès (parent → enfant) se fait **avant** la génération de l'URL, via la policy RLS.

## Lifecycle et rétention

- `bulletins/*` : rétention infinie (archives multi-années)
- `attachments/*` : 2 ans puis purge
- `photos/*` : durée de scolarité + 1 an

## Conventions

- **Jamais** d'URL publique sur `bulletins/*` et `attachments/*` → signed URL uniquement
- Photos élèves : bucket public + CDN, mais **jamais** le nom dans l'URL (uniquement `student_id`)
- Toutes les clés préfixées par `<school_id>` pour l'isolation et la purge
- `ContentType` et `Metadata` obligatoires
- Chiffrement SSE `AES256` sur tous les PUT

## Pièges à éviter

- Région non-EU → non-conformité mineurs
- Signed URL trop longue (>1h) → fuite possible
- Oublier de vérifier le droit d'accès côté RLS avant de signer l'URL
- Exposer un bucket en public par défaut
- Ne pas gérer le retry R2 (erreurs 5xx transitoires)
- Stocker le `schoolId` uniquement dans les metadata sans le mettre dans la clé → pas d'audit simple

## Livrable

Service `StorageService` NestJS avec méthodes `upload(key, buffer, meta)` + `getSignedUrl(key, ttl)` + tests avec R2 mock (S3 local).
