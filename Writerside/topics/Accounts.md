# Accounts

The Catena `AccountsService` is responsible for creating, fetching, updating, and deleting accounts and their associated metadata.

## How does Catena define an account?

An `Account` is an `Entity` with an `account_id` and a `display_name`.

```csharp
Create.Table("accounts")
    .WithColumn("account_id").AsString().PrimaryKey()
    .WithColumn("display_name").AsString()
    .WithColumn("auth_role").AsString();
```

An account may also have an arbitrary number of **metadata key-value pairs** associated with it.

```csharp
Create.Table("accounts_metadata")
    .WithColumn("account_id").AsString().PrimaryKey().ForeignKey("accounts_accounts_metadata", "accounts", "account_id")
    .WithColumn("key").AsString().PrimaryKey()
    .WithColumn("payload_type").AsInt32().NotNullable()
    .WithColumn("int_payload").AsInt64().Nullable()
    .WithColumn("string_payload").AsString().Nullable();
```

Notably, on account creation, a single key-value metadata pair `"auth-role": "user"` is associated with an account by default. This auth role will be checked by the `AuthServerInterceptor` on subsequent requests made by this account.

*(see [Feature: Metadata](https://www.notion.so/Feature-Metadata-10c467de69064a6ba6c872b45bf24d71?pvs=21) for more information on supported metadata types and operations)*

Finally, an account also has one or more **auth provider associations**. These define the auth providers that a user may use for login that result in their account / role being used as the sender of subsequent requests.

```csharp
Create.Table("accounts_providers")
    .WithColumn("account_id").AsString().ForeignKey("accounts_accounts_providers", "accounts", "account_id")
    .WithColumn("provider_account_id").AsString().PrimaryKey()
    .WithColumn("provider_type").AsString().PrimaryKey();
```

## Creating a new account / fetching an existing account

*Note: In order to create a new account or fetch an existing account, a user must have previously logged in via `LoginWithProvider`.*

When the `AccountsService` receives a `CreateOrGetAccountFromTokenRequest`, it will use the **provider account ID**, **provider display name**, and **provider type** stored in the `ProviderLoginPayload` in the session store created at login to query the accounts database for the existence of an account that has previously been created with the same provider information.

If such an account exists, the handler for `CreateOrGetAccountFromToken` will return the account in a `CreateOrGetAccountFromTokenResponse` after adding its base-64 encoded representation the the session store for future access by method handlers of subsequent RPCs.

Otherwise, an account with this provider information will be created, being assigned the role of `user` by default, its base-64 representation will similarly be added to the session store, and it will be returned in a `CreateOrGetAccountFromTokenResponse` by the method handler.

```protobuf
rpc CreateOrGetAccountFromToken(CreateOrGetAccountFromTokenRequest) returns (CreateOrGetAccountFromTokenResponse) {
    option (google.api.http) = {
        post: "/api/v1/accounts"
        body: "*"
    };
};
message CreateOrGetAccountFromTokenRequest {}
message CreateOrGetAccountFromTokenResponse {
    Account account = 1;
}
/* Bundled data that describes an account and its metadata. */
message Account {
    string id = 1; // GUID for this account
    string display_name = 2; // display name for this account
    string auth_role = 3; // auth level of the account
    map<string,groups.EntityMetadata> metadata = 4; // metadata associated with the account
}
```

## Fetching an existing account by ID & updating an account

The Catena `AccountsService` supports fetching an existing account by its ID via `GetAccountById`.

```protobuf
rpc GetAccountById(GetAccountByIdRequest) returns (GetAccountByIdResponse) {
    option (google.api.http) = {
        get: "/api/v1/accounts/{account_id}"
    };
};
message GetAccountByIdRequest {
    string account_id = 1;
}
message GetAccountByIdResponse {
    Account account = 1;
}
```

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

## Linking an account to a new auth provider

The Catena `AccountsService` supports users linking new auth providers to their accounts via `LinkAccountToProvider`.

To facilitate this, the `LinkAccountToProviderRequest` contains a `ProviderLoginPayload` identical in structure to the one added to the session store by the method handler for `LoginWithProvider`.

The information contained within this `ProviderLoginPayload` - **provider account ID**, **provider display name**, and **provider type** will be used to link a users account to a new auth provider.

If this operation completes successfully, a user will be allowed to use this newly added auth provider for future logins, in addition to those that they already had associated with their account.

```protobuf
rpc LinkAccountToProvider(LinkAccountToProviderRequest) returns (LinkAccountToProviderResponse) {
    option (google.api.http) = {
        post: "/api/v1/accounts/link"
        body: "*"
    };
};
message LinkAccountToProviderRequest {
    ProviderLoginPayload login_payload = 1; // only provider_type and provider_display_name required / expected
}
message LinkAccountToProviderResponse {
    Account account = 1;
    bool account_linked = 2; // indicator of success of linking existing account to new auth provider
}
/* Bundled data required to log in or create an account using a given provider. */
message ProviderLoginPayload {
    string provider_account_id = 1; // provider-specific user ID
    authentication.Provider provider_type = 2; // unique string to describe this provider to associate the account with this token
    string provider_display_name = 3; // provider-specific display name of user
}
```

## Metadata CRUD operations

The Catena `AccountsService` supports various CRUD operations for:

1. Adding a metadata entry to an account (`CreateMetadataEntry`)
2. Updating a metadata entry associated with an account (`UpdateMetadataEntry`)
3. Deleting a metadata entry associated with an account (`DeleteMetadataEntry`)
4. Getting **a single** metadata entry associated with an account (`GetMetadataEntry`)
5. Getting **all** metadata entries associated with an account (`GetMetadataEntries`)

These are touched on in much greater detail in [Feature: Metadata](Metadata.md).