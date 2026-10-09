# CLAUDE.md

Blog personale di Enrico Aquilano: https://EnrAqu.github.io

## Scope

- Post tecnici su **Azure, DevOps, .NET** (e dintorni: SQL, IaC, PowerShell).
- La fonte principale sono **casi reali di lavoro**: problemi incontrati sul campo, come sono stati analizzati e risolti.
- Niente post generici "cos'è X": ogni post parte da un problema concreto.

## Stile dei post

- **Lingua: italiano.** Termini tecnici in inglese dove è naturale (deploy, app settings, timeout…).
- **Tono chill**, colloquiale, mai noioso: qualche battuta, qualche emoji (☕ 🙃 🔍 ✅ ⚠️), senza esagerare.
- **Struttura fissa** per i post da casi reali:
  1. intro breve con il gancio;
  2. `## Il problema`
  3. `## L'analisi`
  4. `## La soluzione`
  5. `## Gli effetti`, a punti con ✅ (benefici) e ⚠️ (trade-off, cose da sapere);
  6. `## TL;DR`, come callout `{: .prompt-tip }`.
- Usa le **feature di Chirpy** per spezzare il testo:
  - callout `{: .prompt-tip }`, `{: .prompt-warning }`, `{: .prompt-info }`, `{: .prompt-danger }` sotto un blockquote;
  - diagrammi **Mermaid** (`mermaid: true` nel front matter) per catene causali e flussi;
  - tabelle per metriche e confronti;
  - snippet di codice reali ma generalizzati (bash/az, bicep, sql, csharp).
- **Onestà tecnica**: non inventare risultati, numeri o default non verificati. Se qualcosa non è stato confermato, dirlo ("non do per scontato…").

## Anonimizzazione (obbligatoria)

I casi vengono da progetti di clienti. Nei post **non** devono comparire:
- nomi di clienti o progetti (es. Stella, Bvg), nomi di resource group, app, registry, database, stored procedure;
- tag di immagini, ID, URL interni, credenziali, region specifiche se non necessarie.

Usa placeholder (`<nome-function-app>`, `<nome-db>`) o descrizioni generiche ("la stored procedure dei prodotti").

## Front matter

```yaml
---
title: "Titolo del post"
date: YYYY-MM-DD 00:00:00 +0200   # +0100 in inverno
categories: [Azure, <Sottocategoria>]
tags: [tag-in-kebab-case, azure-dal-campo]
mermaid: true   # solo se il post ha diagrammi
---
```

- File: `_posts/YYYY-MM-DD-slug-in-italiano.md`. Il permalink è `/posts/:title/`, cioè lo slug del file senza la data.
- Il tag **`azure-dal-campo`** identifica la serie di post nati da incidenti reali.

## Sito: tecnologia e personalizzazioni

- Jekyll + tema **Chirpy 7.4** (gem `jekyll-theme-chirpy`). Gli asset statici vengono dal submodule `assets/lib`.
- `lang: it-IT`, `timezone: Europe/Rome`. Tagline e description sono in italiano in `_config.yml`.
- **Override del tema** (se aggiorni Chirpy, controlla che siano ancora allineati):
  - `_includes/footer.html`: copia del footer originale, senza la riga "Servizio offerto da Jekyll con tema Chirpy";
  - `assets/css/jekyll-theme-chirpy.scss`: stili custom. Contiene `#sidebar .profile-wrapper { flex-shrink: 0; }` per evitare che la tagline si sovrapponga al menu.
- Commenti: **Giscus** sulle Discussions della repo, categoria *Announcements*, mapping `pathname`, lingua `it`.
- Tab: About (`_tabs/about.md`), Progetti (`_tabs/projects.md`, scritta a mano dai repo pubblici di GitHub), più quelle standard.
- `Z-Command.MD` contiene appunti locali ed è escluso dalla build.

## Build, verifica, deploy

```bash
bundle exec jekyll s                     # anteprima locale
JEKYLL_ENV=production bundle exec jekyll b
bundle exec htmlproofer _site --disable-external \
  --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

- Il deploy è automatico: un push su `main` lancia `.github/workflows/pages-deploy.yml` (build, htmlproofer, GitHub Pages).
- Prima di un commit, esegui sempre build + htmlproofer.
- Commit e push solo su richiesta esplicita.

## Idee per i prossimi post

Dai casi reali delle ultime settimane, non ancora scritti:
- **Dry-run mode** in un'integrazione con Azure Functions: invece di scrivere sul sistema esterno, salva payload e parametri della stored procedure su file, con percorso configurabile.
- **Resilienza di una Function HTTP con DB dietro**: retry sugli errori transitori, timeout, `host.json` (`maxConcurrentRequests` e simili).
- **`-WhatIf` negli script PowerShell di deploy Bicep**: post breve, "pillola".
- **Verso il secretless**: Function verso ACR e Storage con managed identity, outbound su VNet, permessi per GitHub Actions.
- **Workbook di Application Insights** con parametri (intervallo temporale, filtri per tabella).

Per trovarne altri: le sessioni di Claude Code sono in `~/.claude/projects/` (file `.jsonl`, una cartella per progetto).

## Da fare

- Avatar: oggi è l'identicon di GitHub (`https://github.com/EnrAqu.png`). Va caricata una foto su GitHub, oppure messa in `assets/img/`.
- Analytics: GoatCounter non è configurato (serve l'ID).
- Immagini di anteprima per i post (`image:` nel front matter).
- `_tabs/about.md`: bozza da rivedere. `_tabs/projects.md`: manca la descrizione di *Utilities*.

## Registro dei post

| Data | Post | Tema |
|---|---|---|
| 2025-11-29 | Hello World | primo post |
| 2025-12-25 | Backpressure di Natale | backpressure con Azure Service Bus, metafora della fabbrica di Babbo Natale |
| 2026-09-25 | Il redeploy che ti cancella le app settings | Bicep incrementale + `siteConfig.appSettings` = lista sostituita; fix con `union()` |
| 2026-10-02 | Il database era lento ma i query plan erano perfetti | timeout -2 su Function + Azure SQL; il DB era innocente, colpa di payload e piano Consumption |
| 2026-10-09 | WEBSITES_ENABLE_APP_SERVICE_STORAGE | Function su immagine ACR: il mount su `/home` copre `wwwroot`, 0 function caricate |

Aggiorna questa tabella a ogni nuovo post.
