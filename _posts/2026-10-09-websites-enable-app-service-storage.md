---
title: "WEBSITES_ENABLE_APP_SERVICE_STORAGE: il booleano che ti nasconde le function"
date: 2026-10-09 00:00:00 +0200
categories: [Azure, Functions]
tags: [azure-functions, acr, docker, app-service, containers, bicep]
---

# WEBSITES_ENABLE_APP_SERVICE_STORAGE: il booleano che ti nasconde le function

Ci sono bug che arrivano uno alla volta, educati. E poi ci sono quelli che si presentano in coppia, uno nascosto dietro l'altro, tipo matrioska. Questo è uno di quelli: una Function App su immagine ACR, un redeploy da Bicep e due problemi impilati. Il secondo l'ha causato un booleano che nemmeno avevo impostato.

## Il problema

Dopo un redeploy, una Function App (container Linux, immagine su ACR) risultava **Running** nel portale, ma `/admin/host/status` rispondeva **503**. Il solito "la luce è verde ma in casa non c'è nessuno".

Scavando è venuto fuori che i guai erano due, uno dopo l'altro.

**Guaio n°1: il container non partiva.** Nel log di Docker:

```
ImagePullUnauthorizedFailure
```

L'immagine esisteva, il tag pure. Semplicemente la app non aveva modo di autenticarsi sul registry: nessuna `DOCKER_REGISTRY_SERVER_*` tra le app settings. Le avevo messe a mano in passato, e il redeploy da Bicep le aveva spazzate via (perché nel Bicep non c'erano mai state).

Rimesse `DOCKER_REGISTRY_SERVER_URL`, `_USERNAME` e `_PASSWORD` (le credenziali admin dell'ACR, volevo l'autenticazione a password) e riavviata la app: il container finalmente parte. `Site started`, `WarmUpProbeSucceeded`, applausi.

**Guaio n°2: il container parte, ma di function nemmeno l'ombra.** `az functionapp function list` restituiva una lista vuota. Il nuovo log diceva che l'host aveva caricato **0 function** e stava usando impostazioni di default invece del `host.json` dell'immagine.

Strano, perché l'immagine era la stessa che qualche minuto prima, in un avvio precedente, le function le caricava eccome.

## L'analisi

Confrontando i log delle run "buone" con quella "cattiva" la differenza era questa: nelle run buone l'host leggeva il `host.json` dell'immagine, in quella cattiva no. Qualcosa stava **montando qualcosa sopra `/home/site/wwwroot`**.

Il Dockerfile copia l'app pubblicata proprio lì (`AzureWebJobsScriptRoot=/home/site/wwwroot`). Se la piattaforma monta uno storage persistente su `/home`, quel mount **copre** quello che l'immagine ha messo in quella cartella. L'host guarda in `wwwroot`, trova il vuoto: niente `host.json`, niente `functions.metadata`, niente function.

Il colpevole è `WEBSITES_ENABLE_APP_SERVICE_STORAGE`, che controlla proprio questo:

- **`true`**: App Service monta uno storage persistente (una share di Azure Storage, condivisa tra le istanze) su `/home`. I file sopravvivono ai restart. Il rovescio della medaglia è che il mount **fa ombra** a tutto quello che l'immagine ha in `/home`.
- **`false`**: nessun mount. `/home` è il filesystem del container, quindi vale quello che c'è nell'immagine. Il filesystem però è effimero: quello che scrivi a runtime sparisce al restart.

Nel mio caso la setting **non era impostata** dopo il redeploy, e il log mostrava il mount persistente in azione. Che cosa faccia esattamente la piattaforma quando il valore manca può dipendere dal tipo di app e dalla piattaforma, quindi non do per scontato un default: l'unica cosa certa è che *non* impostarlo mi ha lasciato con un wwwroot coperto.

## La soluzione

Impostarlo esplicitamente a `false` e riavviare:

```bash
az functionapp config appsettings set \
  --name <nome-function-app> \
  --resource-group <resource-group> \
  --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=false
```

Subito dopo il restart, tutte e 5 le function sono comparse. Prova del nove: una POST con body vuoto a uno degli endpoint ha risposto

```
400 {"error":"The 'correlationid' header is required."}
```

che per una function è il modo gentile di dire "sono viva e rifiuto la tua richiesta sgangherata". 🎉

Costo della scelta: `/home` non è più persistente. Per me va benissimo, perché lo stato di queste function sta in storage account, SFTP e Oracle, e la telemetria va su Application Insights (richiede `APPLICATIONINSIGHTS_CONNECTION_STRING`). Gli unici log che perdo sono quelli di Kudu sotto `/home/LogFiles`; i log di Docker e di startup della piattaforma continuano a funzionare.

## Gli effetti

- ✅ L'host legge il `host.json` dell'immagine e carica le function: quello che c'è in ACR è quello che gira.
- ✅ Comportamento prevedibile tra ambienti: niente file "fantasma" di uno storage persistente che si mette in mezzo.
- ⚠️ **Le fix fatte a mano non sono fix.** Entrambe le correzioni (registry settings e questo flag) stavano solo sulla app. Il prossimo deploy che sovrascrive le app settings le cancella di nuovo. Vanno scritte nel modulo Bicep: il flag come `'false'`, e le credenziali del registry con `listCredentials()` o come parametri secure, mai la password in chiaro in un `.bicepparam`.
- ⚠️ Dopo ogni deploy vale la pena controllare che le setting siano sopravvissute. Fidarsi è bene, `az functionapp config appsettings list` è meglio.

## TL;DR

Function su immagine ACR + `WEBSITES_ENABLE_APP_SERVICE_STORAGE` non a `false` = rischio che lo storage persistente su `/home` copra il tuo `wwwroot`, e l'host parte con **0 function**. Impostalo a `false` esplicitamente, **nel Bicep**, non solo a mano. Il te del prossimo redeploy ringrazia. ☕
