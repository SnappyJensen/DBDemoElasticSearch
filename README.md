# DBDemoElasticSearch

Lokal Elasticsearch-demo med Kibana som browserinterface.

Start Docker Desktop. Opret din lokale `.env` én gang med PowerShell-blokken i [opsætningsguiden](02-OPSAETNINGSGUIDE.md), og kør følgende fra repositoryets mappe:

```powershell
docker compose up -d --wait --wait-timeout 300
```

- [Kibana](http://localhost:5601)
- [Elasticsearch-hjemmesiden i Kibana](http://localhost:5601/app/elasticsearch/home)
- [Dev Tools Console](http://localhost:5601/app/dev_tools#/console)
- [Elasticsearch API](http://localhost:9200)
- [Trinvis opsætningsguide](02-OPSAETNINGSGUIDE.md)

Log ind som **`elastic`** med værdien af **`ELASTIC_PASSWORD`** fra din lokale `.env`. Brug ikke `kibana_system` til browserlogin; det er Kibanas servicekonto.

Begge tjenester er kun tilgængelige lokalt. Demoen bruger HTTP med login. `.env` indeholder lokale adgangskoder og er ignoreret af Git.
