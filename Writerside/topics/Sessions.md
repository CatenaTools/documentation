# Sessions

Sessions can be used authorize access to API calls which require them. A session is typically used to represent users
after a successful login and expire after some time. A session can be referenced by its ID and may contain metadata that
is accessed by services.

[//]: (TODO: Need a topic that explains the separation between authentication and accounts.)
> A session produced by an authentication service typically represents a user but is only populated with account
> information like a role (user, admin, etc) by an account service. Both an authentication service and an account
> service (or a combined service) are required to make use of session types/roles.

## Producing a session

An authentication service can produce a new session by depending on `ISessionStoreAccessor` and calling `NewSession()`
after a successful user login. The session ID assigned in this call uniquely represents the session.

```C#
string sessionId = IdentifiableUuid.Session.ToString();
_sessionStoreAccessor.NewSession(
    sessionId,
    SessionDataKeys.PROVIDER_LOGIN_PAYLOAD,
    payload
);
```

## Producing a session with an account role

An account service can add a role to be used as part of session authorization by depending on `ISessionStoreAccessor`
and calling `SetSessionData()` with the `SessionDataKeys.ACCOUNT` key and an entity containing a role in
the `AccountRoleMetadataKey` key.

However, for the account role to be handled as a session type, it must be mapped by `ParseSessionTypeFromString()`.
Existing Catena services support the following roles:

* "user" (`UserAccountSessionType`) corresponds to a `USER_ACCOUNT` session type
* "admin" (`AdminAccountSessionType`) corresponds to an `ADMIN_ACCOUNT` session type

For example, to identify an existing session as an admin:

```C#
var accountEntity = new Entity();

accountEntity
.Metadata
.Add(
    AccountRoleMetadataKey,
    new EntityMetadata { StringPayload = AdminAccountSessionType }
);

_sessionStoreAccessor.SetSessionData(
    context.ToAuthServerCallContext(),
    SessionDataKeys.ACCOUNT,
    accountEntity
);
```

## Requiring a session

A service that requires any session to operate can use the `AuthRequired` attribute on its method(s), ex:

```C#
[AuthRequired(SessionType.VALID_SESSION)]
public override Task<LogoutResponse> Logout(
    LogoutRequest request,
    ServerCallContext context
)
```

This will require the caller to have any valid session before calling logout. The Catena node will have already
validated the session before `Logout()` is called.

## Requiring a session with an account role

In the example above, the session type `VALID_SESSION` indicates any valid session is sufficient. However, other session
types such as `USER_ACCOUNT` and `ADMIN_ACCOUNT` can be required which correspond to particular account roles. In these
cases, the session must also have an account payload containing the corresponding role,
see [](#producing-a-session-with-an-account-role).

The RPC method will only be called if there is a valid session **and** it contains an account **and** that account has a
role _at least_ what is specified in the `AuthRequired` attribute.

For example, this `AdminDeleteAccount` call requires the caller have a session with an admin account:

```C#
[AuthRequired(SessionType.ADMIN_ACCOUNT)]
public override Task<DeleteAccountResponse> AdminDeleteAccount(
    DeleteAccountRequest request,
    ServerCallContext context
)
```

## Accessing the session in the RPC method

When a service RPC method requires a session, it will receive an `AuthServerContext` in its `ServerCallContext`
parameter. The original session information like the ID or session type can be accessed by casting back to
the `AuthServerContext` using `ToAuthServerCallContext()`.

```C#
[AuthRequired(SessionType.VALID_SESSION)]
public override async Task AwaitSession(
    AwaitSessionRequest request,
    IServerStreamWriter<AwaitSessionResponse> responseStream,
    ServerCallContext context
)
{
    var sessionId = context.ToAuthServerCallContext().GetSessionId();
```

> Using `ToAuthServerCallContext()` requires that the RPC method use the `AuthRequired` attribute or there will be no
> session validation and therefore no session to access.

A set of extensions to the session store allows reading/writing the session using the `AuthServerContext` and should be
preferred over fetching and passing the session ID, ex:

```C#
var (sessionId, data) = _sessionStoreAccessor.GetSessionIdAndData<Entity>(
    context.ToAuthServerCallContext(),
    SessionDataKeys.ACCOUNT
);
```

The current session type is also available in the `AuthServerContext`. This can be useful if an RPC method needs to
behave differently when called by users with different roles. For example, an RPC to get a particular leaderboard entry
may only return the player and score when called normally but may _also_ return additional data like telemetry (creation
date, map, etc) when called as an admin.

```C#
AuthRequiredAttribute.SessionType? sessionType = context.ToAuthServerCallContext().GetSessionType();
```

`GetSessionType()` will return null only if the session type required is `VALID_SESSION`; the Catena node will not look
up the current account role if it is not essential.