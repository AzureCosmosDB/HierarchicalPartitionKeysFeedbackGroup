# Overview

Welcome to the private preview of hierarchical partition keys — also known as subpartitioning — in Azure Cosmos DB. This repo contains onboarding instructions, documentation, and a link to the latest SDK versions. 

# Getting started

- Sign-up here to get your Cosmos DB account(s) enabled with the feature: https://aka.ms/cosmos-subpartitioning-signup. 
  - You'll receive an email confirmation when the feature has been enabled on your account, within 5 business days. 
- Find the latest preview version of the supported SDK:
    - [.NET V3 SDK](https://www.nuget.org/packages/Microsoft.Azure.Cosmos/) - version 3.17.0-preview (or a higher preview version)
    - [Java V4 SDK](https://mvnrepository.com/artifact/com.azure/azure-cosmos) - version 4.16.0-beta (or a higher beta version)

# Feature overview

Azure Cosmos DB distributes your data across logical and physical partitions based on your partition key to enable horizontal scaling. With subpartitoning, you can now configure up to 3 levels of hierarchical partition keys to further optimize the data distribution and enable higher scale. 

If you use synthetic keys today or have scenarios where partition keys can exceed 20GB of data, hierarchical partition keys can help. With this feature, logical partition key prefixes can exceed 20GB and 10k RU/s, and queries by prefix are efficiently routed to the subset of partitions with the data.

## Example scenario for hierarchical partition keys

Suppose you have a multi-tenant scenario where you store event (e.g. login, clickstream, payment, etc) information for users in each tenant. Some tenants are very large with thousands of users, while the majority are smaller with a few users. Partitioning by /TenantId may lead to exceeding Cosmos DB's 20GB storage limit on a single logical partition, while partitioning by /UserId will make all queries on a tenant cross-partition. Using a synthetic partition key that combines TenantId and UserId adds complexity to the application and queries for a tenant will still be cross-partition, unless all users are specified. 

With hierarchical partition keys, we can partition first on TenantId, then UserId. When a single tenant exceeds 20GB of storage, Cosmos DB will automatically split the underlying physical partition so that roughly half of the data for the tenant's users will be on one physical partition, and half on the other. Effectively, this means that a single TenantId can exceed 20GB of data. 

Queries that specify either the TenantId, or both TenantId and UserId will be efficiently routed to the subset of physical partitions that contain the relevant data, avoiding a full fanout query. For example, if the container had 1000 physical partitions, but a particular TenantId was only on 5 of them, the query would only be routed to the 5 relevant physical partitions. 
# FAQ
- Is there a storage limit on the size of a logical partition key?
    - Yes. Just like in Cosmos DB today, the logical partition size is still limited to 20GB. However, with hierarchical partition keys, the logical partition is now the entire partition key path. For example, if you partitioned by TenantId -> UserId, an example logical partition would be "Contoso_Alice". This means you can have 20GB of data where the partition key value is "Contoso_Alice". The amount of storage allowed for data in "Contoso" is effectively 20GB * number of unique UserIds for the tenant "Contoso."
- Are there any changes to storage and RU/s limits on physical partitions? 
    - No. Just like in Cosmos DB today, a physical partition can hold 50 GB of storage and serve up to 10,000 RU/s. However, with hierarchical partition keys, if data for a particular partition key prefix - e.g. TenantId - are in multiple physical partitions, this means that the total RU/s achievable for a single TenantId can exceed 10,000 RU/s.
- What happens if I query and only specify a partition key in the "middle" of the path?
    - This will still be a cross-partition query. For example, if you partition by TenantId -> UserId, and provide only the UserId in the query, this will fanout to all physical partitions. To have an efficiently routed query, you must provide either the TenantId (query will go to all physical partitions containing the TenantId data), or both the TenantId and UserId (query will go to the single physical partition containing the TenantId and the specific UserId). 
- Do I have to create a new property in my documents to use this feature?
    - No. Simply specify the hierarchy of partition key paths you want to use during container creation. For example, if you partition by TenantId -> UserId, you do not need to create a new property with these values concatenated. Simply ensure that each document has a property TenantId and a property UserId. See the [code examples](#sample-code).

# Limitations / known issues in the preview 

- Your Cosmos DB account must be [enrolled in the preview](https://aka.ms/cosmos-subpartitioning-signup) in order to use this feature. Make sure that you have signed-up and received confirmation your account has been onboarded. 
- Working with containers that use hierarchical partition keys is supported only in the preview versions of the .NET V3 and Java V4 SDK. You must use the supported SDK to create new containers with hierarchical partition keys and to perform CRUD/query operations on the data. 
    - When issuing queries from the SDK, passing in a partition key in `QueryRequestOptions` is not currently supported. You must specify the partition key paths in the query text itself.
    - Support for Portal, RP, PowerShell, and CLI, and other SDK languages is planned and not yet available. 
- In the Data Explorer in the portal, you currently are not be able to view the documents in a container with hierarchical partition keys. You can read or edit these documents with the supported .NET V3 or Java V4 SDK version.
- You can specify up to 3 hierarchical partition keys. 
- Hierarchial partition keys can currently only be enabled on new containers. The desired partition key paths must be specified upfront at the time of container creation and cannot be changed later. 
- Hierarchical partition keys is currently supported only for SQL API accounts (API for MongoDB and Cassandra API are not currently supported).


# Sample code

### Create new container with hierarchical partition keys

#### .NET V3 SDK
```csharp
// List of partition keys, in hierarchical order. You can have up to three levels of keys.
List<string> subpartitionKeyPaths = new List<string> { "/TenantId", "/UserId", "/TransactionId" };
// Get reference to database that container will be created in
Database database = await cosmosClient.GetDatabase("DatabaseName");
// Create container - Subpartitioned by TenantId -> UserId -> SessionId
ContainerProperties containerProperties = new ContainerProperties(id: "ContainerName", partitionKeyPaths: subpartitionKeyPaths);
container = await database.CreateContainerAsync(containerProperties, throughput: 400);
```

#### Java V4 SDK
```java
// List of partition keys, in hierarchical order. You can have up to three levels of keys.
 List<String> partitionKeyPaths = new ArrayList<String>();
        partitionKeyPaths.add("/tenantId");
        partitionKeyPaths.add("/userId");
        partitionKeyPaths.add("/sessionId");
 //Create a partition key definition object with Kind("MultiHash") and Version V2
 PartitionKeyDefinition subpartitionKeyDefinition = new PartitionKeyDefinition();
        subpartitionKeyDefinition.setPaths(partitionKeyPaths);
        subpartitionKeyDefinition.setKind(PartitionKind.MULTI_HASH);
        subpartitionKeyDefinition.setVersion(PartitionKeyDefinitionVersion.V2);       
// Create container - Subpartitioned by TenantId -> UserId -> SessionId
CosmosContainerProperties containerProperties = new CosmosContainerProperties(containerName, subpartitionKeyDefinition);
ThroughputProperties throughputProperties = ThroughputProperties.createManualThroughput(400);
Mono<CosmosContainerResponse> containerIfNotExists = database.createContainerIfNotExists(containerProperties, throughputProperties);
```

### Add an item to a container 
#### .NET V3 SDK

```csharp
public class PaymentEvent
{
    public string id { get; set; }
    public string TenantId { get; set; }
    public string UserId { get; set; }
    public string Status { get; set; }
}
// Get a reference to the container
Container container = cosmosClient.GetDatabase("DatabaseName").GetContainer("ContainerSubpartitionByTenantId_UserId");
// Create new item
PaymentEvent sampleEvent = new PaymentEvent()
{
    id = Guid.NewGuid().ToString(),
    TenantId = "Contoso",
    UserId = "Alice",
    Status = "Complete"
};
// Option 1: Specify the full partition key path when creating the item (recommended for best performance)
PartitionKey partitionKey = new PartitionKeyBuilder()
            .Add(sampleEvent.TenantId)
            .Add(sampleEvent.UserId)
            .Build();
ItemResponse<PaymentEvent> itemResponse1 = await container.CreateItemAsync(sampleEvent, partitionKey);
// Option 2: Pass in the object and the SDK will automatically extract the full partition key path
ItemResponse<PaymentEvent> itemResponse2 = await container.CreateItemAsync(sampleEvent);
```

#### Java V4 SDK

```java
// Option 1: Pass in the object and the SDK will automatically extract the full partition key path

 UserSession userObject = new UserSession();
 userObject.setTenantId("Microsoft");
 userObject.setUserId("1");
 userObject.setSessionId("0000-11-0000-1111");
 userObject.setId(UUID.randomUUID().toString());
                
Mono<CosmosItemResponse<UserSession>> createdUserObject = container.createItem(userObject);

// Option 2: Specify the full partition key path when creating the item (recommended for best performance)
PartitionKey partitionKey = new PartitionKeyBuilder()
            .add(userObject.getTenantId())
            .add(userObject.getUserId())
            .add(userObject.getSessionId())
            .build();
            
Mono<CosmosItemResponse<UserSession>> createdUserObject = container.createItem(userObject, partitionKey);

```

### Perform a key/value lookup (point read) of an item
Suppose we partition by TenantId and UserId. The Id property is a Guid that represents a unique document transaction Id. To do a key value lookup (point read) on a single item, pass in the id of the item and the full value of the partition key path.  
#### .NET V3 SDK

```csharp
// Get the full partition key path
var id = "0a70accf-ec5d-4c2b-99a7-af6e2ea33d3d"; 
var fullPartitionkeyPath = new PartitionKeyBuilder()
        .Add("Contoso") //TenantId
        .Add("Alice") //UserId
        .Build();
var itemResponse = await containerSubpartitionByTenantId_UserId.ReadItemAsync<dynamic>(id, fullPartitionkeyPath);
```

#### Java V4 SDK
```java
String id = "0a70accf-ec5d-4c2b-99a7-af6e2ea33d3d"; 
PartitionKey partitoinKey = new PartitionKeyBuilder()
    .add("Microsoft")
    .add("1")
    .add("0")
    .build();
Mono<CosmosItemResponse<UserSession>> asyncItemResponseMono = container.readItem(userSession.getId(), partitionKey, UserSession.class);
```

### Run a query
The SDK code to run a query on a subpartitioned container is identical to running a query on a non-subpartitioned container. 
When the query specifies all values of the partition keys in the `WHERE` filter or a prefix of the key hierarchy, the SDK automatically routes the query to the corresponding physical partitions.  Queries that provide only the "middle" of the hierarchy will be cross partition queries. 
For example, suppose we subpartition by `TenantId` -> `UserId`. 
- The query, `"SELECT * FROM c WHERE c.TenantId = 'Contoso' AND c.UserId = 'Alice'"` will be routed to the single logical and physical partition that contains the data for the specified values of `TenantId` and `UserId`. 
- The query, `"SELECT * FROM c WHERE c.TenantId = 'Contoso'"` will be routed to only the subset of logical and physical partition(s) that contain data for the specified value of `TenantId`. This is a targeted cross-partition query and return data for all users in a tenant.
- The query, `"SELECT * FROM c WHERE c.UserId = 'Alice'"` will be routed to all physical partitions, resulting in a fanout cross-partition query. 

#### .NET V3 SDK
```csharp
// This query will be a single-partition query, as it specifies the full partition key path (TenantId and UserId)
QueryDefinition query1 = new QueryDefinition(
    "SELECT * from c WHERE c.TenantId = @TenantIdInput AND c.UserId = @UserIdInput")
    .WithParameter("@TenantIdInput", "Contoso")
    .WithParameter("@UserIdInput", "Alice");
// This query will be a targeted-partition query, as it specifies the prefix path (TenantId).
QueryDefinition query2 = new QueryDefinition(
    "SELECT * from c WHERE c.TenantId = @TenantIdInput")
    .WithParameter("@TenantIdInput", "Contoso")

List<PaymentEvent> allPaymentEvents = new List<PaymentEvent>();
using (FeedIterator<PaymentEvent> resultSet = container.GetItemQueryIterator<PaymentEvent>(query1))
{
    while (resultSet.HasMoreResults)
    {
        FeedResponse<PaymentEvent> response = await resultSet.ReadNextAsync();
        PaymentEvent resultEvent = response.First();
        Console.WriteLine($"\nFound item with TenantId: {resultEvent.TenantId}; UserId: {resultEvent.UserId};");
        allPaymentEvents.AddRange(response);
    }
}
```

#### Java V4 SDK
```java
 CosmosPagedFlux<UserSession> pagedFluxResponse = container.queryItems(
                "SELECT * FROM t WHERE t.TenantId IN ('Microsoft')", queryOptions, UserSession.class);
 try {

            pagedFluxResponse.byPage(preferredPageSize).flatMap(fluxResponse -> {
                logger.info("Got a page of query result with " +
                        fluxResponse.getResults().size() + " items(s)"
                        + " and request charge of " + fluxResponse.getRequestCharge());

                logger.info("Item Ids " + fluxResponse
                        .getResults()
                        .stream()
                        .map(UserSession::getId)
                        .collect(Collectors.toList()));

                return Flux.empty();
            }).blockLast();

        } catch(Exception err) {
            if (err instanceof CosmosException) {
                //Client-specific errors
                CosmosException cerr = (CosmosException) err;
                cerr.printStackTrace();
                logger.error(String.format("Read Item failed with %s\n", cerr));
            } else {
                //General errors
                err.printStackTrace();
            }
        }
```

# Link to Java SDK Sample
[Java Samples](https://github.com/Azure-Samples/azure-cosmos-java-sql-api-samples)

#### Using ARM templates to create a Hierarchical partition containers.

Please replace the `PartitionKey` object in the CosmosDB container container to match the kind and version values below.

```
"partitionKey": {
            "paths": [
              <Your PartitionKeys>
            ],
            "kind": "MultiHash",
            "Version": 2
          }
```

# Have questions? / Report an issue/bug
Create an issue in this repo with your feedback/issue/bug. 
