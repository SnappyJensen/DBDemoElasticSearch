# 2. Opsætningsdemo — Elasticsearch

Denne guide viser installation og konfiguration af Elasticsearch med Kibana som browserinterface. Følg trinene i rækkefølge. Til sidst kan du søge i tre produkter direkte fra Kibanas Dev Tools Console. Et valgfrit Node.js-eksempel viser også integration i et lille program.

Guiden tager udgangspunkt i **Windows og PowerShell**. Repositoryet indeholder `compose.yaml`. Filen `soeg.mjs` er kun nødvendig, hvis du vil følge det valgfrie programeksempel.

## 1. Gør computeren klar

Du skal have:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) med WSL 2 og Linux-containere. Start programmet, og vent, til Docker er klar.
- En editor, eksempelvis Visual Studio Code.

Docker skal have mindst 4 GB hukommelse til rådighed. Elasticsearch-containeren får en grænse på 1 GB, og Kibana får 2 GB. Se [Elastics Docker-vejledning](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-basic).

[Node.js](https://nodejs.org/en/download) version 22 eller nyere er kun nødvendig til det valgfrie programeksempel.

Åbn PowerShell i **repositoryets mappe `DBDemoElasticSearch`**, eksempelvis via editorens terminal. Alle følgende kommandoer køres derfra i samme terminal.

Kontrollér værktøjerne:

```powershell
docker version
docker compose version
```

**Kontrolpunkt:** `docker version` skal vise både `Client` og `Server`. Hvis `Server` mangler, skal Docker Desktop startes, før du fortsætter.

## 2. Opret konfigurationen

### Lokale adgangskoder

Kibanas Elasticsearch-hjemmeside forventer brugerroller. Derfor bruger denne opsætning login i stedet for den tidligere demo uden login. De eksisterende produktdata kan genbruges.

Opret `.env` én gang fra PowerShell i repositoryets mappe. Blokken genbruger filen, hvis den allerede findes:

```powershell
if (Test-Path -LiteralPath '.env') {
    Write-Output '.env findes allerede og bliver bevaret.'
} else {
    $names = @(
        'ELASTIC_PASSWORD',
        'KIBANA_SYSTEM_PASSWORD',
        'KIBANA_SECURITY_ENCRYPTION_KEY',
        'KIBANA_SAVED_OBJECTS_ENCRYPTION_KEY',
        'KIBANA_REPORTING_ENCRYPTION_KEY'
    )
    $lines = foreach ($name in $names) {
        $bytes = New-Object byte[] 16
        $rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
        try { $rng.GetBytes($bytes) } finally { $rng.Dispose() }
        $value = -join ($bytes | ForEach-Object { $_.ToString('x2') })
        '{0}={1}' -f $name, $value
    }
    [System.IO.File]::WriteAllLines((Join-Path (Get-Location) '.env'), [string[]]$lines)
}
```

`.env` er ignoreret af Git. Del og commit aldrig filen. `.env.example` dokumenterer blot variabelnavnene. Behold de genererede værdier; ændringer i `ELASTIC_PASSWORD` alene ændrer ikke adgangskoden i en allerede initialiseret database.

### Docker Compose

Filen **`compose.yaml` i repositoryets rodmappe** indeholder følgende. Hvis du følger guiden i et nyt, tomt projekt, kan du selv oprette filen med dette indhold:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.5.6
    environment:
      discovery.type: single-node
      cluster.name: eksamensdemo
      node.name: es-demo
      xpack.security.enabled: "true"
      xpack.security.autoconfiguration.enabled: "false"
      xpack.security.http.ssl.enabled: "false"
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:?Create .env using the setup guide first}
    ports:
      - "127.0.0.1:9200:9200"
    mem_limit: 1g
    volumes:
      - esdata:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS --user \"elastic:$${ELASTIC_PASSWORD}\" http://localhost:9200 > /dev/null || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

  setup-kibana:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.5.6
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:?Create .env using the setup guide first}
      KIBANA_SYSTEM_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:?Create .env using the setup guide first}
    entrypoint: ["/bin/bash", "-euc"]
    command:
      - |
        curl --fail --silent --show-error \
          --user "elastic:$${ELASTIC_PASSWORD}" \
          --header 'Content-Type: application/json' \
          --request POST \
          --data "{\"password\":\"$${KIBANA_SYSTEM_PASSWORD}\"}" \
          http://elasticsearch:9200/_security/user/kibana_system/_password
    mem_limit: 256m

  kibana:
    image: docker.elastic.co/kibana/kibana:9.5.6
    depends_on:
      elasticsearch:
        condition: service_healthy
      setup-kibana:
        condition: service_completed_successfully
    environment:
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
      ELASTICSEARCH_USERNAME: kibana_system
      ELASTICSEARCH_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:?Create .env using the setup guide first}
      XPACK_SECURITY_ENCRYPTIONKEY: ${KIBANA_SECURITY_ENCRYPTION_KEY:?Create .env using the setup guide first}
      XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY: ${KIBANA_SAVED_OBJECTS_ENCRYPTION_KEY:?Create .env using the setup guide first}
      XPACK_REPORTING_ENCRYPTIONKEY: ${KIBANA_REPORTING_ENCRYPTION_KEY:?Create .env using the setup guide first}
      SERVER_HOST: "0.0.0.0"
    ports:
      - "127.0.0.1:5601:5601"
    mem_limit: 2g
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:5601/api/status > /dev/null || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 24
      start_period: 60s

