# Accounts

Catena provides an accounts system that is designed for maximum flexiblity. The rest of Catena is designed to not depend on the Account implementation, as a result, you can bring your own account implementation and swap it out for Catena's.

## How does Catena define an account?

The fields on an account are shown below, Catena primarily cares about the `account_id` and `display_name`, with the `auth_role` being the tie in to the authentication system.

| Field        | Type      | Description                                                                                                                         |
|--------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------|
| account_id   | string    | The account ID of the user, in catena, these are in the form `account-f5c1791c-24d1-4bec-824a-866794b3f045`                         |
| display_name | string    | A user's display name, usually pulled from the first provider they logged in with.                                                  |
| auth_role    | string    | The user's authentication level                                                                                                     |
| metadata     | key-value | Key value pairs representing additional metadata on the account. This data can specified depending on the requirements of your game |

An account may have an arbitrary number of **metadata key-value pairs** associated with it. These values are typed in order to enable you to write more robust code.

| name | type |
|---|
| key | string |
| int_payload | integer |
| string_payload | string |

## Creating a new Account

> The account creation flow ties in with the [](Auth-Framework.md), parts of this interface can be swapped out as-needed in order to support differing requirements.

The diagram below shows the default out-of-the-box auth flow for Catena using Twitch Authentication. Each step is described in detail below.

<include from="AuthenticationLibrary.topic" element-id="login-with-twitch"></include>


### Create a new Session

In a default configuration, prior to creating an account (or logging into an existing account) the user must first get a provider login session. This can be thought of as an identifier tied to a third party, like Discord or Twitch that uniquely identifies the user.

After the user hits the LoginWithProvider api and completes the flow, they will have a session in the Catena backend. This session can then be used by the account service to create the user's account.

<chapter collapsible="true" title="LoginWithProvider API">
    This request returns a <code>session-id</code> header in the response that can be used to refer to this user's session.
    <api-endpoint openapi-path="../apispec/openapi/api/v1/catena_authentication.swagger.json" endpoint="/api/v1/authentication/login" method="POST" collabsible = true>
        <request>
        {
            "provider": "PROVIDER_GOOGLE"
        }
        </request>
    </api-endpoint>
</chapter>

### Creating a new account / fetching an existing account

After the user has created their login session, they can follow the steps below.

When the `AccountsService` receives a `CreateOrGetAccountFromTokenRequest`, it will use the **provider account ID**, **provider display name**, and **provider type** stored in the `ProviderLoginPayload` in the session store created at login to query the accounts database for the existence of an account that has previously been created with the same provider information.

If such an account exists, the handler for `CreateOrGetAccountFromToken` will return the account in a `CreateOrGetAccountFromTokenResponse`. Additionally, it will update the user's session to reference this account.

Otherwise, an account with this provider information will be created, being assigned the role of `user` by default, its base-64 representation will similarly be added to the session store, and it will be returned in a `CreateOrGetAccountFromTokenResponse` by the method handler.

<chapter title="CreateOrGetAccountFromToken" collapsible="true">
This request requires you to set a <code>session-id</code> header.
<api-endpoint openapi-path="../apispec/openapi/api/v1/catena_accounts.swagger.json" endpoint="/api/v1/accounts" method="POST" />
</chapter>

### Fetching an existing account by ID & updating an account

The Catena `AccountsService` supports fetching an existing account by its ID via `GetAccountById`.

<chapter title="GetAccountById" collapsible="true">
<api-endpoint openapi-path="../apispec/openapi/api/v1/catena_accounts.swagger.json" endpoint="/api/v1/accounts/{accountId}" method="GET">
</chapter>

The `AccountsService` also supports updating an account - specifically its **display name** and **associated metadata** - via `UpdateAccount`.

```protobuf
// Updates a passed in account's display_name and metadata 
rpc UpdateAccount(UpdateAccountRequest) returns (UpdateAccountResponse) {
    option (google.api.http) = {
        patch: "/api/v1/accounts"
        body: "*"
    };
};
message UpdateAccountRequest {
    Account account = 1; // The modified account whose values will overwrite those currently in the database
	  google.protobuf.FieldMask account_mask = 2;
}
message UpdateAccountResponse {}
```

**Note: This RPC facilitates account updates via `google.protobuf.FieldMasks` meaning that only the data specified in the update request will be overwritten. The rest of the account’s data will remain unchanged.**
