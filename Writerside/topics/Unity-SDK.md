# Unity SDK

## Purpose

The Unity SDK provides a mechanism for clients (and servers) to interact with the Catena platform from their games.

## Integration Process (Quick)
Just copy the entire `UnitySDK/` directory from `catena-tools-core` into `Assets/Scripts/` directory of the Unity project.

For Unity versions **older** than 2020.1 you will need to **remove** `UnitySDK/Plugins/System.Runtime.CompilerServices.Unsafe.dll`. [More information][Unity 2020.1 and protobuf]

## Integration Process (For Custom Development)

1. (Re)generate C# types from `.proto` files by running `buf generate` at the root of the `catena-tools-core` repo.
2. Make any required changes to the service clients.
3. Copy the entire `UnitySDK/` directory from `catena-tools-core` into `Assets/Scripts/` directory of the Unity project.

## Design & Structure

### Generated Types

The `catena-tools-core` project contains a Buf module that allows for the generation of Protobuf message and enum types for various languages. In the case of the Unity SDK, the language is C#.

To generate these types, run `buf generate` at the root of the project’s directory.

The Buf module is also configured to run `buf breaking`, `buf format`, and `buf lint`.

**Note that this should be done each time a modification is made to a `.proto` file within the project.**

**Also note that the generated types have been removed from the project’s compilation as to not introduce duplicate types. This is required since the Catena platform is written in C#.**

These generated types are used to construct service clients which the `CatenaHttpClient` leverages to make HTTP requests to the Catena platform.

### Service Clients

Each Catena service with at least one publicly facing endpoint will have a corresponding service client in the Unity SDK.

The two main purposes of these service clients are to:

1. Allow the `CatenaHttpClient` to send HTTP requests to the Catena platform (where they are transcoded to gRPC requests)
2. Allow the `CatenaHttpClient` to parse HTTP responses into the expected Protobuf response message type before proceeding to work with them further

Currently, these are being written by hand which is very inconvenient since developers must manually update them when changes are made to their corresponding `.proto` file.

We plan to generate them automatically in the future using a Go module and a template.

(see current progress here: https://gitlab.com/wolfjaw/protoc-gen-unity)

### `CatenaHttpClient`

The `CatenaHttpClient` initializes and maintains an instance of each service client in the Unity SDK.

It defines a delegate function `SendHttpRequest` that is passed to each service client upon their initialization to ensure consistency in how we send requests across the board for each of our Catena services. This delegate verifies that requests that should not have a body do indeed not (i.e. GET and DELETE requests) and handles exceptions.

The `CatenaHttpClient` also stores:

1. The base URL that requests are sent to (e.g. `https:localhost:5001` for Single Node mode)
2. A session ID for the currently signed in user if it has been sent in response metadata by the platform after a user successfully authenticated with `LoginWithProvider`

It can inject this session ID into the request headers of subsequent authenticated requests via it’s `AuthHandler` delegating handler.

### Delegating Handlers

The `CatenaHttpClient` currently has one delegating handler - `AuthHandler`.

`AuthHandler` has two main functions:

1. Pull a session ID from response metadata and store it locally (if it is present) after a response is received
2. Inject a session ID into a request’s headers before sending the request to the Catena platform (if it has one stored locally)

We also plan to add a `LoggingHandler` that will leverage a logger whose output we can aggregate and store in something like OpenSearch.

### External Launcher / Login Page

Catena offers an external launcher built in Flask to handle logins in an engine-agnostic way. The purpose of this launcher is to provide an easy and out-of-the-box solution without requiring developers to set-up their own login process. This external launcher is totally optional and developers can implement their own login page directly in Unity using `LoginWithProvider`.

The Catena Launcher can be found in the `catena-tools-core` repository under the `Launcher/` directory. A configuration file can be found under `Launcher/Data/config.yaml`. The modifiable fields in there are:

| Field | Description |
| --- | --- |
| login_endpoint | The Catena endpoint the launcher will attempt to send login requests to. By default this is configured to target a local Catena node (https://localhost:5001/). |
| app_file_path | The application to open following a successful authentication attempt. This application is what will receive the session_id as a command line argument upon a user completing the login process. |
| providers | A customizable list of providers that developers can add or remove from. Providers are required to have a display_name and a provider_type. display_name will be the name shown to the user when opening the launcher. provider_type is a value that is used to differentiate which provider should be used and how the payload that is sent to Catena should be constructed. |

Upon a successful login through the launcher, the application specified in the config will be opened with the `session_id` received from Catena passed in as a command line argument.

**Important note:** The format for the command line argument is “-catsessionid [session-id]”.

### Motivation

Q: Why don’t we just send gRPC requests to the Catena platform directly?

A: At the time of writing, most versions of Unity do not support this. As such, we must use this model of `sending an HTTP request —> gRPC transcoding —> receiving an HTTP response —> parsing the HTTP response into a Protobuf response message`.

Q: What if a client is using a newer version of Unity that supports gRPC?

A: In this case, to take advantage of the performance benefit this would give us, we may choose to modify the Buf module to also generate **client stubs**. These would function similarly to the service clients we have now and the model would become `sending a gRPC request --> receiving a Protobuf response message`.

```csharp
// Parse the color from the player's json value
JObject test = player.Value.Value<JObject>();
if (test != null && test.TryGetValue("player-color", out JToken value))
{
    JArray playerColorArray = value as JArray;
    Color playerColor = new Color(playerColorArray[0].Value<float>(), playerColorArray[1].Value<float>(), playerColorArray[2].Value<float>());
    _playerColorDict.Add(player.Key, playerColor);
}
```

[Unity 2020.1 and protobuf]: https://github.com/protocolbuffers/protobuf/issues/9618#issuecomment-1185348928