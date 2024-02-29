# Service Registration

**Warning: This is an RFC/in progress spec**

A simple registration system for users to create new Catena services and for us to provide routing to those services. If a user follows this template they get a SQL Database accessor setup as well as a GRPC/http service which can be built using the typical GRPC/http pattern.

## Problem

Current service registration process is difficult and cumbersome. Additionally, it can be unclear to the user how to setup a logger or register their services for use in Catena.

## Design Overview & Scope

We will create an interface that the user can implement to create a service that complies to the Catena Specification. Once the user has an implementation they register it and we can instantiate and run the service for them.

Users can add Routes via GRPC modifications and they will be picked up automatically. The core of this feature will be the CatenaNodeBuilder which will build a node of all Catena services it knows about and handle setting up Networking Routes and requirements for all of the Catena services.

## Interfaces

### CatenaService

Below is the CatenaService interface, all services must implement this interface. For now the only relevant field is the ServiceName which is the name of the argument that the user must pass to start that service. Eventually this will also contain helper functions.

```csharp
public interface ICatenaService
{
    public string ServiceName();

} 
```

### CatenaServiceWithDatabase

CatenaServiceWithDatabase allows the user to specify a database accessor that will be initialized by the Catena platform.

```csharp
// Register a Catena Service and It's dependencies.
    // Handle service discovery
    // Establish and initialize database connections
    public interface ICatenaServiceWithDatabase<TDatabaseAccessor> : ICatenaService where TDatabaseAccessor : IDatabaseAccessor
    {
        public static void WithDatabase(WebApplicationBuilder builder)
        {
            builder.Services.AddSingleton<TDatabaseAccessor>();
        }

        TDatabaseAccessor GetDatabase();

        public Type GetDatabaseAccessorType()
        {
            return typeof(TDatabaseAccessor);
        }

        public void Migrate()
        {
            GetDatabase().Migrate();
        }
    }
```

## Usage

Users can register their services with the CatenaNodeBuilder. The node builder than handles setting up all database accessors and connections to other CatenaNodes using the provided config. An example of this is provided below.

```csharp
CatenaNodeBuilder nodeBuilder = new(builder);

var app = nodeBuilder
    .RegisterService<AuthenticationService>("authentication")
    .RegisterService<AccountsService>("accounts")
    .RegisterService<GroupsService>("groups")
    .BuildDatabases()
    .BuildApp();

app.Run();
```

Since the group service implements `ICatenaServiceWithDatabase<EntitiesSqlDbAccessor>` the CatenaNodeBuilder will provide an instance of EntitiesSqlDbAccessor to the constructor via c# dependency injection.

```csharp
public class GroupsService : Groups.GroupsBase, ICatenaServiceWithDatabase<EntitiesSqlDbAccessor> { }
```

## Service Discovery & Routing

**Goal:**

- CatenaNode - The main entrypoint for all services in the Catena mesh. The CatenaNode will register one or many GRPC services and expose them to the network.
- CatenaService - One GRPC service in the Catena network, for example `Accounts` or `Auth`
- Service Registration and Routing
    - User Configures one or many Catena Nodes with some number of services
    - The `CatenaNode` registers with the service mesh what `CatenaServices` it is running as well as what `CatenaRoutingPolicies` they support. The `CatenaNode` also handles health checks and metrics for the service mesh.
        - `CatenaService` has a name and unique id
        - `CatenaRoutingPolicy` can be one of
            - `All` - Requests broadcast to all instance of a service
            - `Any` - Requests can go to any service (round robin or random)
            - `Hash` - Requests will be routed by hash to a specific service consistently
