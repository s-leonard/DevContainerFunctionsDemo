# DevContainerFunctionsDemo

# Create the Environment


### Login to Azure CLI

`az login`

### Run the folowing script

```
RG_NAME=rgcc$(date +%s%N | md5sum | cut -c1-6)
Location=uksouth
Storage_Name="storage${RG_NAME}"
Function_Name="function${RG_NAME}"
AppInsights_Name="insights${RG_NAME}"
Cosmos_Name="db${RG_NAME}"



az group create --name $RG_NAME --location $Location

az storage account create --name $Storage_Name --location $Location --resource-group $RG_NAME --sku Standard_LRS

az functionapp create --resource-group $RG_NAME  --consumption-plan-location $Location --runtime dotnet --functions-version 3.1 --name $Function_Name --storage-account $Storage_Name 

az cosmosdb create -n $Cosmos_Name -g $RG_NAME --default-consistency-level Eventual --locations regionName=$Location failoverPriority=0 isZoneRedundant=False --capabilities EnableServerless

```


# Create Function App

```
F1 Azure Functions Create
```

# Test the function

```
F5
```

# Register Cosmos in Numget
```
dotnet add package Microsoft.Azure.WebJobs.Extensions.CosmosDB
```

# Add cosmos setting

```
F1 Azure Functions: Add New Setting

Type: CosmosDbConnectionString
Value: ???
```

# Download remote settings

```
F1 Azure Functions: Download Remote Settings
```


# Create a Stock Models

```
using System;
public class CreateStockRequest
{
    public string Code { get; set; }
    public string Name { get; set; }
    public string Market { get; set; }
    public decimal Price { get; set; }
}


public class Stock
{
    public string id { get; set; }
    public string Code { get; set; }
    public string Name { get; set; }
    public string Market { get; set; }
    public decimal Price { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

# Create Function - Create

F1 Azure Functions Create New 

Name: CreateStock

```
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace Company.Function
{
    public static class CreateStock
    {
        [FunctionName("CreateStock")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] CreateStockRequest req,
            [CosmosDB(
            databaseName: "stockdb",
            collectionName: "stock",
            ConnectionStringSetting = "CosmosDbConnectionString")]IAsyncCollector<dynamic> stockDb,
            ILogger log)
        {
            log.LogInformation("Create stock request received");

            if (string.IsNullOrWhiteSpace(req.Code))
                return new BadRequestObjectResult("Code is required");

            var newStock = new Stock()
            {
                id = req.Code,
                Code = req.Code,
                LastUpdated = DateTime.Now,
                Price = req.Price,
                Market = req.Market,
                Name = req.Name
            };

            await stockDb.AddAsync(newStock);
    
            return new OkObjectResult(newStock);
        }
    }
}

```


# Test It Locally

Create file LocalTests/tests.http

Run `F5` to debug

```
POST http://localhost:7071/api/CreateStock HTTP/1.1
content-type: application/json

{
  "Code": "MSFT",
  "Name": "Microsoft Corporation",
  "Market": "Nasdaq",
  "Price": 325
}


###

POST http://localhost:7071/api/CreateStock HTTP/1.1
content-type: application/json

{
  "Code": "TSLA",
  "Name": "Tesla Inc",
  "Market": "Nasdaq",
  "Price": 1040
}

```

# Create new Function to Get the Stock

Name: GetStoock.cs

```
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;


namespace Company.Function
{
    public static class GetStock
    {
        [FunctionName("GetStock")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "stock/{id}")] HttpRequest req, string id,
            [CosmosDB(
                databaseName: "stockdb",
                collectionName: "stock",
                ConnectionStringSetting = "CosmosDbConnectionString",
                Id = "{id}",
                PartitionKey = "{id}")] Stock item,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            if (item == null)
                return new NotFoundResult();
            else
                return new OkObjectResult(item);
        }
    }
}

```

# Add More Tests

```
###

GET http://localhost:7071/api/stock/TSLA

```

# Deploy to Azure 

use the Azure functions Extension









