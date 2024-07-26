# External, custom, or token-based sessions

The Catena [authentication service](Authentication.md) is typically responsible for producing sessions when users login and the Catena [accounts service](Accounts.md) will populate the session with account information. Other Catena services can then use this session when authorizing certain operations or to lookup data.

However, it is possible use a completely separate service for accounts/sessions/authorization, even one external to Catena, without using the Catena authentication and accounts services and **still** use other Catena services that may rely on a session.

There are several phases to accessing a session:
* [Finding the session](#finding-the-session-isessionresolver) - A <tooltip term="session resolver">session resolver</tooltip> will report the current session ID, typically by reading a header value.
* [Validating the session](#accessing-the-session-contents-isessionstoreaccessor) - The auth framework will always attempt to validate a session using a <tooltip term="session store accessor">session store accessor</tooltip>.
* [Accessing the session data](#accessing-the-session-contents-isessionstoreaccessor) - With a valid session, services can use a <tooltip term="session store accessor">session store accessor</tooltip> to read/write session data. Additional information about how services use sessions in Catena is covered [here](Sessions.md).

## Finding the session (`ISessionResolver`)

The authentication framework for sessions uses a resolver to identify the current, unique session ID. When using the Catena authentication service, this simply extracts the session ID from a header.

A custom implementation of `ISessionResolver` can be used to extract the true session ID from anywhere - including a 3rd-party header/token or service or generating one from other data. The most important thing for a resolver is that it consistently returns a persistent, unique session ID; a new session ID is a new session.

The `ISessionResolver.GetSessionId()` will be called during each API request authorization to get the current session ID, and this method will receive access to all the metadata/headers sent with the request. If a session ID can be resolved and is valid, this session ID is what the request handler will receive and use if it needs to access the session.

For example, if the session ID is stored in a custom header, then `GetSessionId` can just return the value of that header. If that custom header stores the entire session - the ID and all the data - then it may be necessary to decode and extract the ID before returning it.

A custom session resolver can be used by configuring the Authentication SessionResolver:

```json
    {
      "Authentication": {
        "SessionResolver": "MyCustomSessionResolver"
      }
    }
```

> This is a top-level configuration for the auth framework, not the Catena Authentication service.

## Accessing the session contents (`ISessionStoreAccessor`)

Validating the session and reading/writing data in the session is done using the current session ID once it's known from the resolver. The authentication framework uses an `ISessionStoreAccessor` implementation, typically a database, for these operations. (Creating a new session or invalidating an existing session is also done through the same session store accessor.)

During each API request authorization, once the session ID is known, `ISessionStoreAccessor.IsSessionValid()` will be called which may use the session ID and any data source to determine if the session is still valid. It may check a database, make a call to a 3rd-party service, or extract validity information from the header/token itself.

When a service needs to access data in the session, it will call some form of `GetSessionData` or `SetSessionData`. These calls may cause the session store to query a 3rd-party service, access a database, or decode the data from a header.

A custom session store can be used by configuring the Catena SessionProvider:

```json
    {
      "Catena": {
        "SessionStore": {
          "SessionProvider": "MyCustomSessionStore"
        }
      }
    }
```

## Emulating session data

Since existing services may expect certain keys/values to exist in the session, a session store may need to emulate or translate the data it stores to provide those other values. For example, the Catena party service expects to get the current account information as an `Entity` using `account` as the key. A custom session store may need to be able to produce this from its own data to be compatible with the Catena party service.

> `TokenSessionStore` contains an example of this emulation.


## Read-only, external tokens

> `TokenSessionStore` is an example of a read-only, token session store that may provide a basis for a custom implementation.

For sessions where the data is contained entirely within a header/token, it is necessary to implement `ISessionResolver` and `ISessionStoreAccessor` together. This is due to the fact that headers are only available to the resolver and only the session ID is available to services utilizing the session. Combining these interfaces allows the header to be decoded once while resolving the session ID and then the same data can be referenced again later to access the contents of the session.

When using a combined session resolver+store, be sure to configure it as both the session resolver and provider:

```json
    {
      "Catena": {
        "SessionStore": {
          "SessionProvider": "TokenSessionStore"
        }
      },
      "Authentication": {
        "SessionResolver": "TokenSessionStore"
      }
    }
```

## Read-write custom sessions

Read-write custom sessions are supported when a custom header contains only a session ID and all operations on the session are conducted by accessing a separate data store - either directly or via a 3rd-party service.

Read-write custom sessions where the entire session/token with data is contained within a custom header is **not supported at this time**.