volumes:
  esdata:
```

Brug mellemrum til indrykning, og kontrollér, at filen ikke får endelsen `.txt`.

De vigtigste valg:

| Indstilling | Betydning |
| --- | --- |
| `image` | En fast version, så gruppen bruger samme udgave. |
| `discovery.type: single-node` | Ét Elasticsearch-medlem, kaldet en node, til demoen. |
| `127.0.0.1:9200:9200` | REST API'et er tilgængeligt på din egen computer på port 9200. |
| `xpack.security.enabled: "true"` | Aktiverer login og brugerroller, som Elasticsearch-hjemmesiden i Kibana forventer. |
| `mem_limit` | Begrænser containerens hukommelse til 1 GB. |
| `esdata` | Gemmer data i en Docker-volume, så de bevares, når containeren fjernes. |
| `healthcheck` | Kontrollerer, at Elasticsearch svarer, før opstarten meldes klar. |
| `kibana` | Browserinterfacet på port 5601. Bruger samme version som Elasticsearch. |
| `ELASTICSEARCH_HOSTS` | Kibana kontakter Elasticsearch via servicenavnet på Dockers interne netværk. |
| `depends_on` | Kibana starter, når Elasticsearch har bestået sit sundhedstjek. |
| `setup-kibana` | Engangsopgave, der sætter adgangskoden for Kibanas servicekonto, inden Kibana starter. |
| `XPACK_*_ENCRYPTIONKEY` | Faste lokale nøgler, som Kibana bruger til blandt andet sessioner og gemte objekter. |

Elasticsearch kan konfigureres gennem miljøvariabler i containeren; se [Elastics konfigurationsvejledning](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-configure).

## 3. Start Elasticsearch og Kibana

Kontrollér først konfigurationen:

```powershell
docker compose config --quiet
```

Ingen output betyder, at konfigurationen er gyldig. Start derefter:

```powershell
docker compose up -d --wait --wait-timeout 300
docker compose ps
```

Første gang downloades begge images, hvilket kan tage flere minutter. `-d` kører containerne i baggrunden, og `--wait` venter på, at de er klar. Hvis Elasticsearch allerede kører fra tidligere trin, starter kommandoen den nye Kibana-tjeneste ved siden af.

**Kontrolpunkt:** Begge containere skal stå som `healthy`.

Kontrollér også forbindelsen fra PowerShell:

```powershell
$esUrl = 'http://localhost:9200'
$credential = Get-Credential -UserName 'elastic' -Message 'Brug ELASTIC_PASSWORD fra din lokale .env-fil'
$auth = '{0}:{1}' -f $credential.UserName, $credential.GetNetworkCredential().Password
$basicAuth = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($auth))
$headers = @{ Authorization = "Basic $basicAuth" }
Invoke-RestMethod -Uri $esUrl -Headers $headers | ConvertTo-Json -Depth 5
```

Svaret skal blandt andet indeholde:

```json
{
  "name": "es-demo",
  "cluster_name": "eksamensdemo",
  "version": {
    "number": "9.5.6"
  }
}
```

Der er flere felter i det rigtige svar. Du kan også åbne [localhost:9200](http://localhost:9200) i browseren; browseren vil bede om samme brugernavn og adgangskode. HTTP uden TLS bruges kun til denne lokale demo, og portene er bundet til din egen computer. Se [Elastics minimale sikkerhedsopsætning](https://www.elastic.co/docs/deploy-manage/security/set-up-minimal-security).

## 4. Opret et indeks til demoen

Et **indeks** er en samling af dokumenter. Vores indeks hedder `produkter`. Hvert produkt gemmes som et JSON-dokument.

Kør hele blokken i PowerShell:

```powershell
$indexBody = @'
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "navn": { "type": "text" },
      "beskrivelse": { "type": "text" },
      "pris": { "type": "float" }
    }
  }
}
'@

