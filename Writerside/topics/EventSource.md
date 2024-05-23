# EventSource

An event source provides a translation layer between your <tooltip term="game server">game server</tooltip>, the <tooltip term="backend platform">backend platform.</tooltip> and <tooltip term="fleet manager">fleet manager</tooltip>.
This way your game only needs to interface with the sidecar, and it can be deployed anywhere.

Currently, Catena supports the following event sources:

<snippet id="event-source-list">
<ul>
<li>Catena Backend Event Source - An event source which matches the <a href="Server-Manager.md">Catena Server Manager</a> API.</li>
<li>Virtual Server Event Source - A mock server for simulating calls or doing zero-code implementations</li>
</ul>
</snippet>


<chapter title="Why use a Virtual Server?" collapsible="true">

</chapter>

## Configuration

Below is an example configuration for an event source in the sidecar.

```json lines
{
  "app": {
      "event_source": {
        "use": "catena_api_event_source",
        "configs": {
          "catena_api_event_source": {
            "GrpcListenAddr": ":8080",
            "HttpListenAddr": ":8085"
          },
          "virtual_server_event_source": {
            "strategy": "continuous_strategy",
            "continuous_strategy": {
              "RequestInterval": "5s"
            }
          }
        }
      },
    }
}
```

`use` - The event source to use to ingest events into the sidecar, must match one of the configurations specified in `configs`.

`configs` - The configurations for the event sources.

[//]: # () TODO This will eventually be modified to support multiple event sources. These docs need to be updated when that changes

## Catena API Event Source

This event source will make calls to the backend on behalf of the sidecar. Their behavior is documented below, & the protobuf definitions can be found [here](https://github.com/CatenaTools/sidecar/blob/main/sidecar/proto/catena-tools-core/api/v1/catena_server_manager.proto).

<include from="MatchBroker-Snippets-Library.md" element-id="request-match-request"></include>

For request match, we will also register the server with the sidecar. If no ServerID is specified, one will be generated.

<include from="MatchBroker-Snippets-Library.md" element-id="match-ready-request"></include>

Match Ready can be sent with no Match ID. If it is omitted, the sidecar will assume one match per server and handle it automatically.

<include from="MatchBroker-Snippets-Library.md" element-id="end-match-request"></include>

Match end can be sent with no Match ID, the sidecar will manage it manually.

### Config Fields

`GrpcListenAddr` - The address to listen on for grpc connections

`HttpListenAddr` - The address to listen on for http connections

## Virtual Server Event Source

The Virtual Server event source is ideal for cases where you want to test allocations without a game server, or you don't want to 
implement any API for the sidecar.

It will generate events based on the config.

### Config

`strategy` - The strategy to use to schedule match requests

#### Continuous Strategy

Continuously requests new matches, readies them up, and then ends them.

`RequestInterval` - The interval at which to request new matches