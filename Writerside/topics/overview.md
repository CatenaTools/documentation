# Getting Started

## Project Structure

Catena is structured as a monorepo. All services run attached to a “CatenaNode” which acts as the main health-checker, config manager and entrypoint for GRPC and HTTP requests. When you start catena with `make dev` it will start the CatenaNode with all registered services in one node.

## Instructions

1. Install Prerequisites
    1. Dotnet 7 - [https://dotnet.microsoft.com/en-us/download/dotnet/7.0](https://dotnet.microsoft.com/en-us/download/dotnet/7.0)
    2. Python3 - [https://www.python.org/downloads/](https://www.python.org/downloads/)
    3. Git
    4. Redis - [https://redis.io](https://redis.io/)
    5. (Optional) Postman - [https://www.postman.com](https://www.postman.com/)
    6. (Optional) Docker - [https://www.docker.com](https://www.docker.com/)
2. Clone the [Catena Tools](https://github.com/CatenaTools/CatenaTools) repository
3. `cd CatenaTools`
4. Start a redis server (platform dependent)
5. Run Catena Tools `make dev`
