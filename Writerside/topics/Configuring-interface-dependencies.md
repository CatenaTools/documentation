# Configuring module loading and dependencies

Catena modules are pluggable and can be enabled or disabled. There may also be multiple modules that satisfy an
interface dependency of a particular service. For flexibility, Catena supports configuring the module and dependency
loading via two mechanisms:

1. Module (and group) enabling/disabling
2. Choosing interface implementations

## Enabling/disabling services (and groups)

Modules (and groups) can be enabled and disabled via the command line, launch profiles, environment variables, and
config files. It is strongly recommended to utilize just one of these approaches to reduce confusion.

By default, all the modules provided with Catena are enabled in the development config unless they are examples. This is
done in `appsettings.Development.json`:

```json
{
  "Catena": {
    "Services": "all,-@CatenaExamples"
  }
}
```

There may be specific services that are not required for a game. They can be excluded by appending the default with the
service name prefixed with a `-`, ex: `all,-@CatenaExamples,-CatenaPartiesService`.

When the required set of services is very small, it may make sense not to append to the default, but to replace the
services list so it only contains the required services, ex: `CatenaAuthenticationService,CatenaAccountsService`.

## Single-implementation interfaces

Many services depend on interfaces that can have multiple extant implementations, however, only a single module can be
used to satisfy an interface dependency on a node at runtime. The default configuration specifies all the necessary
implementations for the default services. When it becomes necessary to change implementations there are several methods
available:

First, simply choosing which module(s) to load can simplify the selection. For example, the entitlements service
utilizes `IOfferProvider` which has two implementations: `SingleNodeInMemoryOfferProvider` and
`RedisDistributedOfferProvider`. Simply enabling/disabling these modules so only one is loaded is a reasonable way to
ensure the desired module is selected as _the_ `IOfferProvider`.

Second, when multiple implementations for an interface _are_ enabled, there is often a configuration option that allows
specifying which implementation to use. For `IOfferProvider`, this option is colocated with the entitlements service
options since it is a component of that service:

```json
{
  "Catena": {
    "Entitlements": {
      "OfferProvider": "SingleNodeInMemoryOfferProvider"
    }
  }
}
```

Third, there is a fallback config option if no specific option is available or utilized: `PreferredImplementations`.
This top-level option uses the interface name to specify which implementation to use, ex:

```json
{
  "PreferredImplementations": {
    "IOfferProvider": "SingleNodeInMemoryOfferProvider"
  }
}
```

Finally, if no specific configuration option is available or utilized and the fallback is also not used, then the Catena
node builder will auto-select an implementation. It is strongly recommended to use one of the configuration approaches
to select an implementation otherwise an under-configured module may be selected or the selected module may change when
updating Catena if a new implementation is introduced.

### Unsatisfied interface dependencies

Interface implementation selection is somewhat independent of module loading. It is possible to create a configuration
where there are unsatisfied interface dependencies:

1. By disabling all modules that satisfy an interface.
2. By configuring a single implementation (with a specific option or a fallback option) and disabling that module.

In these situations the Catena node will fail to start and will report a missing dependency. To fix this, either adjust
the configured interface implementation or the enabled/disabled modules.

### Exclusive single-implementation interface configuration

Some modules implementing interfaces are designed to be loaded concurrently. This done to simplify configuration or so
different services can depend specifically on each implementation.

However, there are some interfaces and/or implementations which are not designed to be loaded concurrently and will
trigger startup errors when loaded together. One such interface is the `ISessionStoreAccessor` (session store).
`RedisSessionStoreAccessor` and `SqliteSessionStoreAccessor` are written such that only one module can be enabled.

When an interface has implementations that are mutually exclusive, it is possible to enable/disable modules so only one
is loaded. It is also possible to use a `!` prefix when specifying the desired implementation in the config, ex:

```json
{
  "PreferredImplementations": {
    "ISessionStoreAccessor": "!RedisSessionStoreAccessor"
  }
}
```

This will ensure only the `RedisSessionStoreAccessor` implementation is selected and all other implementations of
`ISessionStoreAccessor` will be unloaded. This is a good shortcut to specifying individual modules to enable/disable.