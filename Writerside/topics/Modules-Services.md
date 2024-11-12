# Modules/Services

A module is a pluggable component of Catena - usually a service. When a Catena node is started, the `CatenaNodeBuilder`
identifies all the available modules/services that use the `CatenaModule` attribute, inspects their dependencies, and
then starts the configured services. The modules may be common services provided by Catena, they may be core,
always-available modules built into Catena, or they may be provided in a separate library.

Example of a simple Catena (service) module:

```C#
[CatenaModule]
public class CatenaAuthenticationService : CatenaAuthentication.CatenaAuthenticationBase
{
    // implementation here
}
```

This is the `CatenaAuthenticationService` module which implements the `CatenaAuthentication` service. It will be
discovered since it uses the `CatenaModule` attribute.

> A module does not have to be a gRPC service. However, since most modules are services these terms are often used
> interchangeably.
>
> Catena can automatically handle a module whether it implements a gRPC service or not and there is no difference in the
> declaration.

## Service types

There are two types of services in Catena: _singleton_ and _transient_.

By default, a service is registered as a _transient_ service. This means multiple copies of it may exist at runtime,
scaled by Catena to handle the request load, which is preferable for most services.

However, some service don't make sense as _transient_ services. For example, a service that keeps instance data in
memory may not work if there were multiple copies at runtime. These services can be declared a singleton
with `CatenaModule` and Catena will not start multiple copies of the service.

```C#
[CatenaModule(CatenaModuleAttribute.ServiceType.SINGLETON)]
public class CatenaMatchBrokerService : CatenaMatchBroker.CatenaMatchBrokerBase
{
    // implementation here
}
```

This is the `CatenaMatchBrokerService` module which keeps registered service information in memory. By design, it
coordinates all game servers which means it must be a singleton, otherwise some game servers might be known to one
instance of the service while others are known to a different instance.

## Dependencies

Services typically have some dependencies: configs, databases, the event gateway, a subscription manager, the session
store, etc. Instances of these dependencies are made available to the service at runtime when the service's constructor
is called.

This is also how Catena knows what the dependencies of a service are when starting a node. Catena checks the
constructor of any class using the `CatenaModule` attribute and ensures those modules are available, even if they are
not explicitly in the <tooltip term="services list">services list</tooltip>.

Catena will handle creating instances of dependencies based on the request load and any constraints of the dependencies.

```C#
public CatenaPartiesService(
    ISessionStoreFactory sessionStoreFactory,
    PartiesSqlDbAccessor dbAccessor,
    GatewayServiceClient? gatewayServiceClient,
    ServiceSubscriptionManager.SubscriptionManager subscriptionManager,
    IOptionsMonitor<CatenaPartiesConfig> configMonitor
)
{
    // implementation here
}
```

This is the constructor of the `CatenaPartiesService`. Catena will ensure the service has access to an instance of each
of these things when creating an instance of the service. Some may be used only during instantiation of the service or
the service may use and access the dependency for the life of the service instance.

> It is possible to add a parameter to the constructor that is otherwise unused and simply creates a dependency to
> ensure another module/service is loaded.

### Singleton dependencies

Dependencies are treated similar to top-level modules/services and may also declare themselves as a singleton. This can
be particularly useful if a service can be transient - and therefore scale with load - but must depend upon some
subcomponent that must be a singleton.

The `CatenaWarmbodyMatchmaker` and `CatenaMatchmakingService` are an example of this case; the service is transient but
the actual matchmaking algorithm is a singleton. The `CatenaWarmbodyMatchmaker` is also an example of a module that is
not a gRPC service.

> Some core types such as the `ISessionStoreFactory` don't declare themselves as a singleton but are instead hardcoded
> as a singleton.
>
> It is usually not important when depending on a module to understand whether it declares itself as a singleton.

### Interface dependencies

Some dependencies may be represented by an interface. For example, `ISessionStoreFactory` may have multiple
implementations but only one will be used based on the configuration. Typically, a service should depend on the
interface and not a specific implementation, so it works regardless of which implementation is configured. This also
permits Catena to share instances of a module between multiple services which depend upon it.

In some rare cases, a service may be written to require a specific implementation of one of these interfaces. It is
important, however, that the service still use the interface type in its constructor so that Catena can use that
implementation for other users of the same interface. Otherwise, the result may be dependencies upon multiple
implementations from different modules and the node will fail to start.

In the rare case where a service requires a specific implementation of an interface, it can use a "keyed service" to
both depend on the interface but require that specific implementation. An example of this is provided
in `ApiKeysExampleService`.

More information about configuring the interfaces used at runtime is available in [](Configuring-interface-dependencies.md).

### Database dependencies

When a module or service depends on a class that inherits from `IDatabaseAccessor`, the database will automatically be
added to a migration set and the migrations will be performed before starting the node.

## Module groups

Modules may be grouped for ease of use when specifying services to start with a node. This may be done two different
ways:

1. A module group can be created when writing modules by adding one or more `CatenaModuleGroup` attributes to the
   module.
2. Ad-hoc/custom by adding an entry to `CustomServiceGroups` in the configuration.

Any discovered modules, whether they are built-in or external, will be in the `@all` group.

These modules in these groups can then be included/excluded with the <tooltip term="services list">services
list</tooltip> by using the group name prefixed with `@`, ex: `--services @all,-@CatenaExamples`

> Group names and module names can be mixed to include/exclude services.
>
> **The <tooltip term="services list">services list</tooltip> is not processed in order. Modules are first included,
then modules are excluded at the end.**
>
> To force include a specific service from a group which has been excluded, prefix the service with a `+`, ex:
`--services @all,-@CatenaExamples,+ExampleTransientService`

## External module libraries

The `ExtraModuleLibraryPaths` configuration can be used to specify paths to libraries containing additional modules to
be discovered by the `CatenaNodeBuilder`.

The services that should be started from any external libraries should also be specified in
the <tooltip term="services list">services list</tooltip>.

[Catena provides a separate example project for creating a module library.](https://github.com/CatenaTools/catena-tools-module-example)

> The `CatenaNodeInspection` service can be used to interrogate which modules were discovered and which modules were
> loaded at startup based on the services list.