Invoke-RestMethod -Method Put -Uri "$esUrl/produkter" -Headers $headers -ContentType 'application/json; charset=utf-8' -Body $indexBody
```

**Kontrolpunkt:** Svaret viser `acknowledged: true`.

`mappings` fastlægger felternes datatyper. `text` bruges til fuldtekstsøgning, og `float` til prisen. Vi vælger én shard og nul replikaer, fordi demoen kun har én node; der er dermed ingen ekstra kopi af data på en anden node. Se [Create an index](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-indices-create).

Hvis du allerede har gennemført dette trin, kan du springe det over. Fejlen `resource_already_exists_exception` betyder, at indekset findes i forvejen.

## 5. Indsæt tre eksempelprodukter

Kør denne blok i PowerShell:

```powershell
$produkter = @(
    @{ id = 1; navn = 'Studiecomputer'; beskrivelse = 'Laptop til studie og programmering'; pris = 5999.0 },
    @{ id = 2; navn = 'Gamingcomputer'; beskrivelse = 'Laptop til gaming og programmering'; pris = 9999.0 },
    @{ id = 3; navn = 'Kontorstol'; beskrivelse = 'Komfortabel stol til kontoret'; pris = 1499.0 }
)

foreach ($produkt in $produkter) {
    $body = @{
        navn = $produkt.navn
        beskrivelse = $produkt.beskrivelse
        pris = $produkt.pris
    } | ConvertTo-Json

    $uri = "$esUrl/produkter/_doc/$($produkt.id)?refresh=wait_for"
    Invoke-RestMethod -Method Put -Uri $uri -Headers $headers -ContentType 'application/json; charset=utf-8' -Body $body
}
```

**Kontrolpunkt:** Du får tre svar med `result: created` ved første kørsel. Ved gentagelse kan der stå `updated`, fordi vi bruger de samme dokument-id'er. Det opretter ikke dubletter.

`refresh=wait_for` venter på, at ændringen bliver synlig for søgning. Så kan næste trin søge i produkterne med det samme. Se [Elastics forklaring af refresh](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter).

## 6. Brug Elasticsearch i browseren med Kibana

Åbn [Kibana](http://localhost:5601) eller gå direkte til [Dev Tools Console](http://localhost:5601/app/dev_tools#/console).

Det er dit lokale Elastic-interface. Log ind med brugernavnet **`elastic`** og værdien af **`ELASTIC_PASSWORD`** fra din lokale `.env`-fil. Du behøver ikke en Elastic Cloud-konto eller en prøveperiode. Brugeren `kibana_system` bruges kun af Kibana i baggrunden og skal ikke bruges til browserlogin. Port 9200 er Elasticsearchs API; port 5601 er Kibanas brugerflade. De eksisterende produkter bliver liggende i det samme indeks.

Hvis en velkomstside vises, vælg at udforske egne data, eksempelvis **Explore on my own**. Find derefter **Dev Tools** via menuen eller søgefeltet øverst, og vælg **Console**. Se [Elastics Console-vejledning](https://www.elastic.co/docs/explore-analyze/query-filter/tools/console).

Indsæt følgende i konsollens venstre side:

```http
GET /produkter/_search
{
  "query": {
    "match_all": {}
  }
}
```

Placér markøren i forespørgslen, og tryk **Ctrl+Enter**, eller klik på afspilningsknappen. JSON-svaret vises til højre. Under `hits.hits` skal du finde de tre produkter.

Prøv derefter en fuldtekstsøgning:

```http
GET /produkter/_search
{
  "query": {
    "multi_match": {
      "query": "programmering",
      "fields": ["navn", "beskrivelse"]
    }
  }
}
```

**Kontrolpunkt:** `hits.total.value` er `2`, og resultaterne indeholder `Studiecomputer` og `Gamingcomputer`. Ændr søgeordet til `kontoret`, og kør igen; det skal give `Kontorstol`.

Du kan også læse et enkelt dokument:

```http
GET /produkter/_doc/1
```

Konsollen sender forespørgslerne til Elasticsearch. Der kræves hverken Node.js eller et separat søgeprogram til denne del af demoen.

### Valgfrit: Se produkterne som en tabel

Åbn **Discover**, og opret en **data view** for indekset `produkter`. Vælg at fortsætte **uden et tidsfelt**, da vores produkter ikke har datoer. Tilføj kolonnerne `navn`, `beskrivelse` og `pris`. Dev Tools bruges til API-forespørgsler; Discover kan bruges til at udforske dokumenterne som en tabel.

### Om kommandoen på Elastics hjemmeside

Elastic tilbyder også `curl -fsSL https://elastic.co/start-local | sh` som en [separat lokal quick-start](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/local-development-installation-quickstart). Den opretter sin egen opsætning med Elasticsearch og Kibana og køres i en Unix-shell, eksempelvis Bash i WSL. Denne guide bruger repositoryets Compose-fil; du skal ikke køre quick-starten oveni. Se også [Kibana i Docker](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-kibana-with-docker).

