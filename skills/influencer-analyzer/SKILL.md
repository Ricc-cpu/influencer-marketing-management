---
name: influencer-analyzer
description: >
  Analizza profili Instagram da OpenSearch per stimare le performance medie ADV
  di un influencer. Usa questa skill ogni volta che l'utente chiede di analizzare
  profili Instagram, stimare performance di influencer, calcolare medie di views
  o engagement rate, identificare collaborazioni passate, o generare una scheda
  profilo per un creator. Trigger anche quando l'utente menziona "analisi profilo",
  "stima performance", "media views", "ER medio", "collaborazioni brand",
  "scheda influencer", o fornisce una lista di URL/username Instagram da valutare.
---

# Influencer Analyzer

Analizza profili Instagram interrogando l'indice `channel_posts` su OpenSearch
(tramite il tool MCP `opensearch_query` di insights-proxy) e produce una scheda
completa con bio editoriale, dati audience, performance ADV e collaborazioni.

## Quando usare questa skill

- L'utente fornisce uno o piu URL Instagram / username e chiede un'analisi
- L'utente chiede stime di performance, media views, ER medio
- L'utente vuole sapere con quali brand ha collaborato un influencer
- L'utente carica un file (CSV/TXT) con una lista di profili da analizzare

## Input accettati

- URL Instagram: `https://www.instagram.com/username/`
- Username con @: `@username`
- Username semplice: `username`
- File CSV o TXT con una colonna di URL/username (uno per riga)

Estrarre lo username da ciascun input con regex: `instagram\.com/([^/?#\s]+)` per URL,
oppure rimuovere `@` e spazi per username diretti. Convertire sempre in lowercase.

---

## Workflow per ciascun profilo

Esegui questi step nell'ordine indicato. Se l'utente fornisce piu profili,
processali tutti e alla fine presenta una tabella riassuntiva.

### Step 1 — Recupera il profilo canale da OpenSearch

Query OpenSearch con il tool `mcp__insights-proxy__opensearch_query`:

```json
{
  "query": {
    "size": 1,
    "query": {
      "bool": {
        "must": [
          {"term": {"join_field": "channel"}},
          {"term": {"channel.name": "<username>"}},
          {"term": {"channel.type": "ig"}}
        ]
      }
    },
    "_source": [
      "channel",
      "audience_followers_genders",
      "audience_followers_ages",
      "audience_followers_geo"
    ]
  }
}
```

Dal risultato estrai:
- `channel.full_name` — nome completo
- `channel.description` — bio Instagram
- `channel.followers` — follower attuali
- `channel.following`, `channel.post_count`
- `channel.is_verified`
- `channel.gender`, `channel.age_group`, `channel.language`
- `channel.geo.country.code`, `channel.geo.city.name`
- `channel.engagement_rate` (decimale, moltiplica x100 per %)
- `channel.like_avg`, `channel.comment_avg`, `channel.play_avg`
- `audience_followers_genders` — distribuzione genere audience
- `audience_followers_ages` — distribuzione eta audience

Se il profilo non viene trovato, segnala all'utente e passa al successivo.

### Step 2 — Classifica il tier

| Follower          | Tier               |
|-------------------|---------------------|
| >= 1.000.000      | Mega-influencer     |
| >= 100.000        | Macro-influencer    |
| >= 10.000         | Mid-tier influencer |
| < 10.000          | Micro-influencer    |

### Step 3 — Bio editoriale (WebSearch)

Cerca sul web informazioni sul profilo usando `WebSearch` con query tipo:
`"<full_name>" OR "@<username>" instagram influencer`

Combina le informazioni trovate online con i dati OpenSearch (genere, eta,
localita, tier, bio Instagram, categorie di contenuto) per scrivere un
**paragrafo di 3-5 frasi in italiano** che descriva:
- Chi e la persona (nome reale, professione/nicchia)
- Che tipo di contenuti crea
- Per che audience (fascia demografica)
- Eventuali riconoscimenti o tratti distintivi

Non limitarti a ricopiare la bio Instagram: scrivi un profilo editoriale informativo.

### Step 4 — Post sponsorizzati e calcolo performance

Query OpenSearch per gli ultimi 30 post sponsorizzati:

```json
{
  "query": {
    "size": 30,
    "query": {
      "bool": {
        "must": [
          {"term": {"join_field": "post"}},
          {"term": {"channel.name": "<username>"}},
          {"term": {"channel.type": "ig"}},
          {"term": {"is_sponsored": true}}
        ]
      }
    },
    "sort": [{"published_at": "desc"}],
    "_source": [
      "post_id", "post_code", "published_at",
      "watch_count", "view_count", "play_count",
      "engagement_rate", "engagement",
      "like_count", "comment_count", "followers",
      "post_type", "mentions", "caption"
    ]
  }
}
```

#### Calcolo views per post

Le views possono trovarsi in campi diversi. Per ogni post usa:

```
views = max(watch_count, view_count, play_count)
```

Prendi il massimo tra i tre campi (alcuni possono essere 0 o assenti).

#### Rimozione outlier con metodo IQR (views)

