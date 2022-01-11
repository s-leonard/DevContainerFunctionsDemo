# Dev Container Functions Demo

# Create the Environment 

Create Azure resources using the Azure CLI

## Login to Azure CLI

`az login`

## Now run the folowing script in the Integrated terminal

> To launch the terminal, just right click on any file on the left menu and select `Open in Integrated Terminal`

```
# variables
RG_NAME=rgcc$(date +%s%N | md5sum | cut -c1-6)
Location=uksouth
Storage_Name="storage${RG_NAME}"
Function_Name="function${RG_NAME}"
AppInsights_Name="insights${RG_NAME}"
Cosmos_Name="db${RG_NAME}"

# create resource group
az group create --name $RG_NAME --location $Location

# create stroage account
az storage account create --name $Storage_Name --location $Location --resource-group $RG_NAME --sku Standard_LRS

# create functions app for .net with app insights
az functionapp create --resource-group $RG_NAME  --consumption-plan-location $Location --runtime dotnet --functions-version 3.1 --name $Function_Name --storage-account $Storage_Name 

# create cosmos database
az cosmosdb create -n $Cosmos_Name -g $RG_NAME --default-consistency-level Eventual --locations regionName=$Location failoverPriority=0 isZoneRedundant=False --capabilities EnableServerless
```


# Create Function App

Typing `F1` in VS Code allows you to search all available functions within your installed extensions. So we will look for the command to create a function

```
F1 Azure Functions Create Function
```

Use the follow options on the prompts:
- Create Functions Project: yes
- Langauge: c#
- Runtime: .NET Core 3
- Trigger: HTTPtrigger
- Name: default value e.g. HttpTrigger1
- Namespace: default value
- Access Rights: Anonymous


## Test the function

Once the app is created you can start debuging it

```
F5
```

Once running you should be provided with a URL which you can test in your browser


## Register Cosmos in Nuget

In the integrated terminal run the following command

```
dotnet add package Microsoft.Azure.WebJobs.Extensions.CosmosDB
```

# Add Cosmos Setting

Now we will add the CosmosDbConnectionString setting to the deployed function app that exists in Azure (no code deployed yet). 

To do this you will need to find the name of the cosmos database in the Azure Portal, based on the scripts we ran earlier the name will start with `dbrgcc` and will have some random characters following that so its has a unique name

On the Cosmos DB page on the left Menu go to `Settings > Keys` to copy the `PRIMARY CONNECTION STRING`

Now type:

```
F1 Azure Functions: Add New Setting

```

with the options:

- Subscription: choose you your subscription
- Function App: name of your app it will start with `functionrgcc`
- Type: CosmosDbConnectionString
- Value: the value from the `PRIMARY CONNECTION STRING` of the DB

# Download remote settings

Next we will copy the remote settings down to our local function app whihc will copy the database contion string to the file `....`

run the following command

```
F1 Azure Functions: Download Remote Settings
```

# Create The Cosmos Container

In the Cosmos Database overview page go to `Data Explorer`

The click `New Container`

Use the following values for the form:

- Database id: stockdb
- Container id: stock
- Partition key: /id

# Create a Stock Models

Create a new file called `models.cs` and add the following code to it

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

Now let create a new functions using the command:

`F1 Azure Functions Create New `

- Trigger: HttpTrigger
- Name: CreateStock

Once created go to the file called `CreateStock.cs` and change the code to the following

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

Create file `tests.http`

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

Now lets start deguging the the applications by pressing `F5`

Open the `tests.http` file and hit the send request button above earch request. 

> This is available from the extension already installed. More info here:  https://marketplace.visualstudio.com/items?itemName=humao.rest-client



# Create new Function to Get the Stock

now lets create another functions called `GetStoock` using `F5 Create Function`


Name: GetStoock

Now replace the code with the following: 

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

Now lets add some more tests to the  `tests.http` file

```
###

GET http://localhost:7071/api/stock/TSLA

```

To test the app hit `F5` to debug and go to the `tests.http` file and send the requests

# Deploy to Azure 

use the Azure functions Extension to deploy the function to Azure 









