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

## Using the Catena Unity SDK

As a companion to the Unity SDK, there is a [Catena Networking Demo](https://github.com/CatenaTools/catena-networking-demo) that shows how to integrate Catena into a Unity project. The base of the demo is Unity's Galactic Kittens demo, with changes to add Authentication & Login, Matchmaking, and dedicated server support through Catena.
The following sections are guides on how to integrate the Catena SDK with your unity project, giving examples from the Catena Networking Demo.

### Authentication & Login

Before being able to use the matchmaking system, your players must be able to log in first.

#### Setup Steps

1. Add the components `CatenaEntrypoint` and `CatenaPlayer` to the scene you want to perform the login on.
    1. `CatenaEntrypoint` is a singleton class that is used by other Catena components to communicate with the Catena backend, such as `CatenaPlayer`. You can define what the endpoint URL is in the component, which defaults to `http://localhost:5000` - which is the default for the Catena backend as well. 
       
       Note: This component will make the game object it’s on persist between scenes, so it is best to either put it on its own game object, or on another game object you already plan to have persist between scenes. Also, if a Catena component requires an instance of `CatenaEntrypoint` , one will be created - but keep in mind that it will use the default endpoint URL if instantiated this way.
    2. `CatenaPlayer` is a component that interfaces with `CatenaEntrypoint` - you can call its functions for Logging in and out of accounts, and for entering matchmaking. It communicates to the entrypoint, and receives events in response. It also has events that are emitted when various Catena events happen. 
   
   {type="alpha-lower"}
2. Add a call to `CatenaPlayer.CompleteLogin`
    1. This function asks for a username and a password, and will attempt to log the player into the Catena backend.
       * Note: The password is currently unused, and the function just attempts an unsafe login with the username.
       * Note: Any username starting with the prefix ‘test’ will work as a dev solution, and will allow you to log in.
    
    {type="alpha-lower"}
3. Add a call to `CatenaPlayer.Logout`
    1. This function will log out of the current session.

   {type="alpha-lower"}
4. Subscribe to events from `CatenaPlayer`
    1. `OnAccountLoginComplete`  - This event is called when the call to `CompleteLogin` has returned successfully.
    2. `OnSessionInvalid` - This event is called on start by `CatenaPlayer`  if there is not already a session, as well as after logging out, failing to get an account, or failing to log in.

   {type="alpha-lower"}

#### Demo Example

TODO

### Matchmaking

Once the player is logged in, they are able to use the matchmaking system to enter a game through `CatenaPlayer`.

#### Setup Steps

1. Once the player is logged in with an account, add a call to `CatenaPlayer.EnterMatchmaking`
    1. This function asks for two arguments - a both of which are Dictionaries mapping a string to `EntityMetaData`. `EntityMetaData`  is a class to contain a payload that will be sent to the Catena backend - which can be of type `string`, `int`, or `Json` . (The Json payload type is also defined as a string in Unity, but will be interpreted as Json data by the Catena backend)
    2. The second dictionary is for the match metadata - made for defining what type of match the player is searching for.


        | ID field | Metadata example | Description |
        | --- | --- | --- |
        | queue_name | “team_of_2” | This is defining which matchmaking queue you want the player to search in - defined in the Catena config. |
        | TODO |  |  |
    3. The first dictionary is for the player’s metadata - which will provide information about the player to the matchmaker.
        
        
        | ID field | Metadata example | Description |
        | --- | --- | --- |
        | address | The player’s local address, and local host port. |  |
        | TODO |  |  |

    {type="alpha-lower"}
2. Subscribe to the event `OnFindingServer` from `CatenaPlayer`
    * This event is invoked with a string included. The event is invoked when one of the following happens, after trying to find a match:
        1. The client failed to find a match. *(The string provided will be null)*
        2. The client found a match while matchmaking, and is now waiting for a dedicated server. *(The string provided will be empty)*
        3. The client found a match that is hosted by another player. *(The string provided will have the connection info of the host)*

#### Demo Example

TODO

### Dedicated Server

This guide assumes that you have a dedicated server build of your game already working, and instead describes how to have your build register itself with Catena, and become available as a matchmaking option for players.

#### Setup Steps

First, we will make the necessary changes to the client so that it can connect to a dedicated server instead of another host player. The only client-specific change needed is to subscribe to the event `OnFoundServer` from `CatenaPlayer`

This event is called when a client had already found a match while matchmaking, and has now found a dedicated server. It is invoked with a string, which contains the IP and port of the dedicated server, separated by a ‘:’ character.

Next, these are the steps needed to have your dedicated server work with Catena:

1. To the scene where your server is ready to accept connections, add the component `CatenaSingleMatchGameServer`
    1. `CatenaSingleMatchGameServer` is a singleton component, and that interfaces with the `CatenaEntrypoint`  instance to communicate with the Catena Backend for your dedicated server.
    2. Note: This component will make the game object it’s on persist between scenes, so it is best to either put it on it’s own game object, or on another game object you already plan to have persist between scenes. Also, if a Catena component requires an instance of `CatenaEntrypoint` , one will be created - but keep in mind that it will use the default endpoint URL if instantiated this way.

   {type="alpha-lower"}
2. Add a call that is only executed on your dedicated server to `CatenaSingleMatchGameServer.GetMatch();`  whenever your dedicated server is ready to start a match.
   * You can get the current instance of the component with `CatenaSingleMatchGameServer.Instance`
3. Add a call to the dedicated server to `CatenaSingleMatchGameServer.EndMatch();` . This should be called either when the match is finished, or there is no clients connected to the game.

Your dedicated server should now be set up to accept connections through Catena. When the Catena matchmaking broker creates a match, it will find an available server, and send that server the match and player information - and send the players the address of the dedicated server.

#### Demo Example

TODO

### Frequently Asked Questions

TODO

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