1. Filtra solo i post con views > 0 (pool)
2. Ordina le views e calcola Q1 (25-esimo percentile) e Q3 (75-esimo percentile)
3. IQR = Q3 - Q1
4. Limite inferiore = Q1 - 1.5 * IQR
5. Limite superiore = Q3 + 1.5 * IQR
6. Mantieni solo i post con views dentro [limite inferiore, limite superiore]
7. Se hai meno di 4 post nel pool, salta il filtraggio (campione insufficiente)

#### Filtro coerenza follower (ER)

Il campo `followers` nei post rappresenta il valore al momento della raccolta dati,
che a volte puo essere inconsistente (es. 68K su un profilo da 2M). Questo distorce
il campo `engagement_rate` calcolato a livello di post.

Dopo il filtro IQR sulle views, applica questo ulteriore filtro:

1. Prendi il valore `channel.followers` dal profilo canale (Step 1) come riferimento
2. Per ogni post rimasto, controlla il campo `followers` del post
3. **Escludi i post dove `followers` < 50% del valore attuale del canale**
   (es. se il canale ha 2M follower, escludi post con followers < 1M)
4. Annota quanti post sono stati esclusi per incoerenza follower

Questo evita che ER calcolati su snapshot errati di follower distorcano la media.

#### Selezione e medie

- Dai post puliti (dopo entrambi i filtri), prendi i **10 piu recenti**
- **Media views** = somma views dei selezionati / numero selezionati
- **Media ER%** = somma engagement_rate dei selezionati / numero selezionati * 100
  (il campo `engagement_rate` e un decimale, es. 0.035 = 3.5%)

Annota quanti outlier views e quanti post con follower incoerenti sono stati rimossi.

### Step 5 — Collaborazioni (ultimi 6 mesi)

Query OpenSearch per i post sponsorizzati degli ultimi 6 mesi:

```json
{
  "query": {
    "size": 100,
    "query": {
      "bool": {
        "must": [
          {"term": {"join_field": "post"}},
          {"term": {"channel.name": "<username>"}},
          {"term": {"channel.type": "ig"}},
          {"term": {"is_sponsored": true}}
        ],
        "filter": [
          {"range": {"published_at": {"gte": "now-6M", "format": "strict_date_optional_time||epoch_millis"}}}
        ]
      }
    },
    "_source": ["post_id", "published_at", "mentions", "caption"]
  }
}
```

Estrai i brand collaborati da due fonti:
1. **Campo `mentions`** (array nested): prendi `mention.name` per ogni mention
2. **@tag nella caption**: estrai con regex `@(\w+)` dal testo della caption

Unisci le due fonti (deduplicando per username lowercase), escludi lo username
del profilo stesso, conta le occorrenze e ordina per frequenza decrescente.
Mostra i top 15 brand.

---

## Formato output

### Per un singolo profilo

Presenta i risultati con queste sezioni:

```
## @username — Nome Completo [badge verificato se is_verified]

**Bio:** [paragrafo editoriale dallo Step 3]

**Dati profilo**
| Campo | Valore |
|-------|--------|
| Follower | 000.000 |
| Tier | Macro-influencer |
| Genere | M/F |
| Eta | 25-34 |
| Localita | Citta, Paese |
| Lingua | IT |
| ER medio canale | 0.00% |

**Audience**
| Genere | % |
|--------|---|
| Uomini | 00% |
| Donne | 00% |

| Fascia eta | % |
|------------|---|
| 18-24 | 00% |
| 25-34 | 00% |
| ... | ... |

**Performance ADV** (ultimi 10 post sponsorizzati puliti, X outlier rimossi su Y totali)
| # | Data | Tipo | Views | ER% |
|---|------|------|-------|-----|
| 1 | 2025-01-15 | reel | 150.000 | 3.2% |
| ... |

| Media Views | Media ER% |
|-------------|-----------|
| 000.000 | 0.00% |

**Collaborazioni** (ultimi 6 mesi, N post analizzati)
| Brand | Post |
|-------|------|
| @brand1 | 5 |
| @brand2 | 3 |
```

### Per piu profili — Tabella riassuntiva

Dopo aver mostrato i dettagli singoli, aggiungi una tabella comparativa:

```
## Riepilogo

| Profilo | Follower | Tier | Media Views ADV | Media ER% | Top brand (6m) |
|---------|----------|------|-----------------|-----------|----------------|
| @user1 | 500K | Macro | 120.000 | 3.5% | @brand1 (5) |
| @user2 | 50K | Mid-tier | 15.000 | 5.2% | @brand2 (3) |
```

---

## Note importanti

- Usa SEMPRE `max(watch_count, view_count, play_count)` per le views, mai un solo campo
- Il campo `engagement_rate` e un decimale (0.035), moltiplicalo per 100 per la percentuale
- Il campo `join_field` e obbligatorio in ogni query: "post" per i contenuti, "channel" per i profili
- Formatta i numeri grandi con separatore delle migliaia (es. 150.000 non 150000)
- Se un profilo non ha post sponsorizzati, segnalalo e mostra comunque i dati profilo e audience
- Per la bio editoriale, non ricopiare la bio Instagram: sintetizza informazioni dal web in un paragrafo originale
