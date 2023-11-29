# Authentication

The `AuthenticationService` is responsible for handling login requests from client applications sent via `LoginWithProvider`. 
It's responsible for authenticating with different login providers, validating the login state and storing information about 
that provider in the session store.

Tokens from the Authentication Service do not implicitly create an account. Instead, an account must be created by the Accounts service.
Authentication tokens can be used for a number of functions that do not require an account.

## Service Definition

The following gRPC definition describes the authentication service. Routes may be hit from either http or grpc endpoints, 
see [Making Requests](Making-Requests.md) for more detail on how routes and request types are managed.

```protobuf
service Authentication {
    // LoginWithProvider Logs in a user with the given provider.
	rpc LoginWithProvider(LoginWithProviderRequest) returns (LoginWithProviderResponse) {
        option (google.api.http) = {
            post: "/api/v1/authentication/login"
            body: "*"
        };
    };

    // AwaitSession will send a message if the user successfully logs in or fails to log in after a timeout.
    // In this flow you will log in & go through the oauth flow, then wait for the callback to be called.
    rpc AwaitSession(AwaitSessionRequest) returns (stream AwaitSessionResponse) {
        option (google.api.http) = {
            get: "/api/v1/authentication/await"
        };
    };

    // CallbackHandler handles the OAuth2 callback from the provider
    // This route is designed to be used as an HTTP Request not as a gRPC call.
    rpc CallbackHandler(OAuthCallbackRequest) returns (OAuthCallbackResponse){}

    /* GetServiceHealth Gets the health of the Authentication service, logging any errors */
    rpc GetServiceHealth(core.GetServiceHealthRequest) returns (core.GetServiceHealthResponse) {
        option (google.api.http) = {
            get: "/api/v1/authentication/healthz"
        };
    };
}
```

## Basic Authentication Workflow

To Authenticate a user, utilize the `LoginWithProvider`, `AwaitSession` and `CallbackHandler` endpoints. These three work together to support a variety of login flows. For an OpenID or Multi-step provider, this usually looks like:

Make a request to LoginWithProvider
```HTTP
POST /api/v1/authentication/login HTTP/1.1
Content-Type: application/json
Content-Length: 63

{
    "provider": "PROVIDER_DISCORD",
    "payload": ""
}
```

The response will always contain a `session-id` header which can be used to refer to this session. Some flows may require a redirection to a confirmation screen.

If the response contains a redirect uri navigate the user to that url. It will complete the login flow. You can subscribe to the await endpoint to know when the user completes the auth flow.

**Example Response:**
```JSON
{
    "redirectUri": "https://discord.com/api/oauth2/authorize?client_id=[ClientID]&redirect_uri=https://app.catenatools.com/api/v1/authentication/PROVIDER_DISCORD/callback&response_type=code&scope=identify&state=26338590-d3df-4157-848e-cbe9951adb5b"
}
```

**Awaiting Login Complete:**
```HTTP
GET /api/v1/authentication/await HTTP/1.1
Host: app.catenatools.com
session-id: d5311d50-b53a-4f43-b465-5551e1c0fdf1
```

The handler will not respond until the login flow completes, when it will return:

```JSON
{
  "successfulAuth": true,
  "message": ""
}
```

If the response does not contain a redirect uri, the provider does not require further confirmation and you many login with the session token returned in the header.

## Inner Workings

When the `AuthenticationService` receives a `LoginWithProviderRequest`, it will use the `AuthValidatorFactory` within the application’s service collection to select an AuthValidator to use based on the `Provider` specified in the request. These will implement the IAuthValidator interface.

After which the workflow diverges. For OpenID/Oauth login providers the handler will return a redirect uri as defined by the validator. For other workflows it will store the relevant session data and return a success response. 
In either case the handler returns a `LoginwithProviderResponse` with the ID of the newly created session added as response metadata. 
Subsequent requests sent to the application with a `session-id` included in request metadata will be authenticated requests assuming the user’s session is valid.

An auth validator’s `Validate()` function is used to validate the access token specified as the `payload` in the request. This often involves the use of a third-party API e.g. Discord’s `https://discord.com/api/users/@me`.

After successfully receiving third-party API user information, using a portion of said information, `Validate()` will then construct and return and `AuthValidatorResponse`.

```csharp
public AuthValidatorResponse(string accountId, Authentication.Provider providerType, string displayName)
{
	AccountId = accountId;
    ProviderType = providerType;
    DisplayName = displayName;
}
```

*Note: Even with a valid session, a user’s request may still be disallowed due to their role level.*