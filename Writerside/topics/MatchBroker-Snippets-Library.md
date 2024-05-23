# MatchBroker-Snippets-Library

{is-library="true"}

<snippet id="match-broker-match">
<code-block lang="plain text">
message Match {
    string match_id = 1;
    repeated string player_ids = 2;
    // TODO: other required match properties and custom data passing fields
    // A JSON object that contains all the custom match properties
    // May be empty
    string match_properties = 3;
    // A JSON object per-player where the map key is a player_id in the player_ids field
    // May contain values for none, some, or all players
    map (string, string) player_properties = 4;
}
</code-block>

[//]: # () Note, the map<string,string> doesn't play nice with writerside's code parser right now, so it will always give an error. use parens instead of <>
</snippet>


[//]: # (Temporarily specify these manually, until we are able to generate an openapi spec consistently)
<snippet id="request-match-request">
    <api-doc openapi-path="../apispec/openapi/api/v1/catena_server_manager.swagger.json">
        <api-endpoint endpoint="/api/v1/server/request_match" method="POST" />
    </api-doc>
</snippet>
<snippet id="match-ready-request">
    <api-doc openapi-path="../apispec/openapi/api/v1/catena_server_manager.swagger.json">
        <api-endpoint endpoint="/api/v1/server/match_ready" method="POST" />
    </api-doc>
</snippet>

<snippet id="end-match-request">
    <api-doc openapi-path="../apispec/openapi/api/v1/catena_server_manager.swagger.json">
        <api-endpoint endpoint="/api/v1/server/end_match" method="POST" />
    </api-doc>
</snippet>