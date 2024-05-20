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
    <deflist collapsible="true">
        <def title="RequestMatch" default-state="expanded">
            Request a new Match from the Backend.
            <deflist collapsible="true">
            <def title="RequestMatchRequest">
                <p><b>serverId <shortcut>string</shortcut></b> - The self-reported ID of the server.</p>
                <p><b>requirements <shortcut>string</shortcut></b> - The requirements of the match to be passed to the backend (eg gamemode)</p>
            </def>
            <def title="RequestMatchResponse">  
                <p><b>match <shortcut >matchbroker.Match</shortcut></b> - The match data from the backend.</p>
                <include from="MatchBroker-Snippets-Library.md" element-id="match-broker-match"></include>
            </def>
            </deflist>
        </def>
    </deflist>
</snippet>

<snippet id="match-ready-request">
    <deflist collapsible="true">
        <def title="MatchReady" default-state="expanded">
            Signal to the backend that a match is ready
            <deflist collapsible="true">
            <def title="MatchReadyRequest">
                <p><b>serverId <shortcut>string</shortcut></b> - The self-reported ID of the server.</p>
                <p><b>matchId <shortcut>string</shortcut></b> - The ID of the match to start</p>
                <p><b>connectionDetails <shortcut>string</shortcut></b> - The connection details of the server. If using the sidecar, these do not need to be provided</p>
            </def>
            <def title="MatchReadyResponse">  
                <p><b>matchId <shortcut >string</shortcut></b> - The match ID that was started.</p>
            </def>
            </deflist>
        </def>
    </deflist>
</snippet>

<snippet id="end-match-request">
    <deflist collapsible="true">
        <def title="EndMatchRequest" default-state="expanded">
            Signal to the backend that a match has ended
            <deflist collapsible="true">
            <def title="MatchReadyRequest">
                <p><b>serverId <shortcut>string</shortcut></b> - The self-reported ID of the server.</p>
                <p><b>matchId <shortcut>string</shortcut></b> - The ID of the match to end</p>
            </def>
            <def title="MatchReadyResponse">  
                <p><b>matchId <shortcut >string</shortcut></b> - The match ID that was started.</p>
            </def>
            </deflist>
        </def>
    </deflist>
</snippet>