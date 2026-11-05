# DBDemoElasticSearch

Lokal Elasticsearch-demo med Kibana som browserinterface.

Start Docker Desktop, og kør følgende fra repositoryets mappe:

```powershell
docker compose up -d --wait --wait-timeout 300
```

- [Kibana](http://localhost:5601)
- [Dev Tools Console](http://localhost:5601/app/dev_tools#/console)
- [Elasticsearch API](http://localhost:9200)
- [Trinvis opsætningsguide](02-OPSAETNINGSGUIDE.md)

Begge tjenester er kun tilgængelige lokalt. Demoen bruger HTTP uden login.

