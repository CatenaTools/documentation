# Getting Started

## What is Catena/Who should use Catena?

An all-in-one game platform as a service intended to help game devs take their multiplayer game to the next level with live services. Including Parties, Matchmaking, Accounts and Authentication. We integrate industry best practices out of the box but allow maximum flexibility for game developers.

## Project Structure

Catena is structured as a monorepo. All services run attached to a “CatenaNode” which acts as the main health-checker, config manager and entrypoint for GRPC and HTTP requests. When you start catena with `make dev` it will start the CatenaNode with all registered services in one node.

## Running the Platform

1. Install Prerequisites
   1. Dotnet 7 - [https://dotnet.microsoft.com/en-us/download/dotnet/7.0](https://dotnet.microsoft.com/en-us/download/dotnet/7.0)
   2. Python3 - [https://www.python.org/downloads/](https://www.python.org/downloads/)
   3. Git
   4. Redis - [https://redis.io](https://redis.io/)
   5. (Optional) Postman - [https://www.postman.com](https://www.postman.com/)
   6. (Optional) Docker - [https://www.docker.com](https://www.docker.com/)
   7. (Optional) Visual Studio

   {type="alpha-lower"}
2. Agree to the [Terms Of Service](https://tos.catenatools.com)
   * You will be invited to the CatenaTools Repo and the Catena Networking Demo Repo
3. Clone the [Catena Tools](https://github.com/CatenaTools/CatenaTools) repository
4. `cd CatenaTools`
5. Start a redis server (platform dependent)
6. Run Catena Tools `make dev`

## Running the Networking Demo

This demo is designed to show you the basics of Catena and The Catena Unity SDK.

1. Install Unity https://unity.com/download
2. Agree to the [Terms Of Service](https://tos.catenatools.com)
3. Clone the [Catena Networking Demo](https://github.com/CatenaTools/catena-networking-demo) repo
4. Run the platform (see above)
5. Open the networking demo in unity and run it

## Using the Launcher

The Catena Launcher is a Flask App, eventually this will be packaged into a usable Desktop app. In the meantime the launcher can be used to demo what an in-game flow would look like.

1. Follow the steps above to clone the launcher    
2. Install python3
3. Configure the launcher in `Launcher/data/config.yaml`
4. Run `make launcher`

## Other useful info