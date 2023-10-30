# Auth Flow & Session Management

# Overview

Metadata is a derivation from the [Entity System](https://www.notion.so/Feature-Entities-30c7eb876730482f9349cfb61051d801?pvs=21) and understanding the rationality of the Entity System is important to understanding the reasoning of Metadata. Metadata serves as a way to attach data to certain data types, including Accounts, Parties, and Entities.

**An important note**: Metadata is **not transient when using default Catena services.** This means metadata that is attached to a type (like Catena’s `Account` type), will persist in a backing database and any unneeded metadata entries should be cleaned up. ****The Metadata structure is found in `groups.proto` and looks like:

```protobuf
message EntityMetadata {
    oneof payload {
        string string_payload = 1;
        int64 int_payload = 2;
        string json_payload = 3;
    }
}
```

These `EntityMetadata` objects are normally stored in a `map<string, EntityMetadata>` field such as in `accounts.proto`

```protobuf
/* Bundled data that describes an account and its metadata. */
message Account {
    string id = 1; // GUID for this account
    string display_name = 2; // display name for this account
    string auth_role = 3; // The auth level of the account
    map<string,groups.EntityMetadata> metadata = 4; // Metadata associated with the account
}
```

What this means is that in almost all scenarios, `EntityMetadata` is a value associated with a specific key and any CRUD operations will be reliant on that key.

**Currently, querying for a specific metadata entry by entry value is not supported.**

There are three options for what type of value `EntityMetadata` stores:

- `int_payload`
    - int64 or long in C#
    - ******Does not****** support appending update operation
- `string_payload`
    - **Does** support appending update operation
- `json_metadata`
    - Represents a JSON object that is serialized to a string
    - ****************Does**************** support appending update operation
    - ********Does******** support partial entry deletion operation

## JSON Metadata

JSON metadata is a special metadata type that supports a more flexible / expandable approach compared to more primitive metadata types. Both partial updates (`UpdateMetadataOperationType::Append`) and partial deletion (`DeleteJsonMetadataType::PartialEntry`) are supported by JSON metadata.

## Database Storage

For Catena Accounts and Parties, metadata is stored in a backing database so that it can be retrieved later. The database model looks like:

- Key - string
    - Primary key
    - Used to uniquely identify the metadata entry
- Id - string
    - Primary key, foreign key
    - This can be party ID, player ID, or account ID
- Payload_type - int32
    - Enum that corresponds to `EntityMetadata.PayloadOneofCase` which is used when reconstructing the `EntityMetadata`
- Int_payload - int64 or long
- String_payload - string
- Json_payload - string

# CRUD Operations

## Create Metadata

```protobuf
message CreateMetadataEntryRequest {
    string id = 1; 
    string entry_key = 2; // The key in the metadata to be added
    EntityMetadata entry_value = 3; // The metadata entry value
}
```

If a `CreateMetadataEntryRequest` is processed and the `entry_key` is already present, then an `RpcException` is thrown due to a metadata entry already existing.

## Read Metadata

```protobuf
message GetMetadataEntryRequest {
    string id = 1;
    string entry_key = 2;
}

message GetMetadataEntryResponse {
    EntityMetadata entry_value = 1;
}

message GetMetadataEntriesRequest {
    string id = 1;
}

message GetMetadataEntriesResponse {
    map<string, EntityMetadata> metadata = 1;
}
```

## Update Metadata

```protobuf
// When updating a metadata entry, the entry can either be overwritted (supported by all types) or appened (supported by string and json)
enum UpdateMetadataOperationType {
    OVERWRITE = 0; // Works with all payload types
    APPEND = 1; // Only works with string and json payload
}

message UpdateMetadataEntryRequest {
    string id = 1; 
    string entry_key = 2; // The key in the metadata whose entry needs to be changed
    EntityMetadata entry_value = 3; // The metadata entry value
    UpdateMetadataOperationType update_operation_type = 4; // The update type
}
```

By default, if the `update_operation_type` is not specified in a request, it will default to `OVERWRITE`.

`OVERWRITE` will simply overwrite whatever entry is currently present, not caring about what payload case is currently there.

`APPEND` has different behavior depending on the data type that is passed in. If the payload case of `entry_value` does not equal the payload case of the already present metadata stored at `entry_key`, an `RpcException` will be thrown.

`**APPEND` behavior table:**

| Payload case | Already present metadata value | Update metadata request value | Resulting value |
| --- | --- | --- | --- |
| Int | 2 | 5 | RpcException due to attempting to append a data type that is not appendable |
| String | “testing-string” | “new-string” | “testing-stringnew-string” |
| JSON | {
    “example-key” : “example-1”,
    “example-array” : [ “example-1” ]
} | {
“example-key” : “example-2”,
“example-array” : [ “example-2” ]
} | {
“example-key” : “example-2”,
“example-array” : [ “example-1”, “example-2” ]
} |

## Delete Metadata

```protobuf
// If you are deleting a json, can choose whether to delete the entire entry or a partial part of an entry
// Currently, only whole property removal is supported.
// Removal of elements in a json array (e.g. trying to remove "value" from json {"array":["value","value2"]})
// is not currently supported by DeleteMetadataEntryRequest.
enum DeleteJsonMetadataType {
    ENTIRE_ENTRY = 0; // Delete the entire json
    PARTIAL_ENTRY = 1; // Delete a specific part of json
}

message DeleteMetadataEntryRequest {
    string id = 1;
    string entry_key = 2; // The key in the metadata whose entry needs to be deleted
    DeleteJsonMetadataType json_handling_type = 3; 
    repeated string properties_to_remove_paths = 4; // Use this to remove entire properties from a json. This value is a Path
}

message DeleteMetadataEntryResponse {
    bool entry_deleted = 1; // If the entry was found in the database and deleted
}
```

Unless attempting to delete JSON data, `json_handling_type` and `properties_to_remove_paths` can be omitted from a `DeleteMetadataEntryRequest`. By default, `json_handling_type` will be set to `ENTIRE_ENTRY`.

********************************Currently, only whole property removal is supported by JSON deletion.******************************** What this means is that values of an array in a JSON object can not be removed without removing the entire array property.

```json
{
	"example-key" : "example-value", <-- This CAN be removed
	"example-array" : [ <-- This CAN be removed
		"example-value",  <-- This CANNOT be removed by itself, instead "example-array" needs to be removed
		"example-value-2"
	],
	"example-object" : { <-- This CAN be removed
		"example-key" : "example-value", <-- This CAN be removed
		"example-array" : [ <-- This CAN be removed
			"example-value" <-- This CANNOT be removed by itself, instead "example-object.example-array" needs to be removed
		]
	}
}
```

The `entry_deleted` value in `DeleteMetadataEntryResponse` represents if a value was actually removed from the backing database. If a value of `false` is returned, that does not mean that the delete operation failed, as any serious failures would result in an `RpcException`. Instead, a `false` return value means that the deleted entry was not present in the backing database.

## Update Account / Party Player

Metadata can be added or overwritten (************************but not appended************************) using the `UpdateAccount` and `UpdatePartyPlayer` rpc. Both of the requests take in a [FieldMask](https://cloud.google.com/dotnet/docs/reference/Google.Protobuf/latest/Google.Protobuf.WellKnownTypes.FieldMask) argument. If the “metadata” path is added to the `FieldMask`, then whatever data is in the `metadata` field of the request will either be added (if there is no entry present on the Catena backend) or overwritten (if there is an entry present on the Catena backend).

# Parties

Parties are a little bit special in regards to metadata. This is due to the structure of the `Party` object and the `Player` object.

```protobuf
message Party {
	string party_id = 1;
	string leader_id = 2;
	string invite_code = 3;
	repeated Player players = 4;
	map<string,groups.EntityMetadata> metadata = 5;
}

message Player {
	string player_id = 1;
	string display_name = 2;
	bool is_ready = 3;
	bool is_leader = 4;
	int32 team_number = 5;
	map<string,groups.EntityMetadata> metadata = 6;
}
```

Basically, there are two layers of metadata associated with a party. One associated with the party itself and one associated with the individual members of each party. The CRUD requests are slightly different for parties and are formatted like this:

```protobuf
message CreatePartyMetadataEntryRequest {
	groups.CreateMetadataEntryRequest inner_request = 1;
	string player_id = 2; /* Optional: If included, query for a specific party member */
}

message UpdatePartyMetadataEntryRequest {
	groups.UpdateMetadataEntryRequest inner_request = 1;
	string player_id = 2; /* Optional: If included, query for a specific party member */
}

message DeletePartyMetadataEntryRequest {
	groups.DeleteMetadataEntryRequest inner_request = 1;
	string player_id = 2; /* Optional: If included, query for a specific party member */
}

message GetPartyMetadataEntryRequest {
	groups.GetMetadataEntryRequest inner_request = 1;
	string player_id = 2; /* Optional: If included, query for a specific party member */
}

message GetPartyMetadataEntriesRequest {
	groups.GetMetadataEntriesRequest inner_request = 1;
	string player_id = 3; /* Optional: If included, query for a specific party member */
}
```

CRUD requests that do not include the `player_id` field will be assumed to be targeting the `Party` level of metadata, not the `Player` level.

## Creating Party

```protobuf
message CreatePartyRequest {
	map<string,groups.EntityMetadata> creating_player_metadata = 1;
}
```

When creating a party, an optional metadata map can be passed in through the `creating_player_metadata` field. The metadata that is passed in will be associated with the specific `Player` that created the party.

## Joining Party

```protobuf
message JoinPartyWithInviteCodeRequest {
	string invite_code = 1; // code used to join the party
	map<string,groups.EntityMetadata> joining_player_metadata = 2;
}
```

When joining a party, an optional metadata map can be passed in through the `joining_player_metadata` field. The metadata that is passed in will be associated with the specific `Player` that joined the party.

## Leaving / Kicking from Party

When a player leaves or is kicked from a party, their associated metadata is removed from the party. No metadata at the `Party` level is removed when a player leaves or is kicked, only that individual `Player`'s metadata.

# Examples

## Associating player color to a party member

For this example, assume that players want to have a color associated with them ingame. These colors are supposed to be visible to other party members and will be passed along as part of matchmaking as well. This is a perfect opportunity to use metadata as the information is important to the game but color is not a direct member of either `Player` or `Party`. This color will be represented by three floats in an array. Each float corresponds to a value of either red, green, or blue. If this were represented in a JSON object it might look similar to:

```json
{
	"player-color" : [ 0.5, 0.5, 0.0 ]
}
```

With the first 0.5 representing red, the second 0.5 representing green, and 0.0 representing blue. When creating a party, that JSON object can be serialized and passed in as part of `creating_player_metadata`. This might look like:

```csharp
CreatePartyRequest request = new CreatePartyRequest();
EntityMetadata playerDataMetadataEntry = new EntityMetadata { JsonPayload = "{\"player-color\":[0.5,0.5,0.0]}" };
request.CreatingPlayerMetadata.Add("player-data", playerDataMetadataEntry);
// Call to Catena backend here
// If using UnitySDK...
// CatenaEntrypoint.Instance.CreateParty(request);
```

This should return a `Party` object that looks similar to this (serialized to JSON):

```json
"party": {
        "players": [
            {
                "metadata": [ <-- Metadata associated with this specific player
                    {
                        "key": "player-data",
                        "value": {
                            "json_payload": "{\"player-color\":[0.5,0.5,0.0]}"
                        }
                    }
                ],
                "player_id": "player-id-1",
                "display_name": "test02",
                "is_ready": false,
                "is_leader": true,
                "team_number": 1
            }
        ],
        "metadata": [], <-- Metadata associated with the entire party
        "party_id": "party-id-1",
        "leader_id": "player-id-1",
        "invite_code": "2ATQ4V"
    }
```

If that player’s color needs to be updated at any point, it is easy to do it with a call to `UpdateMetadataEntry`.

```csharp
EntityMetadata updatedMetadataEntry = new EntityMetadata {
	JsonPayload = "{\"player-color\":[0.2,0.0,0.8]}"
}
UpdateMetadataEntryRequest innerRequest = new UpdateMetadataEntryRequest {
	Id = "party-id-1",
  EntryKey = "player-data",
  EntryValue = updatedMetadataEntry,
  UpdateOperationType = UpdateMetadataOperationType.Overwrite // Needs to overwrite the three values currently present
}
UpdatePartyMetadataEntryRequest request = new UpdatePartyMetadataEntryRequest {
	InnerRequest = innerRequest,
	PlayerId = "player-id-1"
}
// Call to Catena backend here
// If using UnitySDK...
// CatenaEntrypoint.Instance.PartyUpdateMetadataEntry(request);
```

# Problems / Shortcomings / TODOs

- JSON metadata does not support removing entries in an array
- If updating a JSON metadata entry and an array value needs to be overwritten, the entire entry needs to overwritten as the append method will just cause the array to be expanded