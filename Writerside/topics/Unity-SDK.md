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
The following sections are guides on how to integrate the Catena SDK with your unity project, using the Catena Networking Demo as an example.

### Authentication & Login

Before being able to use the matchmaking system, your players must be able to log in first.

#### Authentication & Login: Setup Steps

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

#### Authentication & Login: Demo Example

In the Catena Galactic Kittens demo, navigate to the scene `Assets//Scenes//Menu.unity`. In this scene, you will see the gameobjects CatenaEntrypoint and CatenaPlayer, which have their respective components added.

On the MenuController gameobject, you will see the Menu Manager component, which has a few changes made for logging in/out.

For logging in, clicking on the login button calls this coroutine in `MenuManager`:
```csharp
// Parse the color from the player's json value
private IEnumerator Login()
{
    if (UsernameField == null)
    {
        print("Missing username; cannot login");
        yield break;
    }

    // capture input field values
    var username = UsernameField.text;
    var password = PasswordField.text;

    if (string.IsNullOrEmpty(username))
    {
        print("Empty username; cannot login");
        yield break;
    }

    var catenaPlayer = FindObjectOfType<CatenaPlayer>();
    catenaPlayer.CompleteLogin(username, password);
}
```

For logging out, clicking on the logout button after logging in calls this coroutine in `MenuManager`:
```csharp
private IEnumerator Logout()
{
var catenaPlayer = FindObjectOfType<CatenaPlayer>();
catenaPlayer.Logout();
yield break;
}
```

The Menu Manager also subscribes to Login/Logout events, as shown here:
```csharp
var catenaPlayer = FindObjectOfType<CatenaPlayer>();
print($"Setup Catena UI events using CatenaPlayer {catenaPlayer.GetHashCode()}");

catenaPlayer.OnAccountLoginComplete += (_, _) =>
{
    print("Online (showing online controls)");
    showOnlineControls = true;
};

catenaPlayer.OnSessionInvalid += (_, _) =>
{
    print("Offline (showing login controls)");
    showLoginControls = true;
};
```

### Matchmaking

Once the player is logged in, they are able to use the matchmaking system to enter a game through `CatenaPlayer`.

#### Matchmaking: Setup Steps

1. Once the player is logged in with an account, add a call to `CatenaPlayer.EnterMatchmaking`
    1. This function asks for two arguments - a both of which are Dictionaries mapping a string to `EntityMetaData`. `EntityMetaData`  is a class to contain a payload that will be sent to the Catena backend - which can be of type `string`, `int`, or `Json` . (The Json payload type is also defined as a string in Unity, but will be interpreted as Json data by the Catena backend)
    2. The first dictionary is for the player’s metadata - which will provide information about the player to the matchmaker.

        | ID field | Metadata example | Description |
        | --- |------------------| --- |
        | address | "192.0.2.1:5000" | The player’s local address, and local host port. |
    3. The second dictionary is for the match metadata - made for defining what type of match the player is searching for.

        | ID field | Metadata example | Description |
        | --- | --- | --- |
        | queue_name | “team_of_2” | This is defining which matchmaking queue you want the player to search in - defined in the Catena config. |

    {type="alpha-lower"}
2. Subscribe to the event `OnFindingServer` from `CatenaPlayer`
    * This event is invoked with a string included. The event is invoked when one of the following happens, after trying to find a match:
        1. The client failed to find a match. *(The string provided will be null)*
        2. The client found a match while matchmaking, and is now waiting for a dedicated server. *(The string provided will be empty)*
        3. The client found a match that is hosted by another player. *(The string provided will have the connection info of the host)*

#### Matchmaking: Demo Example

Similar to account login/logout, the MenuController gameobject in `Assets//Scenes//Menu.unity` has changes to enable matchmaking.

Once logged in, the player can click the Find button, which will call this coroutine:
```csharp
private IEnumerator Find()
{
    var local = NetworkManager.Singleton.GetComponent<UnityTransport>().ConnectionData;
    var address = $"{local.Address}:{GetLocalHostPort()}";
    // TODO: make match size selectable
    var matchMetadata = new Dictionary<string, EntityMetadata> { {"queue_name", new EntityMetadata{ StringPayload = "team_of_2"} } };
    var playerMetadata = new Dictionary<string, EntityMetadata> { { "address", new EntityMetadata{ StringPayload = address} } };

    var catenaPlayer = FindObjectOfType<CatenaPlayer>();
    catenaPlayer.EnterMatchmaking(playerMetadata, matchMetadata);

    yield break;
}
```

