# Influencer Marketing Management

Plugin Claude Code per l'analisi di profili Instagram da dati OpenSearch.

## Skills

### influencer-analyzer

Analizza profili Instagram e produce schede complete con:
- Bio editoriale (ricerca web + dati OpenSearch)
- Dati audience (genere, eta, geo)
- Performance ADV medie (views, ER%) con rimozione outlier IQR
- Collaborazioni brand (ultimi 6 mesi)

## Requisiti

- MCP server `insights-proxy` configurato con accesso all'indice `channel_posts` su OpenSearch
- Tool `WebSearch` disponibile per la generazione della bio editoriale

## Installazione

```
claude plugin add <github-url>
```