## Valgfrit: Integrér Elasticsearch i et lille Node.js-projekt

Opret **`soeg.mjs` i repositoryets rodmappe**, ved siden af `compose.yaml`, og indsæt:

```javascript
const soegeord = process.argv.slice(2).join(' ') || 'programmering';

try {
  const response = await fetch('http://localhost:9200/produkter/_search', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${Buffer.from(`elastic:${process.env.ELASTIC_PASSWORD}`).toString('base64')}`,
    },
    body: JSON.stringify({
      query: {
        multi_match: {
          query: soegeord,
          fields: ['navn', 'beskrivelse'],
        },
      },
    }),
    signal: AbortSignal.timeout(10000),
  });

  if (!response.ok) {
    throw new Error(`Elasticsearch svarede ${response.status}: ${await response.text()}`);
  }

  const resultat = await response.json();
  console.log(`Søgning: ${soegeord}`);
  console.log(`Antal fund: ${resultat.hits.total.value}`);
  console.table(resultat.hits.hits.map((hit) => ({
    id: hit._id,
    ...hit._source,
  })));
} catch (error) {
  console.error(`Søgningen fejlede: ${error.message}`);
  process.exitCode = 1;
}
```

Programmet bruger Node.js' indbyggede `fetch`, så du behøver ikke installere npm-pakker. Det sender en HTTP-forespørgsel til Elasticsearch og søger i både navn og beskrivelse. `--env-file=.env` indlæser den lokale adgangskode som en miljøvariabel; adgangskoden skrives ikke i programkoden.

Kør:

```powershell
node --env-file=.env soeg.mjs programmering
```

**Forventet resultat:** To produkter, `Studiecomputer` og `Gamingcomputer`. Rækkefølgen kan variere.

Prøv derefter:

```powershell
node --env-file=.env soeg.mjs kontoret
```

**Forventet resultat:** Ét produkt, `Kontorstol`.

Nu har du demonstreret, at et selvstændigt program kan bruge Elasticsearch gennem REST API'et.

## 7. Stop og start igen

Stop og fjern containeren, mens data bevares i volumen:

```powershell
docker compose down
```

Start igen med de eksisterende data:

```powershell
docker compose up -d --wait --wait-timeout 300
```

Åbn Kibana igen, og kør en søgning i Dev Tools. Du behøver ikke gentage trin 4 og 5. Undlad at tilføje `--volumes` til stopkommandoen, hvis du vil beholde demoens data.

## Forslag til selve fremlæggelsen

Gennemfør hele guiden inden fremlæggelsen, så imaget er downloadet, og data er klar. Selve opsætningsdelen kan derefter tage cirka 3–4 minutter:

1. Vis `compose.yaml`, og forklar version, én node, lokal port og lagring.
2. Kør opstartskommandoen, og vis `docker compose ps`.
3. Vis svaret fra `http://localhost:9200` for at bekræfte forbindelsen.
4. Åbn Kibanas Dev Tools Console, og kør de to søgninger.
5. Forklar forskellen på Elasticsearch som søgemotor og Kibana som browserinterface. Vis eventuelt det valgfrie Node.js-eksempel for at demonstrere integration med et program.