To be informed when a match is found, Menu Manager also subscribes to the event `OnFindingServer`:
```csharp
catenaPlayer.OnFindingServer += (_, success) =>
{
    // This event is driven off a separate thread; can't update UI directly

    if (success == null)
    {
        print("Failed to find a match");
        return;
    }

    if (success == "")
    {
        print("Found a match; waiting for a server");
        SetMatchmakingStatus("Found match; waiting for a server...");
        return;
    }

    print($"Found a match - data: {success}");

    ConnectionInfo connectionInfo;
    try
    {
        connectionInfo = JsonConvert.DeserializeObject<ConnectionInfo>(success);
    }
    catch (Exception e)
    {
        print($"Failed to deserialize connection info: {e}");
        return;
    }

    print($"Connection info: {connectionInfo}");

    if (connectionInfo.ServerId == catenaPlayer.Account.Id)
    {
        print("This client is hosting");
        SetMatchmakingStatus("Starting match...");
        transitionHost = true;
    }
    else
    {
        print($"This client is connecting to {connectionInfo.Ip}:{connectionInfo.Port}");
        SetMatchmakingStatus("Found match, connecting...");
        transport.SetConnectionData(connectionInfo.Ip, connectionInfo.Port);
        transitionClient = true;
    }
};
```

### Dedicated Server

This guide assumes that you have a dedicated server build of your game already working, and describes how to have your build register itself with Catena, and become available as a matchmaking option for players. The Catena Unity Demo has already made the necessary modifications for the Galactic Kittens game to be built as a dedicated server.

#### Dedicated Server: Setup Steps

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

#### Dedicated Server: Demo Example

In the Catena Galactic Kittens demo, there are a few changes made to allow dedicated server support. When running as a dedicated server, the server immediately goes to the Character Selection scene, where it waits for other players to join. So, as the Character Selection scene is the default menu, that's where most of the Catena changes are located.

First, for setting up the clients to connect to the dedicated server, code was added to the `MenuManager` component to subscribe to the `OnFoundServer` event. The process is similar to joining a player-hosted server:
```csharp
catenaPlayer.OnFoundServer += (_, success) =>
{
    // This event is driven off a separate thread; can't update UI directly

    var parts = success.Split(":");
    var ip = parts[0];
    var port = ushort.Parse(parts[1]);
    print($"This client is connecting to server at {ip}:{port}");
    SetMatchmakingStatus("Found server, connecting...");
    transport.SetConnectionData(ip, port);
    transitionClient = true;
};
```

There are also changes made for the dedicates server build to register itself with Catena, making it open to have players connect for matches. If you navigate to the scene `Assets//Scenes//CharacterSelection.unity`, you will see the added gameobject/component for `CatenaSingleMatchGameServer`.

The call to GetMatch can be seen in `CharacterSelectionManager.cs`, in the Start function:
```csharp
void Start()
{
    if (IsServer && !IsClient)
    {
        print("Dedicated server starting");
        // Removed non-Catena startup logic from this code snippet...
        print("Using Catena to obtain match");
        m_catenaGameServer = CatenaSingleMatchGameServer.Instance;
        m_catenaGameServer.GetMatch();
    }
}
```

For ending the match, there is a prefab located at `Assets//Prefabs//EndGameManager.prefab`, which handles the end of the match in both the Victory and Defeat scenes. For the dedicated server, it waits until the clients disconnect before shutting down, seen in `EndGameManager.cs`. In this example, the function is called from the Unity Network Manager's `OnClientDisconnectCallback` event, and `m_connectedClients` containing the list of connected client Id's:
```csharp
private void OnClientDisconnect(ulong clientId)
{
    // Event only fired on dedicated server

    m_connectedClients.Remove(clientId);

    if (m_connectedClients.Count == 0)
    {
        m_catenaGameServer.EndMatch();
    }
}
```

### Catena Events

For knowing when various steps of the Catena flow are complete, there are events that can be subscribed to that provide information about accounts, metadata, etc.

Here's a list of events that are available to subscribe to from `CatenaPlayer`:

