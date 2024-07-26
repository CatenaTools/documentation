# Matchmaking

The Catena matchmaker is responsible for grouping players or parties together to form games or matches. This may be done based on data sent by clients or retrieved from other sources and rules configured in the matchmaking service. The matchmaker can support both peer-to-peer and dedicated server arrangements.

Catena decouples matchmaking from server assignment and fleet and capacity management; these tasks are handled by the [](Match-Broker.md).

## Overview

To begin, each player or a party leader submits a ticket to the matchmaker. This ticket may take many forms and can include metadata about the players, the party, or the match such as the desired map, character classes, etc.

Central to the design of the matchmaker are one or more queues that can be configured based on the needs of the game to partition matchmaking tickets. These queues act almost like mini-matchmakers, supporting many different game types and matchmaking needs and improving scale:

 * Each queue can have its own rules for match sizes, teams, <tooltip term="matchmaking strategy">matchmaking strategy</tooltip> or other custom requirements.
 * Tickets are never moved or matched between queues.
 * The matchmaker will assign each ticket to a queue based on the metadata it contains.

When a matchmaking strategy forms a complete match, two events are emitted:

1. All the clients on all the tickets in the match receive a matchmaking complete/finding server event
2. A new match event is broadcast so other services like the [](Match-Broker.md) can find or start servers if necessary

## Configuring basic queues

With many possible scenarios for matchmaking, there are many options available for the matchmaker. First, here is an example of a basic configuration with two queues: 

```json
"Catena": {
    "Matchmaker": {
        "MatchmakingQueues": {
            "1v1": {
                "QueueName": "1v1",
                "Teams": 2,
                "PlayersPerTeam": 1,
                "TicketExpirationSeconds": 180
            },
            "2v2": {
                "QueueName": "2v2",
                "Teams": 2,
                "PlayersPerTeam": 2,
                "TicketExpirationSeconds": 180
            }
        }
    },
    "StatusExpirationMinutes": 15
}
```

There are many different ways these queues could be utilized by a game, for example:

* a 1v1 match could be satisfied by a lobby-style ticket containing two players
* a 1v1 match could be satisfied by two separate players each submitting their own individual ticket
* a 2v2 match could be satisfied by two parties, each with two players
* (and many more)

