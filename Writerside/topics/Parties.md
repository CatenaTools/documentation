# Parties


The Catena `PartiesService` is responsible for the creation and updating of parties as well as handling various party related operations e.g. a player joining a party using an invite code.

## How does Catena define a party?

A `Party` is comprised of a unique identifier `party_id`, the unique identifier of the player (account) designated as the leader of the party, a six-digit alphanumeric `invite_code` that other players can use to join the party, a list of `Player` objects, and map of key-value metadata pairs.

```protobuf
message Party {
	string party_id = 1;
	string leader_id = 2;
	string invite_code = 3;
	repeated Player players = 4;
	map<string,groups.EntityMetadata> metadata = 5;
}
```

## How does Catena define a player?

A `Player` is comprised of a unique identifier `player_id` that is the same as the player’s Catena `account_id`, a `display_name` that will be shown to other players in game, two boolean values `is_ready` and `is_leader` to track if the player has readied up or not and whether or not the player is the leader of the party respectively, a `team_number` that is assigned by a matchmaker, and a map of key-value metadata pairs.

```protobuf
message Player {
	string player_id = 1;
	string display_name = 2;
	bool is_ready = 3;
	bool is_leader = 4;
	int32 team_number = 5;
	map<string,groups.EntityMetadata> metadata = 6;
}
```

## How are parties created?

Party creation is facilitated through the use of the `CreateParty` RPC.

```protobuf
/* Creates a new party setting the player who requested its creation as its leader. */
rpc CreateParty(CreatePartyRequest) returns (CreatePartyResponse) {
    option (google.api.http) = {
		    post: "/api/v1/parties/create"
			  body: "*"
		};
};

message CreatePartyRequest {
    map<string,groups.EntityMetadata> creating_player_metadata = 1;
}

message CreatePartyResponse {
    Party party = 1; // newly created party
}
```

The method handler for `CreateParty` will create a unique identifier for the party, generate and assign it a new six-digit alphanumeric invite code, and set the player who requested its creation as the party leader.

The user who requested the party’s creation may also provide metadata specific to themselves as a player in this party in a `CreatePartyRequest`.

The newly created party will be returned in a `CreatePartyResponse`.

## Can parties be updated once created?

Information specific to a party itself (`party_id` & `invite_code`) will not change after its creation.

What may be updated, however, are a party’s metadata, the list of players it maintains, and who the leader of the party is (`leader_id`).

The players themselves may also update their own metadata as well as whether or not they are ready to begin matchmaking (`is_ready`).

All of these supported updates are facilitated through the use of the `UpdatePartyPlayer` RPC.

```protobuf
/* Updates information for a player in a party (and therefore information for the party itself). */
/* Note: Append operations are not currently supported when updating metadata with this method */
rpc UpdatePartyPlayer(UpdatePartyPlayerRequest) returns (UpdatePartyPlayerResponse) {
    option (google.api.http) = {
        put: "/api/v1/parties/update-player"
        body: "*"
    };
};

message UpdatePartyPlayerRequest {
    Player payload = 1; // payload specifying what player info. to update
	  google.protobuf.FieldMask payload_mask = 2;
}

message UpdatePartyPlayerResponse {
	    Party party = 1; // the updated party (that the updated player is a member of)
}
```

## How can players join a party?

Currently, the `PartiesService` only supports joining a party by entering the invite code of the party a player is attempting to join. This is facilitated through the use of the `JoinPartyWithInviteCode` RPC.

*Note: It is assumed that an existing member of the party will provide the player attempting to join with the party’s invite code.*

```protobuf
/* Allows a player to join a party by specifying its invite code. */
rpc JoinPartyWithInviteCode(JoinPartyWithInviteCodeRequest) returns (JoinPartyWithInviteCodeResponse) {
    option (google.api.http) = {
		    post: "/api/v1/parties/join"
			  body: "*"
		};
};

message JoinPartyWithInviteCodeRequest {
    string invite_code = 1; // code used to join the party
	  map<string,groups.EntityMetadata> joining_player_metadata = 2;
}

message JoinPartyWithInviteCodeResponse {
	  Party party = 1; // the party that was joined
}
```

In a `JoinPartyWithInviteCodeRequest`, a player must specify the invite code of the party they are attempting to join, and may optionally provide metadata specific to themselves as a player in this party.

If they were able to join the party successfully, they will be returned a `JoinPartyWithInviteCodeResponse` containing the `Party` object describing the party of which they are now a member.

**Note: There is currently no restriction on the number of players that may join a single party. That being said, parties attempting to enter matchmaking will have to be of a particular size depending on the game mode for which the matchmaker is attempting to form a match e.g. a max party size of three for a 3v3 game mode.**

## How can players leave a party?

The `PartiesService` supports a player leaving their party via the `LeaveParty` RPC.

```protobuf
/* Allows a player to leave their party and assigns a new leader if the leader leaves. */
rpc LeaveParty(LeavePartyRequest) returns (LeavePartyResponse) {
    option (google.api.http) = {
		    put: "/api/v1/parties/leave"
			  body: "*"
		};
};

message LeavePartyRequest {
    string party_id = 1; // ID of party to leave
}

message LeavePartyResponse {}
```

Currently, a `LeaveParty` request must contain the `party_id` of the party a player is attempting to leave. However, this *****must***** be the party of which they are currently a member.

*Note: If the player designated as the leader of the party leaves and they are the only player in the party, the party will be deleted. If there are other members in the party, another player will be designated as the new leader.*

## Can a party leader transfer their leadership to another player in their party?

