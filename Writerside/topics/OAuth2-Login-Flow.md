# Implementing an OAuth2 Client 

In this tutorial we will cover implementing a client flow for an OAuth2 Provider from the client. For example log in with discord from a game launcher.

## Prerequisites

1. Configure and install [Catena Tools](overview.md)
2. Create a [Discord OAuth Application](https://discord.com/developers/docs/topics/oauth2)
   1. Take note of your client_id, & client secret
   2. Set the redirect URL to https://localhost:5001/api/v1/authentication/PROVIDER_DISCORD/callback - This will be handled by [The Authentication Service's Callback Handler](https://github.com/CatenaTools/catena-tools-core/blob/55376240181152d9f81537051041cf8cf73956b2/Protos/api/v1/authentication.proto#L31)
3. Add Your Discord Information to `appsettings.Development.json`

## Steps

### Step 1: Ensure the Backend Auth Validator is Written and Registered for the given service. Auth Validators should implement the `IAuthValidator` Interface.

```C#
public interface IAuthValidator
{
    public Provider ProviderType { get; }
    public bool IsOauth2Provider();
    // OAuthRedirectUrl() is only used for OAuth2 providers, and should return the URL to redirect to for OAuth2 authentication.
    public string OAuthRedirectUrl(string sessionId) { return  "";}
    public Task<AuthValidatorResponse> Validate();
}
```

**IsOauth2Provider()** - Should return true if this is an oauth provider

**OAuthRedirectUrl(string sessionId)** - The URL that the client should redirect to while performing the authorization flow.

**Validate()** - Will be called by the Auth service, it should return an `AuthValidatorResponse` if the user successfully logged in or throw an Exception if the user failed to log in.

The discord implementation of this interface can be found [here](https://github.com/CatenaTools/catena-tools-core/blob/27d10667009c0353c54da800519613b998ee709c/Services/Validators/AuthValidators/DiscordAuthValidator.cs#L1).

### Step 2: Make the initial request from the client 

An example fetch request is shown below. This will return either a success response or an oauth redirect url if it is an oauth 2 provider.

```Javascript
fetch(CATENA_URL + '/api/v1/authentication/login', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'request-id': ""
    },
    body: JSON.stringify({
        provider: remoteProvider,
        payload: token
    })
})
```

**Example OAuth Response for Discord**

```Javascript
{
    "redirect_uri": "https://discord.com/api/oauth2/authorize?client_id=1103300744876130315&redirect_uri=http://localhost:5001/api/v1/authentication/PROVIDER_DISCORD/callback&response_type=code&scope=identify&state=80333b00-c87e-48fc-a2c7-4bf6fbc913de"
}
```

The response will also include a `session-id` header which will be used for the following steps.

### Step 3: Using the session ID returned by the initial login request, subscribe to the auth ready endpoint

This endpoint will  return a success response if the user successfully logs in, or a failure response if the login fails. It will wait to return this response until the user has completed the login flow or the session has expired.

```Javascript
var ret = await fetch(this.url + "/api/v1/authentication/await", {
    headers: {
        'Content-Type': 'application/json',
        'request-id': "12345",
        'session-id': this.session
    },
})
var json = await ret.json();

if (json.successfulAuth){
    // Login Successful
}else {
    // Login Failed
}
```

### Step 4: Redirect the client to the oauth link returned from the response

This can be done however your client handles redirects, in this case:

```Javascript
var {redirect_uri} = { // This would be the json response from the fetch request above
    "redirect_uri": "https://discord.com/api/oauth2/authorize?client_id=1103300744876130315&redirect_uri=http://localhost:5001/api/v1/authentication/PROVIDER_DISCORD/callback&response_type=code&scope=identify&state=80333b00-c87e-48fc-a2c7-4bf6fbc913de"
};

window.open(redirect_uri, '_blank');
```

After which the callback from step3 should finish.

### Step 5: After the callback succeeds in step 3 you may call CreateOrGetAccountFromToken to get the user's account 

```Javascript
await fetch(this.url + '/api/v1/accounts', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'request-id': "12345",
        'session-id': this.session
    },
    body: JSON.stringify({})
}).then((data) => data.json()).then((data) => {
    console.log(data)
    this.setLoginState({
        id: data.account.id,
        username: data.account.displayName,
        profile: "http://placekitten.com/200/300"
        error: ""
    })
})
```

After which the user will have successfully authenticated using discord for the catena platform.