A client submits a ticket with the queue name in the match metadata. (More information on securing this process is available in [](#custom-hooks).)

In each queue, there is an amount of time to attempt matchmaking (`TicketExpirationSeconds`). If this time elapses, the ticket will be dropped and the clients will be notified that matchmaking timed out.

The `StatusExpirationMinutes` option is the amount of time the matchmaker should hold onto tickets that have completed matchmaking or timed out. This can be adjusted for scaling and game needs like match reconnect or preventing players from matchmaking too quickly.

The Catena matchmaker contains built-in strategies that will be selected automatically based on team counts and sizes.

Advanced uses like team assignment, restricted queues, deadline scheduling, or peer-to-peer matches (to name a few) require the use of [](#deadline-scheduling), [](#custom-hooks), or [](#custom-strategies).

## Custom hooks

Matchmaker custom hooks allow validation or manipulation of the data used in various stages of the matchmaker. Some examples are provided in the `Extras/MatchmakingHooks` directory. For customization of the actual matching algorithm or rules, see [](#custom-strategies).

A single custom hook object may be assigned to the matchmaker and a custom hook object may be assigned separately to each queue. Some hook methods are only run by the matchmaker, regardless of the queue, such as the `ValidateTicketMatchHook` and some hook methods are only run per-queue, such as the `NewMatchHook`.

> A single hook object may be used for all queues and the matchmaker; they do not need to be independent.

```json
"Catena": {
    "Matchmaker": {
        "MatchmakingQueues": {
            "1v1": {
                "QueueName": "1v1",
                "Teams": 2,
                "PlayersPerTeam": 1,
                "TicketExpirationSeconds": 180,
                "CustomHooks": "QueueLevelHook"
            }
        }
    },
    "StatusExpirationMinutes": 15,
    "CustomHooks": "MatchmakerLevelHook"
}
```

### ValidateTicketMatchHook

This hook is run both when creating and submitting a matchmaking ticket. Since it is executed for each ticket before any required internal validation, it is an opportunity to set/override required parameters such as the queue name, set data that is not available in the game client, or validate game-specific data set by the client.

This hook is only run by the matchmaker before queuing a ticket; the queue is not known during this hook unless it is sent by the client or selected and set by the hook itself.

This hook can reject a ticket which will cause an error to be returned to the client and the ticket will not be queued.

### NewMatchHook

This hook is run after a matchmaking strategy forms a match but before any events are sent. This hook may be used to set data that will be sent in the events. This hook can also reject a match which will cause the tickets will be requeued.

This hook is only run per-queue, not for the matchmaker. To use this hook, the object must be set for each queue it is intended to act upon.

## Custom strategies

Custom matchmaking strategies can be defined per queue for custom game/team arrangements or to take additional data, from tickets or other sources, into account while forming matches.

A custom strategy contains a single method, `TryMakeMatch()` that will be called by the matchmaker once per cycle. When this method is called, the strategy itself will have the context of the matchmaker and the queue which it can use to attempt to make up to one match.

Tickets may be picked up from the queue by calling either:

* `Matchmaker.DequeueActiveTicket()` to return 0 or 1 ticket
* `Matchmaker.DequeueActiveTickets(count)` to return 0 up to _count_ tickets

If a match can be formed, it should be emitted by calling `Matchmaker.EmitMatch()` with the list of tickets forming the match.

Tickets that have been dequeued by a matchmaking strategy must either be emitted in a match or returned to the queue with either `Matchmaker.RequeueActiveTicket()` or `Matchmaker.RequeueActiveTickets()` **otherwise those tickets will be lost**.

Matchmaking strategies should only make one match in each cycle/call to `TryMatchMatch()`.

## Deadline scheduling

Deadline scheduling is an approach to queueing where an object remains queued until it meets some requirement(s) OR a deadline is reached. For matchmaking, deadlines can be useful to ensure players get into matches in a reasonable amount of time even if full matches can't be made.

By default, with a ticket expiration (`TicketExpirationSeconds`) of 3 minutes, if a full match can't be made within 3 minutes, then that ticket will time out and matchmaking will fail. However, if a partial match is possible, then `ExpirationAction` can be set on any applicable queue, enabling the queue matchmaking strategy to operate on expired tickets just before they are removed.

```json
"Catena": {
    "Matchmaker": {
        "MatchmakingQueues": {
            "up_to_3": {
                "QueueName": "up_to_3",
                "Teams": 0,
                "PlayersPerTeam": 3,
                "TicketExpirationSeconds": 180,
                "ExpirationAction": "minimum"
            }
        }
    },
    "StatusExpirationMinutes": 15
}
```

Since the expiration action must be supported by the strategy, no particular expiration action is guaranteed. However, all the built-in strategies in Catena support the following:

| ExpirationAction | Description                                                                                                                                                                                                                                                                                           |
|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| minimum          | Emits a match using the minimum number of expired tickets.<br/>This may be preferred so other players whose tickets have not expired still have a chance to be put in better fit or full matches later. It may lead to an overall higher match count which may increase server capacity requirements. |

## Starting matchmaking

Creating a ticket for matchmaking is done by submitting an `Entity`. Since entities can contain metadata and can be nested there are many different arrangements supported by the Catena matchmaker to accommodate structuring game data relevant to matchmaking.

The entity metadata may be used to store game data such as character, map, or item selection for matches, players, or parties. Metadata passed to the matchmaker can be used during matchmaking and/or passed on to the clients or game servers in a match.

> The Catena matchmaker requires that the top-level entity contain `queue_name` in the metadata, so it can assign the ticket to the correct queue. This may be sent by a client (not necessarily safe) or set/overridden within a [custom hook](#custom-hooks) based on ticket data (an example of this is contained in `QueueAssignmentMatchmakingHooks`).

### Single player example

In this example a player submits a ticket for themselves to enter the 1v1 queue via [StartMatchmaking](#startmatchmaking-api-example)

The additional `map` metadata could be used by a [custom strategy](#custom-strategies) to ensure this player is only matched with another player choosing the same map. It could also be added to the match data by a [custom hook](#custom-hooks), either to select a server already running that map or to send the map choice to the server to load it when it receives the match.

```json
{
    "entity": {
        "id": "account-d953e83f-b567-4bf1-a628-2c2777ecc58",
        "entity_type": "ENTITY_TYPE_ACCOUNT",
        "metadata": [
            {
                "key": "queue_name",
                "value": {
                    "string_payload": "1v1"
                }
            },
            {
                "key": "map",
                "value": {
                    "string_payload": "the_woods"
                }
            }
        ]
    }
}
```

#### StartMatchmaking API example {collapsible="true"}
<api-endpoint openapi-path="../apispec/openapi/api/v1/catena_matchmaking.swagger.json" endpoint="/api/v1/matchmaking/start" method="POST" collabsible = "true">
    <request>
{
    "entity": {
        "id": "account-d953e83f-b567-4bf1-a628-2c2777ecc58",
        "entity_type": "ENTITY_TYPE_ACCOUNT",
        "metadata": [
            {
                "key": "queue_name",
                "value": {
                    "string_payload": "1v1"
                }
            },
            {
                "key": "map",
                "value": {
                    "string_payload": "the_woods"
                }
            }
        ]
    }
}
    </request>
</api-endpoint>

### Party example

In this example, one of the two party members would submit a ticket for both players to be matched with another pair of players.

The `queue_name` is specified at the top-level but each player also has their own character selection that can be included in player data sent to the server by a [custom hook](#custom-hooks).

```json
{
    "entity": {
        "id": "party-91395716-e454-4b18-93ab-567b2bfd71df6",
        "entity_type": "ENTITY_TYPE_GROUP",
        "entities": [
            {
                "id": "account-6e68514d-e3b7-4944-a861-588406c4c21",
                "entity_type": "ENTITY_TYPE_ACCOUNT",
                "metadata": [
                    {
                        "key": "character",
                        "value": {
                            "string_payload": "waffles"
                        }
                    }
                ]
            },
            {
                "id": "account-0f365672-1cad-4283-8d09-43d8d4a31a03",
                "entity_type": "ENTITY_TYPE_ACCOUNT",
                "metadata": [
                    {
                        "key": "character",
                        "value": {
                            "string_payload": "spike"
                        }
                    }
                ]
            }
        ],
        "metadata": [
            {
                "key": "queue_name",
                "value": {
                    "string_payload": "2v2"
                }
            }
        ]
    }
}
```

### Typical entity hierarchies

> Group and Match entities are treated the same way by the Catena matchmaker; both may contain either all Account or all Group entities but not a mix.
>
> A Match entity does not imply the ticket will not be matched with other tickets if the queue settings warrant it.
>
> A [custom matchmaking strategy](#custom-strategies) may treat these entities differently if necessary.

| Typical entity hierarchy                                                                                                                                                                                                                                                                                                                                                                                               | Description                                                                  | When to use                                                                                                                                                                                                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Account                                                                                                                                                                                                                                                                                                                                                                                                                | Single player/account                                                        | A player enters a matchmaking queue solo and there is no metadata significant to the matchmaking strategy. For example, a player entering a queue to be backfilled into a lobby server.                                                                                                                              |
| Match<ul><li>Account</li></ul><br/>*or*<br/>Group<ul><li>Account</li></ul>                                                                                                                                                                                                                                                                                                                                             | Group or match containing a single player/account                            | A player enters a matchmaking queue solo and there is game data such as a map that's necessary to match with other players **and** separate player data that will be sent to the server such as character selection or load-out.<br/>This hierarchy can be flattened to just an account if preferred.                |
| Group<ul><li>Account</li><li>Account</li><li>...</li></ul><br/>*or*<br/>Match<ul><li>Account</li><li>Account</li><li>...</li></ul>                                                                                                                                                                                                                                                                                     | A match, group, or party of players with all the accounts/players identified | A party enters a matchmaking queue to be matched with other players.<br/>*or*<br/>Lobby-style matchmaking where players have setup the match and only need it validated before assigning it to a server.                                                                                                             |
| Group<ul><li>Group<ul><li>Account</li></ul><ul><li>Account</li></ul><ul><li>...</li></ul></li><li>Group<ul><li>Account</li></ul><ul><li>Account</li></ul><ul><li>...</li></ul></li><li>...</li></ul><br/>*or*<br/>Match<ul><li>Group<ul><li>Account</li></ul><ul><li>Account</li></ul><ul><li>...</li></ul></li><li>Group<ul><li>Account</li></ul><ul><li>Account</li></ul><ul><li>...</li></ul></li><li>...</li></ul> | Group of groups with players specified                                       | Lobby-style matchmaking where players have setup a match with structured teams and only need it validated before assigning it to a server.<br/>*or*<br/>A group enters a queue to me matched with another group where each group has self-arranged into teams. (There the subgroups in each ticket represent teams.) |
| Group<br/>*or*<br/>Match                                                                                                                                                                                                                                                                                                                                                                                               | A match, group, or party of players without the accounts/players identified  | All game data is stored elsewhere and referenced by the ID or a flat structure is preferred and game and player data is stored only in metadata.                                                                                                                                                                     |
| Group<ul><li>Group</li><li>Group</li><li>...</li></ul><br/>*or*<br/>Match<ul><li>Group</li><li>Group</li><li>...</li></ul>                                                                                                                                                                                                                                                                                             | Group of groups without players specified                                    | Lobby-style matchmaking where players self-assign teams **and** all game data is stored elsewhere and referenced by the ID or a flatter structure is preferred and game data is stored only in metadata.                                                                                                             |

**There are other possible hierarchies that may be accepted by the matchmaker, but they are not recommended at this time.**