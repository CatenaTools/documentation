# Authentication

The `AuthenticationService` is responsible for handling login requests from client applications sent via `LoginWithProvider`.

```protobuf
rpc LoginWithProvider(LoginWithProviderRequest) returns (LoginWithProviderResponse) {
    option (google.api.http) = {
        post: "/api/v1/authentication/login"
        body: "*"
    };
};

/* Accepted auth providers. */
enum Provider {
    PROVIDER_UNSPECIFIED = 0; // unauthenticated
    PROVIDER_UNSAFE = 1; // test accounts (development only)
    PROVIDER_DISCORD = 2;
}

message LoginWithProviderRequest {
    Provider provider = 1;
    string payload = 2; // provider-specific auth token (username for unsafe provider)
}
message LoginWithProviderResponse {}
```

When the `AuthenticationService` receives a `LoginWithProviderRequest`, it will use the `AuthValidatorFactory` within the application’s service collection to select an “auth validator” to use based on the `Provider` specified in the request.

**Note: All auth validators must implement the interface** `IAuthValidator`**.**

An auth validator’s `Validate()` function is used to validate the **access token** specified as the `payload` in the request. This often involves the use of a third-party API e.g. Discord’s `https://discord.com/api/users/@me`.

After successfully receiving third-party API user information, using a portion of said information, `Validate()` will then construct and return and `AuthValidatorResponse`.

```csharp
public AuthValidatorResponse(string accountId, Authentication.Provider providerType, string displayName)
{
		AccountId = accountId;
    ProviderType = providerType;
    DisplayName = displayName;
}
```

The `LoginWithProvider` handler uses this information to construct a `ProviderLoginPayload`.

```protobuf
/* Bundled data required to log in or create an account using a given provider. */
message ProviderLoginPayload {
    string provider_account_id = 1; // provider-specific user ID
    authentication.Provider provider_type = 2;
    string provider_display_name = 3; // provider-specific display name of user
}
```

The handler then opens a new user session and stores the `ProviderLoginPayload` in the session store using the `SessionStoreAccessor` in the application’s service collection.

Finally, the handler returns a `LoginwithProviderResponse` with the ID of the newly created session added as response metadata. Subsequent requests sent to the application with a `session-id` included in request metadata will be authenticated requests assuming the user’s session is valid.

*Note: Even with a valid session, a user’s request may still be disallowed due to their role level.*