Yes! If a leader wants to relinquish their leadership of the party to another party member, they may do so using `SetPartyLeader`.

**Note: Only current party leaders may call this RPC.**

```protobuf
/* Allows the current leader of a party to set a new leader. */
rpc SetPartyLeader(SetPartyLeaderRequest) returns (SetPartyLeaderResponse) {
    option (google.api.http) = {
		    put: "/api/v1/parties/set-leader"
			  body: "*"
		};
};

message SetPartyLeaderRequest {
    string player_id = 1; // ID of player to set as party leader
	  string party_id = 2; // ID of party to set leader of
}

message SetPartyLeaderResponse {}
```

## Can party leaders kick other players in their party?

Yes, again! By providing the `party_id` of their party and the `player_id` of the player they wish to kick in a `KickFromPartyRequest`, party leaders can leverage the `KickFromParty` RPC to kick a player from their party.

**Note: Only current party leaders may call this RPC. A party leader may not kick themselves.**

```protobuf
/* Allows the leader of a party to kick another player from their party. */
rpc KickFromParty(KickFromPartyRequest) returns (KickFromPartyResponse) {
    option (google.api.http) = {
		    put: "/api/v1/parties/kick"
			  body: "*"
		};
};

message KickFromPartyRequest {
    string player_id = 1; // ID of player to kick from party
	  string party_id = 2; // ID of party to kick player from
}

message KickFromPartyResponse {}
```

## Fetching party and player information

The `PartiesService` provides two RPCs for fetching a party’s info, and one RPC for fetching a party player’s info.

A party’s information can be fetched by providing the ID of the party via `GetPartyInfoByPartyId`.

```protobuf
/* Fetches information for a party given a party ID. */
rpc GetPartyInfoByPartyId(GetPartyInfoByPartyIdRequest) returns (GetPartyInfoResponse) {
    option (google.api.http) = {
		    get: "/api/v1/parties/{party_id}"
		};
};

message GetPartyInfoByPartyIdRequest {
    string party_id = 1;
}

message GetPartyInfoResponse {
    Party party = 1;
}
```

A party’s information can also be fetched via `GetPartyInfo` which determines the party the requesting account is currently a member of and returns its `Party` object.

```protobuf
/* Fetches party information for the party that the requesting player is currently a member of. */
rpc GetPartyInfo(GetPartyInfoRequest) returns (GetPartyInfoResponse) {
    option (google.api.http) = {
		    get: "/api/v1/parties"
		};
};

message GetPartyInfoRequest {}

message GetPartyInfoResponse {
    Party party = 1;
}
```

A party player’s information can be fetched via `GetPlayerInfoById` by providing the ID of the player whose information is desired.

```protobuf
/* Fetches information for a player in a party given an ID. */
rpc GetPlayerInfoById(GetPlayerInfoByIdRequest) returns (GetPlayerInfoByIdResponse) {
    option (google.api.http) = {
		    get: "/api/v1/parties/{player_id}"
		};
};

message GetPlayerInfoByIdRequest {
    string player_id = 1;
}

message GetPlayerInfoByIdResponse {
    Player player = 1;
}
```

## Party Update Notifications

Catena provides party players with the ability to subscribe to event notifications that are broadcast when updates are made to their party.

This ensures that each game client (player) will always have an up to date `Party` object describing the party of which they are currently a member.

A `PartyUpdateEvent` is structured as follows:

```protobuf
enum PartyUpdateType {
    PARTY_UPDATE_TYPE_UNSPECIFIED = 0;
    PARTY_UPDATE_TYPE_PLAYER_JOINED = 1;
    PARTY_UPDATE_TYPE_PLAYER_LEFT = 2;
    PARTY_UPDATE_TYPE_PLAYER_KICKED = 3;
    PARTY_UPDATE_TYPE_PLAYER_READY_CHANGED = 4;
	  PARTY_UPDATE_TYPE_PLAYER_LEADER_CHANGED = 5;
	  PARTY_UPDATE_TYPE_METADATA_CREATED = 6;
	  PARTY_UPDATE_TYPE_METADATA_UPDATED = 7;
	  PARTY_UPDATE_TYPE_METADATA_DELETED = 8;
}

message PartyUpdateEvent {
    PartyUpdateType type = 1;
    string message = 2;
    string party_id = 3; // The party id this event is associated with
	  map<string, PartyEventValue> event_payload = 4;
}

message PartyEventValue {
    oneof value {
		    bool bool_value = 1; 
		    groups.EntityMetadata metadata_value = 2;
		    string string_value = 3;
		    Player player_value = 4; 
		    int32 int_value = 5;
		    groups.MetadataMap metadata_map_value = 6;
	  }
}
```

As shown in the `PartyUpdateType` enum, party update event notifications are broadcast in response to the following events:

1. **A player joined the party**
2. **A player left the party**
3. **A player was kicked from the party**
4. **A player’s “ready” status changed**
5. **The party’s leader changed**
6. **New metadata was added to the party**
7. **Existing party metadata was updated**
8. **Existing party metadata was deleted**

## Metadata CRUD operations

The Catena `PartiesService` supports various CRUD operations for:

1. Adding a metadata entry to a party(`CreateMetadataEntry`)
2. Updating a metadata entry associated with a party (`UpdateMetadataEntry`)
3. Deleting a metadata entry associated with a party (`DeleteMetadataEntry`)
4. Getting **a single** metadata entry associated with a party (`GetMetadataEntry`)
5. Getting **all** metadata entries associated with a party (`GetMetadataEntries`)
