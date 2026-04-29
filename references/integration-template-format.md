# Format d'une reference d'intégration

Structure obligatoire d'un fichier `references/<vendor>.md` dans un hub d'intégrations.

## 9 sections obligatoires

```markdown
# <Vendor> — référence technique

<Phrase d'intro : rôle du vendor dans le projet>

**Docs** : <URL officielle>

## 1. Authentification
<API Key, OAuth, Bearer, etc. — sandbox vs production>

## 2. Opération principale
<Exemple de code complet avec payload, headers, réponse>

## 3. Webhook / callback
<Signature HMAC si applicable, vérification sur body brut, re-validation si pas de signature>

## 4. Statuts / événements
<Liste des events à gérer, idempotence, action DB, job async>

## 5. Pièges spécifiques
<5-10 items : format montants, rate limits, retry policy, format numéros, templates pré-approuvés, conformité>

## 6. Test
<Exemple de test avec mock HTTP (nock/responses/webmock selon le langage)>

## 7. Intégration queue
<Config du queue system pour ce vendor : rate limiter, retry, backoff>

## 8. Contexte tenant dans les workers
<Rappel : rejouer le contexte tenant dans le handler async>

## 9. Runbook opérationnel (optionnel)
<Logs, rotation secrets, SLA, contacts support, test staging>
```

## Vendors peu connus

Si Claude ne connaît pas bien l'API : générer le squelette complet avec `TODO:` dans les sections 2, 3, 5. Signaler en Phase 4 : "reference inférée, à relire avant la première utilisation".
