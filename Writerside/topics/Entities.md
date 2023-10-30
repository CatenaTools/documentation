# Entities

# Entity Definition

```protobuf
enum EntityType {
    ENTITY_TYPE_UNSPECIFIED = 0;
    ENTITY_TYPE_GENERIC = 1;
    ENTITY_TYPE_ACCOUNT = 2;
    ENTITY_TYPE_GROUP = 3;
		ENTITY_TYPE_REFERENCE = 4;
		ENTITY_TYPE_ANNOTATION = 5;
}

message Entity {
    string id = 1;
    string display_name = 2;
    EntityType entity_type = 3;
    repeated Entity entities = 4;
    map<string,EntityMetadata> metadata = 5;
}

message EntityMetadata {
    oneof payload {
        string string_payload = 1;
        int64 int_payload = 2;
        google.protobuf.Any any_payload = 3;
    }
}
```

Entities can be of type Group, or another specified entity type. Usually, the entities filed of the Entity type will only be used for a group type. Any data that is needed to augment the entity can be stored in the metadata field.

# Example Usage: Accounts

### Catena Accounts

Catena accounts will have a type of `ENTITY_TYPE_ACCOUNT` and will store all data about the account within the Entity object. This means Catena accounts are very thin and usually only include a display name.

### External Accounts

If a client wishes to “Bring their own accounts system” we can support this by having a user create an `ENTITY_TYPE_REFERENCE` that can be used to reference the external account. Any data that is required to retrieve this external account can be stored in the metadata of the entity. Catena will act as a proxy in this case but the external reference can still be used for parties, matches, etc.

## Example Usage: Party Annotations

Entities can be used to annotate parties, keeping the structure of the parties system simple and functional. This does increase complexity downstream from the party system, but enables more flexible definitions.

An example entity to annotate a party by team members is:

```jsx
{
  "id": "123456",
  "display_name": "TEAM_ASSIGNMENTS",
  "entity_type": "ENTITY_TYPE_ANNOTATION",
  "entities": [],
  "metadata": {
    "team1": {
      "string_payload": "player_id_1"
    },
    "team2": {
      "string_payload": "player_id_2"
    }
  }
}
```

A client could add this to a party to specify team assignments and process it during matchmaking.

# Reasoning

The original groups proposal was too generic and ran the risk of becoming a hindrance to adoption rather than a help. This solution pares back the flexibility of the original design while still enabling us to interface with many different data types.

# Risks

- Using entities as annotations and labels will increase complexity when these data types are used down the line. For example storing team data as an annotation entity will increase the complexity of the matchmaker which must validate and then process the data.
- Clients will need to be careful not to expose additional data such as player ids or other sensitive information that could be in the entity metadata.