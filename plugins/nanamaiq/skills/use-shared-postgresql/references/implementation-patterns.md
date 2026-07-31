# Implementation patterns

## .NET with Npgsql

Keep the shared connection information and database name separate:

```json
{
  "ConnectionStrings": {
    "SharedPostgres": "Host=<shared-server>;Username=<user>;Password=<password>;Ssl Mode=Require"
  },
  "ProductDatabase": {
    "Name": "product"
  }
}
```

Build a project-owned data source without changing the shared configuration:

```csharp
var sharedConnectionString =
    configuration.GetConnectionString("SharedPostgres")
    ?? throw new InvalidOperationException(
        "ConnectionStrings:SharedPostgres must be configured.");

var databaseName = configuration["ProductDatabase:Name"];
if (string.IsNullOrWhiteSpace(databaseName))
{
    throw new InvalidOperationException(
        "ProductDatabase:Name must be configured.");
}

var builder = new NpgsqlConnectionStringBuilder(sharedConnectionString)
{
    Database = databaseName
};

var dataSource = NpgsqlDataSource.Create(builder.ConnectionString);
```

Do not use string concatenation or regular expressions to change a connection string.

## Aspire

Add a database to the local PostgreSQL resource. Name the database reference after the shared connection key so configuration remains consistent:

```csharp
var postgres = builder.AddPostgres("postgres");
var productDatabase = postgres.AddDatabase(
    "sharedpostgres",
    "product");

builder.AddProject<Projects.API>("api")
    .WithReference(productDatabase)
    .WithEnvironment("ProductDatabase__Name", "product")
    .WaitFor(productDatabase);
```

`AddDatabase` creates a database within the PostgreSQL resource; it does not create another PostgreSQL server.

## Azure Bicep

Reference the existing server and add only a child database resource:

```bicep
targetScope = 'resourceGroup'

param serverName string
param databaseName string

resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2024-08-01' existing = {
  name: serverName
}

resource database 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2024-08-01' = {
  parent: postgresServer
  name: databaseName
  properties: {}
}
```

If the server is in a shared resource group, deploy the module at that resource-group scope:

```bicep
var sharedResourceScope = resourceGroup(
  subscription().subscriptionId,
  platform.naming.resourceGroup)

module sharedPostgresDatabase 'shared-postgres-database.bicep' = {
  name: 'shared-postgres-database'
  scope: sharedResourceScope
  params: {
    serverName: sharedPostgresServerName
    databaseName: productDatabaseName
  }
}
```

Pass runtime configuration independently:

```bicep
[
  {
    name: 'ConnectionStrings__SharedPostgres'
    value: sharedPostgresConnectionString
  }
  {
    name: 'ProductDatabase__Name'
    value: productDatabaseName
  }
]
```

Mark the shared connection-string parameter `@secure()`. Do not emit it through module outputs.

## GitHub Actions and Bicep parameters

Use one shared secret and separate variables:

```yaml
env:
  SHARED_POSTGRES_SERVER_NAME: ${{ vars.SHARED_POSTGRES_SERVER_NAME }}
  PRODUCT_DATABASE_NAME: ${{ vars.PRODUCT_DATABASE_NAME || 'product' }}
  SHARED_POSTGRES_CONNECTION_STRING: ${{ secrets.SHARED_POSTGRES_CONNECTION_STRING }}
```

```bicep
param sharedPostgresConnectionString =
  readEnvironmentVariable('SHARED_POSTGRES_CONNECTION_STRING', '')
param sharedPostgresServerName =
  trim(readEnvironmentVariable('SHARED_POSTGRES_SERVER_NAME', ''))
param productDatabaseName =
  trim(readEnvironmentVariable('PRODUCT_DATABASE_NAME', 'product'))
```

Never create `PRODUCT_DATABASE_CONNECTION_STRING` or an equivalent project-specific secret.
