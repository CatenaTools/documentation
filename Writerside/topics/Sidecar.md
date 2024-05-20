# Sidecar

The Catena Sidecar is a gateway to enable your <tooltip term="game server">game server</tooltip> to run on a litany of different fleet management solutions, and to interface with different 
backends. You are only required to implement one interface, after which you can deploy your game anywhere.

The Sidecar is made up of a few core components:

* [](Resolver.md) - Determines the publicly accessible address of the game server.
* [](Backend.md) - Interfaces with your game backend, like Catena, Accelbyte or Pragma.
* [](EventSource.md) - Receives events from your game server to report ot the backend.

Through a combination of these components, the sidecar can translate your servers requests to run on any environment.

> The following tutorials utilize the Sidecar:
> <include from="Sidecar-Snippets-Library.md" element-id="list-of-sidecar-tutorials"/>

## Configuring the Sidecar

The sidecar uses a json file `config.json` which sits next to the sidecar bin``ary.