## Hvis noget driller

| Problem | Hvad du gør |
| --- | --- |
| Docker viser kun `Client`, eller forbindelsen til Docker fejler | Start Docker Desktop, vent på at det er klar, og kontrollér, at det bruger Linux-containere. |
| `no configuration file provided` | Åbn terminalen i mappen med `compose.yaml`. |
| Port 9200 er optaget | Stop den anden tjeneste, der bruger porten, eller vælg en anden lokal port og ret adresserne i både kommandoer og program. |
| Opstarten når sin tidsgrænse | Se `docker compose logs --tail 100 elasticsearch`. Ret årsagen, og kør opstarten igen. |
| Containeren stopper med kode 137 | Den kan være løbet tør for hukommelse. Kontrollér Docker-hukommelsen og loggen. Hvis nødvendigt, hæv `mem_limit` til `2g`, og start igen. |
| `resource_already_exists_exception` | Indekset findes allerede. Fortsæt til trin 5. |
| `index_not_found_exception` | Gennemfør trin 4 og 5, før du søger. |
| `fetch failed` | Kontrollér `docker compose ps`, og prøv forbindelseskontrollen i trin 3. |
| `fetch is not defined` | Kontrollér Node-versionen, og installér version 22 eller nyere. |
| Søgningen giver ingen resultater | Gentag trin 5, og prøv de præcise søgeord `programmering` og `kontoret`. |
| Kibana svarer ikke, eller viser at serveren ikke er klar | Kontrollér `docker compose ps`, og se `docker compose logs --tail 100 kibana`. Første opstart kan tage flere minutter. |
| Discover viser ingen produkter | Vælg data view `produkter` uden tidsfelt, og fjern eventuelle filtre. |
| `401 Unauthorized` fra port 9200 | Brug `elastic` og `ELASTIC_PASSWORD` fra `.env`. PowerShell-kommandoerne kræver `$headers` fra trin 3. |
| Elasticsearch-hjemmesiden i Kibana viser `Unable to load page` og en fejl om `roles` eller `includes` | Kontrollér, at sikkerhed er slået til i Compose-filen, genstart tjenesterne, og log ind. Den tidligere opsætning uden login kunne udløse denne fejl i hjemmesiden. |

Hvis loggen specifikt klager over for lav `vm.max_map_count`, kan du på Windows med Docker Desktop og WSL 2 køre:

```powershell
wsl -d docker-desktop -u root -- sysctl -w vm.max_map_count=1048576
docker compose up -d --wait --wait-timeout 300
```

Indstillingen gælder Docker Desktops Linux-miljø og kan skulle sættes igen efter genstart. Se [Elastics vejledning om Docker-systemkrav](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-prod).
