# Match Broker
A match broker coordinates matches across services and components; it sits between things like matchmakers, game clients, game servers, server managers, and capacity managers. In its simplest form, the match broker can be considered a queue or set of queues for matches.

Decoupling the match broker from the matchmaker allows them both to operate independently. Neither service absolutely requires the other and each one can be swapped out if necessary. Matchmakers can be simplified as they do not necessarily need to be aware of game servers, capacity, or how to allocate servers when a match broker is used.

Specifically, a match broker is responsible for picking up match-containing `new-match` events produced by a matchmaker and presenting them to or signalling to other components such as game clients or servers. A match broker is not defined by an API, but by this common event type/tag and a common `Match` payload. A match broker implementation only needs to adhere to utilizing these types/tags. The default match broker is `CatenaMatchBroker`.

[Match broker `Match` reference](https://github.com/CatenaTools/catena-tools-core/blob/main/catena-tools-core/Protos/api/v1/match_broker.proto)

## Catena Match Broker

The Catena default match broker implementation is an "integrated" match broker. It handles the minimum requirement of queuing matches produced by a matchmaker, and it also handles server capacity tracking and implements the [server manager](Server-Manager.md) interface for game servers. This gives it a holistic view of outstanding matches, running servers, and the ability to dynamically fit server capacity to match demand across multiple fleets/servers/infrastructure. To enable this, the Catena match broker supports filtering matches at its front-end, when selecting which <tooltip term="Allocator">allocator</tooltip> to use, and when game servers request matches.

### Configuration

#### Allocator configuration

> A full list of allocators and configuration options is available here: [](Match-Broker-Allocators.md).

The Catena match broker optionally supports using one or more allocators to maintain game servers. This starts with adding one or more allocator configurations to the match broker. Each allocator has its own configuration options. Here is an example of a multi-instance configuration with two uses of the `CatenaLocalBareMetalAllocator`:

```json
"MatchBroker": {
  "Allocators": [
    {
      "Allocator":"CatenaLocalBareMetalAllocator",
      "AllocatorDescription": "Lostwoods allocator",
      "Configuration": {
        "GameServerPath": "<the path to the game server>",
        "GameServerArguments": "-map lostwoods",
        "ReadyDeadlineSeconds": 30,
        "ReaperConfiguration": {
          "AllocatorReaperPeriodSeconds": 10,
          "MaxRunTimeMinutes": 125
        }
      },
      "Requirements": "MatchProperties.map == 'lostwoods'"
    },
    {
      "Allocator":"CatenaLocalBareMetalAllocator",
      "AllocatorDescription": "Forbidden forest allocator",
      "Configuration": {
        "GameServerPath": "<the path to the game server>",
        "GameServerArguments": "-map forbidden_forest",
        "ReadyDeadlineSeconds": 45,
        "ReaperConfiguration": {
          "AllocatorReaperPeriodSeconds": 10,
          "MaxRunTimeMinutes": 125
        }
      },
      "Requirements": "MatchProperties.map == 'forbidden_forest'"
    }
  ]
}
```

Since an allocator can be used multiple times, each with a different configuration or requirements, the configuration of each allocator instance is specified within an entry in the list. The example above includes two instances of the same allocator for demonstration purposes. Each instance loads a different map by passing an argument to the game server process. The expectation that one map takes longer to load is reflected in the `ReadyDeadlineSeconds` that's presented to the [server manager](Server-Manager.md) to ensure game servers start.

A requirement is added to each instance so the correct game server process will be started when a particular map is required for a match. This example shows how a <tooltip term="JMESPath">JMESPath</tooltip> expression can be used to check that a game-specified match property (`map`) is set to a certain value.

> Since requirement filters can be used for allocators **and** [by servers when requesting matches](Server-Manager.md#requirements-filter), it's important not to create conflicting constraints or over-constrain an allocator or a server. This can result in matches that don't get picked up either because a server isn't started or a server starts but filters out the matches.
>
> For example, using the configuration above, if a match has its map set to "forbidden_forest", then a server will be started by the "Forbidden forest allocator". However, if this server is using a filter for a different map when requesting matches (ex: `MatchProperties.map == 'town_square'`), then the match won't be picked up and run. Instead, the match and/or the server will [time out](Match-Broker.md#fail-safe-timer-configuration).

#### Fail-safe timer configuration

There are several timers the Catena match broker implementation uses to handle various failures that can occur during real-world operation, when networks go down or servers experience hardware failures, for example. The name of each timer includes the unit of time, indicating the relative scale of the timer. For example, match run times are typically on the order of many minutes but matches should only be queued before they are picked up on the order of seconds.

```json
"MatchBroker": {
  "ServerMaxLifetimeMinutes": 10,
  "MatchPickupTimeSeconds": 90,
  "MatchReadyTimeSeconds": 30,
  "MatchMaxRunTimeMinutes": 120
}
```

##### ServerMaxLifetimeMinutes

This is the maximum amount of time a server known to the backend should be allowed to run. At the point this timer expires for a given game server, the backend will treat that server as having failed since the broker otherwise should have received from message from the game server before this timer expired. The broker will attempt to clean up/deallocate the server and any matches it was believed to be running at that point.

##### MatchPickupTimeSeconds

The maximum amount of time a match can be queued by the broker before a server obtains the match. Hitting this timer may indicate servers didn't start, couldn't be allocated, or there are network communication, load, or game server SDK integration issues that prevented the timely arrival of `RequestMatch` calls from game servers. This may also happen if the only servers requesting matches are filtering out the matches that are expiring by this timer.

The broker will signal to clients that a server could not be obtained for their match.

##### MatchReadyTimeSeconds

This timer starts when a match is picked up by a game server. Its expiration indicates that a game server did not call `MatchReady` in a timely manner. Without receiving `MatchReady` from the game server that should run the match, the broker has no connection details to return to clients and must instead signal to clients that a server could not be obtained for their match. Additionally, it can treat the server as failing/failed and attempt to clean up/deallocate it.

##### MatchMaxRunTimeMinutes

This is the maximum amount of time a match can run before a server should call `EndMatch`. It should be slightly longer than the longest match should actually run and never trigger; it should only happen if a server fails or fails to communicate with the backend. The broker can treat the server as having failed and attempt to clean up/deallocate the server and will signal to clients that the match is over or has failed.