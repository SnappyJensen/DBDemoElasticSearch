# 2. Opsætningsdemo — Elasticsearch

Denne guide viser installation, konfiguration og integration i et lille projekt. Følg trinene i rækkefølge. Til sidst har du Elasticsearch kørende lokalt og et Node.js-program, der søger i tre produkter.

Guiden tager udgangspunkt i **Windows og PowerShell**. Du opretter selv de to filer, `compose.yaml` og `soeg.mjs`, undervejs.

## 1. Gør computeren klar

Du skal have:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) med WSL 2 og Linux-containere. Start programmet, og vent, til Docker er klar.
- [Node.js](https://nodejs.org/en/download) version 22 eller nyere.
- En editor, eksempelvis Visual Studio Code.

Docker skal have mindst 4 GB hukommelse til rådighed. Elasticsearch-containeren i denne guide får en grænse på 1 GB, hvilket er nok til vores lille tekstsøgningsdemo. Se [Elastics Docker-vejledning](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-basic).

Åbn PowerShell i **repositoryets mappe `DBDemoElasticSearch`**, eksempelvis via editorens terminal. Alle følgende kommandoer køres derfra i samme terminal.

Kontrollér værktøjerne:

```powershell
docker version
docker compose version
node --version
```

**Kontrolpunkt:** `docker version` skal vise både `Client` og `Server`. Hvis `Server` mangler, skal Docker Desktop startes, før du fortsætter.

## 2. Opret konfigurationen

Opret filen **`compose.yaml` i repositoryets rodmappe**, og indsæt:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.5.6
    environment:
      discovery.type: single-node
      cluster.name: eksamensdemo
      node.name: es-demo
      xpack.security.enabled: "false"
    ports:
      - "127.0.0.1:9200:9200"
    mem_limit: 1g
    volumes:
      - esdata:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:9200 > /dev/null || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

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
| `xpack.security.enabled: "false"` | Lokal demo over HTTP uden login. Denne konfiguration er kun til lokal udvikling og fremlæggelse. |
| `mem_limit` | Begrænser containerens hukommelse til 1 GB. |
| `esdata` | Gemmer data i en Docker-volume, så de bevares, når containeren fjernes. |
| `healthcheck` | Kontrollerer, at Elasticsearch svarer, før opstarten meldes klar. |

Elasticsearch kan konfigureres gennem miljøvariabler i containeren; se [Elastics konfigurationsvejledning](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-configure).

## 3. Start Elasticsearch

Kontrollér først konfigurationen:

```powershell
docker compose config --quiet
```

Ingen output betyder, at konfigurationen er gyldig. Start derefter:

```powershell
docker compose up -d --wait --wait-timeout 180
docker compose ps
```

Første gang downloades Elasticsearch-imaget, hvilket kan tage flere minutter. `-d` kører containeren i baggrunden, og `--wait` venter på, at den er klar.

**Kontrolpunkt:** Containeren skal stå som `healthy`.

Kontrollér også forbindelsen fra PowerShell:

```powershell
$esUrl = 'http://localhost:9200'
Invoke-RestMethod -Uri $esUrl | ConvertTo-Json -Depth 5
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

Der er flere felter i det rigtige svar. Du kan også åbne [localhost:9200](http://localhost:9200) i browseren for at se det.

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

Invoke-RestMethod -Method Put -Uri "$esUrl/produkter" -ContentType 'application/json; charset=utf-8' -Body $indexBody
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
    Invoke-RestMethod -Method Put -Uri $uri -ContentType 'application/json; charset=utf-8' -Body $body
}
```

**Kontrolpunkt:** Du får tre svar med `result: created` ved første kørsel. Ved gentagelse kan der stå `updated`, fordi vi bruger de samme dokument-id'er. Det opretter ikke dubletter.

`refresh=wait_for` venter på, at ændringen bliver synlig for søgning. Så kan næste trin søge i produkterne med det samme. Se [Elastics forklaring af refresh](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/refresh-parameter).

## 6. Integrér Elasticsearch i et lille Node.js-projekt

Opret **`soeg.mjs` i repositoryets rodmappe**, ved siden af `compose.yaml`, og indsæt:

```javascript
const soegeord = process.argv.slice(2).join(' ') || 'programmering';

try {
  const response = await fetch('http://localhost:9200/produkter/_search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
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

Programmet bruger Node.js' indbyggede `fetch`, så du behøver ikke installere npm-pakker. Det sender en HTTP-forespørgsel til Elasticsearch og søger i både navn og beskrivelse.

Kør:

```powershell
node soeg.mjs programmering
```

**Forventet resultat:** To produkter, `Studiecomputer` og `Gamingcomputer`. Rækkefølgen kan variere.

Prøv derefter:

```powershell
node soeg.mjs kontoret
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
docker compose up -d --wait --wait-timeout 180
node soeg.mjs programmering
```

Du behøver ikke gentage trin 4 og 5. Undlad at tilføje `--volumes` til stopkommandoen, hvis du vil beholde demoens data.

## Forslag til selve fremlæggelsen

Gennemfør hele guiden inden fremlæggelsen, så imaget er downloadet, og data er klar. Selve opsætningsdelen kan derefter tage cirka 3–4 minutter:

1. Vis `compose.yaml`, og forklar version, én node, lokal port og lagring.
2. Kør opstartskommandoen, og vis `docker compose ps`.
3. Vis svaret fra `http://localhost:9200` for at bekræfte forbindelsen.
4. Vis `soeg.mjs`, og kør de to søgninger.
5. Forklar, at programmet kommunikerer med Elasticsearch via HTTP og JSON.

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

Hvis loggen specifikt klager over for lav `vm.max_map_count`, kan du på Windows med Docker Desktop og WSL 2 køre:

```powershell
wsl -d docker-desktop -u root -- sysctl -w vm.max_map_count=1048576
docker compose up -d --wait --wait-timeout 180
```

Indstillingen gælder Docker Desktops Linux-miljø og kan skulle sættes igen efter genstart. Se [Elastics vejledning om Docker-systemkrav](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-prod).