| Event name | When Invoked                                                                                                                                   | Payload Provided                                                                                                   |
|------------|------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| OnAccountLoginComplete | When the player is logged in                                                                                                                   | `Catena.CatenaAccounts.Account` - The player's account information                                                 |
| OnSessionInvalid | When CatenaPlayer is started if there is not already a Catena session running, when failing to log in, or when logging out.                    | None                                                                                                               |
| OnFindingServer | When a match is found and the player is waiting for a server, or when a match is found from a non-dedicated server, or when matchmaking fails. | `string` - null if failed to find a match, empty if waiting for a server, otherwise has the IP, Port and Server ID |
| OnFoundServer | When a dedicated server has been found                                                                                                         | `string` - The IP, Port, and Server ID                                                                             | 

CatenaEntrypoint also has many events available to be subscribed to:

| Event name | When Invoked                                                                         | Payload Provided                                                                                  |
|------------|--------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| OnCreateOrGetAccountByTokenCompleted | When a new Catena account is created, or when getting an existing one.               | `AccountEventArgs` - Result of the request, and the account info.                                 |
| OnGetAccountByIdCompleted | When an existing Catena account is recieved.                                         | `AccountEventArgs` - Result of the request, and the account info.                                 |
| OnAccountCreateMetadataEntryCompleted | When new metadata has been added for an account.                                     | `CatenaEventStatus` - Result of the request.                                                      |
| OnAccountUpdateMetadataEntryCompleted | When an entry of an account's metadata has been updated.                             | `CatenaEventStatus` - Result of the request.                                                      |
| OnAccountDeleteMetadataEntryCompleted | When an entry of an account's metadata has been deleted.                             | `CatenaEventStatus` - Result of the request.                                                      |
| OnAccountGetMetadataEntryCompleted | When a request for the metadata of an entry has completed.                           | `MetadataEntryArgs` - Result of the request, and the account metadata of the requested entry.     |
| OnAccountGetMetadataEntriesCompleted | When a request for all the metadata of an account has completed.                     | `MetadataEntriesArgs` - Result of the request, and the account metadata of the requested entries. |
| OnLoginWithProviderCompleted | When the request to log in with a provider has been completed.                       | `CatenaEventStatus` - Result of the request.                                                      |
| OnLogoutCompleted | When the request to log from Catena has been completed.                              | `CatenaEventStatus` - Result of the request.                                                      |
| OnCreatePartyCompleted | When a request to create a new Catena party has been completed.                      | `PartyEventArgs` - Result of the request, and the party info.                                     |
| OnUpdatePartyPlayerCompleted | When a request to update a player in the party has been completed.                   | `PartyEventArgs` - Result of the request, and the party info.                                     |
| OnJoinPartyWithInviteCodeCompleted | When a request to join a Catena party using an invite code has been completed.       | `PartyEventArgs` - Result of the request, and the party info.                                     |
| OnGetPartyInfoCompleted | When a request to get the info of the current Catena party has been completed.       | `PartyEventArgs` - Result of the request, and the party info.                                     |
| OnGetPartyInfoByPartyIdCompleted | When a request to get the info of a Catena party with a given Id has been completed. | `PartyEventArgs` - Result of the request, and the party info.                                     |
| OnSetPartyLeaderCompleted | When a request to set a new party leader has been completed.                         | `CatenaEventStatus` - Result of the request.                                                      |
| OnLeavePartyCompleted | When a request to leave the current Catena party has been completed.                 | `CatenaEventStatus` - Result of the request.                                                      |
| OnKickFromPartyCompleted | When a request to kick a member of the current Catena party has been completed.      | `CatenaEventStatus` - Result of the request.                                                      |
| OnPartyCreateMetadataEntryCompleted | When a request to create new metadata for the party has been completed.              | `CatenaEventStatus` - Result of the request.                                                      |
| OnPartyUpdateMetadataEntryCompleted | When a request to update a metadata entry for the party has been completed.          | `CatenaEventStatus` - Result of the request.                                                      |
| OnPartyDeleteMetadataEntryCompleted | When a request to delete a metadata entry for the party has been completed.          | `CatenaEventStatus` - Result of the request.                                                      |
| OnPartyGetMetadataEntryCompleted | When a request to get a metadata entry from the party has been completed.            | `MetadataEntryArgs` - Result of the request, and the party metadata of the requested entry.       |
| OnPartyGetMetadataEntriesCompleted | When a request to get all the metadata entries from the party has been completed.    | `MetadataEntriesArgs` - Result of the request, and the party metadata of all it's entries.        |

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