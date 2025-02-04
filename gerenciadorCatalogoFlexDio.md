1. Criar uma Azure Function para salvar arquivos no Storage Account
Esta função será responsável por receber um arquivo (por exemplo, um JSON contendo informações sobre um filme ou série) e salvá-lo no Azure Storage Account.

Passos:
Criar uma Azure Function:

No portal do Azure, crie uma nova Azure Function App.

Escolha o runtime adequado (por exemplo, .NET, Node.js, Python, etc.).

Crie uma nova função do tipo HTTP Trigger.

Configurar o Storage Account:

No código da função, utilize o SDK do Azure Storage para salvar o arquivo no Storage Account.

Exemplo em C#:
```
using System.IO;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Azure.Storage.Blobs;

public static class SaveToStorage
{
    [FunctionName("SaveToStorage")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
        var blobServiceClient = new BlobServiceClient(Environment.GetEnvironmentVariable("AzureWebJobsStorage"));
        var containerClient = blobServiceClient.GetBlobContainerClient("netflix-catalog");
        await containerClient.CreateIfNotExistsAsync();

        var blobClient = containerClient.GetBlobClient(Guid.NewGuid().ToString() + ".json");
        await blobClient.UploadAsync(new MemoryStream(System.Text.Encoding.UTF8.GetBytes(requestBody)), overwrite: true);

        return new OkObjectResult("File saved successfully.");
    }
}
```
Configurar a conexão com o Storage Account:

No portal do Azure, vá para a Function App e configure a string de conexão do Storage Account nas configurações da aplicação.

2. Criar uma Azure Function para salvar registros no Cosmos DB
Esta função será responsável por receber os dados do arquivo salvo no Storage Account e salvar as informações no Cosmos DB.

Passos:
Criar uma Azure Function:

Crie uma nova função do tipo Blob Trigger que será acionada quando um novo arquivo for salvo no Storage Account.

Configurar o Cosmos DB:

No código da função, utilize o SDK do Cosmos DB para salvar os dados.

Exemplo em C#:
```
using System.IO;
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Cosmos;
using Newtonsoft.Json;

public static class SaveToCosmosDB
{
    private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDBEndpointUri");
    private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDBPrimaryKey");
    private static readonly string DatabaseId = "NetflixCatalog";
    private static readonly string ContainerId = "Movies";

    [FunctionName("SaveToCosmosDB")]
    public static async Task Run(
        [BlobTrigger("netflix-catalog/{name}", Connection = "AzureWebJobsStorage")] Stream myBlob, string name, ILogger log)
    {
        log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");

        var cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);
        var database = cosmosClient.GetDatabase(DatabaseId);
        var container = database.GetContainer(ContainerId);

        using (StreamReader reader = new StreamReader(myBlob))
        {
            string jsonContent = await reader.ReadToEndAsync();
            var movie = JsonConvert.DeserializeObject<Movie>(jsonContent);

            await container.CreateItemAsync(movie, new PartitionKey(movie.Id));
        }
    }
}

public class Movie
{
    public string Id { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public int ReleaseYear { get; set; }
}
```
Configurar a conexão com o Cosmos DB:

No portal do Azure, vá para a Function App e configure as variáveis de ambiente CosmosDBEndpointUri e CosmosDBPrimaryKey com as informações do seu Cosmos DB.

3. Criar uma Azure Function para filtrar registros no Cosmos DB
Esta função será responsável por filtrar registros no Cosmos DB com base em critérios específicos (por exemplo, filtrar filmes por ano de lançamento).

Passos:
Criar uma Azure Function:

Crie uma nova função do tipo HTTP Trigger.
Configurar a consulta no Cosmos DB:

No código da função, utilize o SDK do Cosmos DB para realizar a consulta.

Exemplo em C#:
```
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Cosmos;

public static class FilterMovies
{
    private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDBEndpointUri");
    private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDBPrimaryKey");
    private static readonly string DatabaseId = "NetflixCatalog";
    private static readonly string ContainerId = "Movies";

    [FunctionName("FilterMovies")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        string year = req.Query["year"];
        var cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);
        var database = cosmosClient.GetDatabase(DatabaseId);
        var container = database.GetContainer(ContainerId);

        var query = new QueryDefinition("SELECT * FROM c WHERE c.ReleaseYear = @year")
            .WithParameter("@year", int.Parse(year));

        var iterator = container.GetItemQueryIterator<Movie>(query);
        var results = new List<Movie>();

        while (iterator.HasMoreResults)
        {
            var response = await iterator.ReadNextAsync();
            results.AddRange(response);
        }

        return new OkObjectResult(results);
    }
}
```
Criar uma Azure Function para listar registros no Cosmos DB
Esta função será responsável por listar todos os registros no Cosmos DB.

Passos:
Criar uma Azure Function:

Crie uma nova função do tipo HTTP Trigger.

Configurar a listagem no Cosmos DB:

No código da função, utilize o SDK do Cosmos DB para listar todos os registros.

Exemplo em C#:
```
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Azure.Cosmos;

public static class ListMovies
{
    private static readonly string EndpointUri = Environment.GetEnvironmentVariable("CosmosDBEndpointUri");
    private static readonly string PrimaryKey = Environment.GetEnvironmentVariable("CosmosDBPrimaryKey");
    private static readonly string DatabaseId = "NetflixCatalog";
    private static readonly string ContainerId = "Movies";

    [FunctionName("ListMovies")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        var cosmosClient = new CosmosClient(EndpointUri, PrimaryKey);
        var database = cosmosClient.GetDatabase(DatabaseId);
        var container = database.GetContainer(ContainerId);

        var query = new QueryDefinition("SELECT * FROM c");

        var iterator = container.GetItemQueryIterator<Movie>(query);
        var results = new List<Movie>();

        while (iterator.HasMoreResults)
        {
            var response = await iterator.ReadNextAsync();
            results.AddRange(response);
        }

        return new OkObjectResult(results);
    }
}
```
5. Testar e Publicar
Teste cada função localmente utilizando o Azure Functions Core Tools.

Publique as funções no Azure e configure as permissões necessárias para o Storage Account e Cosmos DB.

6. Integração e Consumo
Você pode integrar essas funções com um front-end (por exemplo, uma aplicação web ou móvel) para permitir que os usuários enviem arquivos, filtrem e listem os registros do catálogo da Netflix.

Considerações Finais
Segurança: Certifique-se de configurar as permissões corretas para o Storage Account e Cosmos DB, e utilize autenticação e autorização nas funções HTTP.

Escalabilidade: Azure Functions é escalável por natureza, mas você pode configurar o Cosmos DB para escalar automaticamente com base na demanda.

Monitoramento: Utilize o Azure Monitor e Application Insights para monitorar o desempenho e a saúde das suas funções.

Com essas etapas, você terá um gerenciador de catálogo da Netflix funcional e escalável utilizando Azure Functions, Storage Account e Cosmos DB.
