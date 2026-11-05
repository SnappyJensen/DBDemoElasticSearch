# DBDemoElasticSearch

Kom i gang med **Elasticsearch** (search engine): data gemmes som JSON-dokumenter i et *index* (≈ tabel) og tilgås via et REST API på port 9200.

## 1. Forudsætninger

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [.NET SDK 8+](https://dotnet.microsoft.com/download)

## 2. Start Elasticsearch

```bash
docker compose up -d
curl http://localhost:9200
```

Svaret indeholder `"tagline" : "You Know, for Search"`, når det kører (tager 30-60 sek. at starte).

> Sikkerhed er slået fra i `docker-compose.yml` for at gøre demoen enkel. Brug det aldrig i produktion.

## 3. CRUD med curl (Git Bash)

```bash
# Create
curl -X PUT localhost:9200/products/_doc/1 -H "Content-Type: application/json" \
  -d '{"name":"Gaming laptop","description":"Kraftig laptop med grafikkort","price":9999.95}'

# Read
curl localhost:9200/products/_doc/1

# Søg (fuldtekst)
curl "localhost:9200/products/_search?q=description:grafikkort"

# Update
curl -X POST localhost:9200/products/_update/1 -H "Content-Type: application/json" \
  -d '{"doc":{"price":8499}}'

# Delete
curl -X DELETE localhost:9200/products/_doc/1
```

Nye dokumenter er søgbare efter ca. 1 sekund (*near real-time*).

## 4. Brug fra .NET

```bash
dotnet new console -n ElasticDemo && cd ElasticDemo
dotnet add package Elastic.Clients.Elasticsearch
```

`Program.cs`:

```csharp
using Elastic.Clients.Elasticsearch;

var client = new ElasticsearchClient(new Uri("http://localhost:9200"));

var product = new Product("1", "Gaming laptop", 9999.95m);

await client.IndexAsync(product, i => i.Index("products").Id(product.Id));        // Create
var get = await client.GetAsync<Product>("1", g => g.Index("products"));          // Read
Console.WriteLine(get.Source?.Name);
await client.UpdateAsync<Product, object>("products", "1", u => u.Doc(new { Price = 8499m })); // Update
await client.DeleteAsync("products", "1");                                        // Delete

record Product(string Id, string Name, decimal Price);
```

Kør med `dotnet run`. (Klientens API ændrer sig mellem versioner - tjek [dokumentationen](https://www.elastic.co/docs/reference/elasticsearch/clients/dotnet) ved fejl.)

## 5. Fejlfinding og oprydning

- Container dør med `vm.max_map_count too low`: `wsl -d docker-desktop sysctl -w vm.max_map_count=262144`
- Se log: `docker compose logs -f`
- Stop og slet: `docker compose down -v`
