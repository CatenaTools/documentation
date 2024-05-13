# Quickstart: Running Your Game With Catena

In this quickstart guide we will walk through how to set up and run your game on your own infrastructure using Catena Tools. 
Through this guide you will set up a backend, basic fleet manager, along with a game client and game server. This is a __Static Fleet Management Solution.__ 
This means you will have to manually provision servers, but they will report themselves to the backend.

> This tutorial will use the following Catena Components to support a Multiplayer Workflow.
>   * [Match Broker](Match-Broker.md)
>   * [Server Manager](Server-Manager.md)
>   * [Sidecar](Sidecar.md)
>   * [Matchmaking](Matchmaking.md)
>   * Infrastructure Templates

## Overview

By the end of this quickstart, you will be able to run your game, or a sample game in a multiplayer/online environment using AWS. Players will be able to:
1. Log into your game
2. Form a Party
3. Request a Match
4. Connect to a Server
5. Play a game

## Prerequisites

What you will need before following the steps in this guide.

* Access to Catena Tools
* A Game with a client server model or the [Catena Lyra Demo](https://github.com/catenaTools/catena-lyra-demo)
* The Catena Tools [AWS Infrastructure Template](https://github.com/CatenaTools/infrastructure/tree/main/aws/catena-core)
* [Terraform](https://www.terraform.io) V1.71 or higher
* An AWS account
* Git bash (optional)
* A domain name that you control and can configure in route53

## Setup the Infrastructure

Prior to running the Terraform code, set up a hosted zone for the domain that you will use. In the example, this will be `catenatools.com.`

See [Creating a public hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html). The Terraform configuration will create a few subdomain routes on this url.

Next, clone the `infrastructure` repo and open it in your favorite editor. We will be working out of the `aws/catena-core/` directory.

**Git Bash**
```bash
git clone git@github.com:CatenaTools/infrastructure.git
cd infrastructure/aws
```

This template sets up a working [dokku](https://dokku.com) instance on the domain you select, along with a few additional settings to simplify deploying and running the platform, including an IAM role for `aws ssm` logins and an Elastic IP.

### Configure Terraform

The Terraform configuration defaults to using an s3 bucket to store it's state, this must be updated to an s3 bucket on your account. You may create this s3 bucket in the AWS Console.

Open `terraform_config.tf` and modify the following portion to match your s3 bucket, or remove it to store state locally.

```
  backend "s3" {
      bucket  = "catena-terraform-state"
      key     = "updated-infra/terraform.tfstate"
      region  = "us-east-1"
      profile = "catena_admin"
  }
```

Be sure to replace `bucket` with your bucket's name, `region` with your region, and `profile` with your local [aws profile](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/configure/index.html).

> If you do not need to back up or share your terraform state, you can remove the "backend" block and terraform will store it's state locally.

Next, we need to set some specific-to-you configurations.

Create a `vars.tfvars` file similar to the following:

```
aws_profile="catena_admin"                  # Your AWS Profile
aws_region="us-east-1"                      # The region you are working in
ec2_ami="ami-0c7217cdde317cfec"             # AWS Ubuntu 22.04 LTS AMI for your region
ec2_instance_size="t2.small"                # Instance Size for the machine running the catena backend
ssh_private_key_path="~/.ssh/id_rsa"        # Your private key
ssh_public_key_path="~/.ssh/id_rsa.pub"     # Your public key
route_53_hosted_zone_name="catenatools.com" # The name of the hosted zone in route53
letsencrypt_email="example@example.com"     # The email address LetsEncrypt will send cert expiry notifications to
```

### Provision the Infrastructure

```bash
terraform init                        # Initialize the terraform backend and install modules
terraform workspace switch -or-create dev
terraform plan -var-file="vars.tfvar" # Preview the steps terraform will take
terraform apply -var-file="vars.tfvar"
```

After the apply step runs, you should see some output like this:

```
add_dokku_remote_command = "git remote add dokku dokku@dev.catenatools.com:platform"
ec2_instance_id = "i-0f47e887113bf882b"
ec2_instance_ssh_command = "aws ec2-instance-connect ssh --os-user ubuntu --instance-id i-0f47e887113bf882b --profile catena"
ec2_ip = "100.28.26.117"
powershell_deploy_command = "$env:GIT_SSH_COMMAND='ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes'; git push dokku main"
unix_deploy_command = "GIT_SSH_COMMAND='ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes' git push dokku main"
windows_deploy_command = "set \"GIT_SSH_COMMAND=ssh -i ~/.ssh/id_rsa -o IdentitiesOnly=yes\" && git push dokku main"
catena_url = "https://platform.dev.catenatools.com"
```

These outputs will be used to deploy the platform to your newly configured instance.

> The rest of this tutorial will use the environment variable `$CATENA_URL`. Set it to the value of the catena_url output.
> ```bash
> export CATENA_URL="https://platform.dev.catenatools.com"
> ```

### Install Catena

Installing Catena is as easy as doing a git push.

Start by cloning the `catena-tools-core` repo:

**Git Bash**
```bash
git clone git@github.com:CatenaTools/catena-tools-core.git
cd catena-tools-core/
```

Then add a new remote and do a git push, this will deploy the `HEAD` of the repo. The command to do so is the value of the `add_doku_remote_command` output.

**Git Bash**
```bash
git remote add dokku dokku@dev.catenatools.com:platform
git push dokku HEAD:main
```

Dokku will then show deploy logs:

```
remote: Resolving deltas: 100% (5066/5066), done.
-----> Set main as deploy-branch
-----> Cleaning up...
-----> Building platform from Dockerfile
remote: #0 building with "default" instance using docker driver
remote:
remote: #1 [internal] load build definition from Dockerfile
remote: #1 transferring dockerfile: 1.50kB done
remote: #1 DONE 0.1s
remote:
remote: #2 [internal] load metadata for mcr.microsoft.com/dotnet/sdk:8.0
remote: #2 DONE 0.2s
remote:
remote: #3 [internal] load metadata for docker.io/library/debian:latest
remote: #3 DONE 0.4s
remote:
remote: #4 [internal] load .dockerignore
remote: #4 transferring context: 51B done
remote: #4 DONE 0.0s

### Truncated for Brevity

=====> Application deployed:
       http://platform.dev.catenatools.com
       https://platform.dev.catenatools.com

To jlyon1.catenatools.com:platform
 * [new branch]      HEAD -> main
```

Now we're all set! 

> You can update your platform the same way, simply commit the changes and push them to the `main` branch on the dokku remote. The server will automatically build and deploy your changes with no downtime.


You can test your platform by logging in:

**Git Bash**
```Bash
SESSION_ID=$(curl -D - -sS --location '$CATENA_URL/api/v1/authentication/login' \
      --header 'Content-Type: application/json' \
      --data '{
      "provider": "PROVIDER_UNSAFE",
      "payload": "test53"
  }' | grep "session-*" | cut -d' ' -f2)

curl --location '$CATENA_URL/api/v1/accounts' \
--header "session-id: $SESSION_ID" \
--header 'Content-Type: application/json' \
--data '{}'

# {"account":{"id":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","displayName":"test53","authRole":"user","metadata":{},"providers":["PROVIDER_UNSAFE"]}}
```

[//]: # (TODO: Add the link here.)
The full postman collection can be found here. Set the http-host environment variable to your domain prior to use.

## Running your game in a multiplayer configuration

Next we will use the server [Sidecar](Sidecar.md) along with our backend to support a multiplayer workflow.

> The sidecar enables you to configure your game to work with one API interface, and deploy it on multiple fleet managers. It can also make requests on behalf of the game server, enabling integrations which require zero server-side integration.

We are making some assumptions and trade-offs to simplify this guide. There is an example implementation in our [catena-lyra-demo repo.](https://github.com/CatenaTools/catena-lyra-demo/tree/main)

1. The server will be "always open" and accepting players
2. There will be only one type of match

### Setup our Game Server

We will be running our game on an AWS EC2 Instance running Windows Server, these instructions also work with a Linux server using minor modifications.

First create a key pair in the AWS console for remote access to the machine:

![access_key_quickstart](access_key_quickstart.png)

Next, configure terraform to provision an ec2 instance where we will run the server, this should be in `ec2/main.tf`

```
resource aws_instance "server_instance" {
    ami = "ami-0f496107db66676ff"
    instance_type = "t2.small"
    key_name = "windows-server"

    root_block_device {
        volume_size = 50
    }

    vpc_security_group_ids = [var.security_group_id]
    subnet_id = var.subnets[0].id

    tags = {
        Name = "${var.workspace_prepend}Windows Server"
    
    }
}
```

Additionally, we should add additional exposed ports to the security group:

> For the purpose of this tutorial, we can add these to the existing security group, but in production it is a better choice to have 
> separate security groups for your game servers and to not expose RDP

```
    ingress {
        description = "Game Server Traffic"
        from_port   = 7777
        to_port     = 7777
        protocol    = "udp"
        cidr_blocks = ["0.0.0.0/0"] 
    }

    ingress {
        description = "RDP Traffic"
        from_port   = 3389
        to_port     = 3389
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    ingress {
        description = "RDP UDP Traffic"
        from_port   = 3389
        to_port     = 3389
        protocol    = "udp"
        cidr_blocks = ["0.0.0.0/0"]
    }

```

Next plan and apply your infrastructure

```
terraform plan -var-file="vars.tfvar" 
terraform apply -var-file="vars.tfvar"
```

Once terraform finishes, you can go to the AWS EC2 Console, click your instance, and click "Connect to instance." Under the RDP tab, download the
Remote Desktop File. Then click "Get Password" and paste in the private key from the keypair you created previously.

At this point you can connect to the Windows server using RDP by opening the file and logging into the machine with the "Administrator" account and password
from the AWS console.

At this point you should download and prepare to run your game on the server. Instructions for the Lyra Demo are below.

<procedure title="Lyra Game Server Installation" id="lyra_setup" collapsible="true">
    <p>These instructions use the <a href="https://github.com/catenatools/catena-lyra-demo">catena-lyra-demo</a> server.</p>
    <step>Download the server onto the machine. The download can be found <a href="https://catena-public-content.s3.amazonaws.com/WindowsServer.zip">here.</a></step>
    <step>Extract the zip file.</step>
    <step>Run the game server from command prompt: <pre>LyraServer.exe -networkversionoverride=1234</pre></step>
</procedure>

## Run the Server Sidecar

The server sidecar enables Catena to be aware of your game server without requiring any changes to the code of the game server itself. It is additionally able to make requests on behalf of your game server
if you so choose. See [Sidecar](Sidecar.md).

First download the Sidecar to the machine, the latest release can be found [here](https://github.com/CatenaTools/sidecar/releases/). Alternatively, clone and build the sidecar.

In the same directory as the sidecar, add a `config.json.` There is an example in the  sidecar repo here: https://github.com/CatenaTools/sidecar/blob/main/config.json

```json
{
  "motd": "Hello Catena",
  "app": {
    "logging": {
      "level": "debug",
      "encoder": "console"
    },
    "event_source": {
      "use": "virtual_server_event_source",
      "configs": {
        "catena_api_event_source": {
          "GrpcListenAddr": "0.0.0.0:8080"
        },
        "virtual_server_event_source": {
          "strategy": "continuous_strategy",
          "continuous_strategy": {
            "RequestInterval": "5s"
          }
        }
      }
    },
    "resolver": {
      "use": "ec2_resolver",
      "configs": {
        "ec2_resolver": {
          "MetadataBaseUrl": "http://169.254.169.254",
          "AvailablePorts": [
            {
              "Min": 7777,
              "Max": 7777
            }
          ]
        },
        "static_resolver": {
          "Address": "server.local:8080"
        }
      }
    },
    "backend": {
      "configs": {
        "catena_backend": {
          "url": "https://platform.dev.catenatools.com"
        },
        "noop_backend": {}
      },
      "use": "catena_backend"
    },
    "sidecar": {
      "DebugPanelAddr": "localhost:8081",
      "MatchManagementStrategy": {
        "AutoPopulateIDs": true
      }
    }
  }
}
```

The key is to set the following fields:

`app.backend.catena_backend.url` field, which must point to your deployed instance url.
`app.resolver.configs.ec2_resolver.Min` - The min port to use for clients to connect.
`app.resolver.configs.ec2_resolver.Max` - The max port to use for clients to connect.

_For this demo, min and max port should be the same, and be the port your server runs on. If you are following this demo, it should be `7777`

After which you can launch the sidecar using `cmd.exe`

```
sidecar-windows-amd64.exe
```

At this point you should see the sidecar querying for matches repeatedly. that's it, Catena is aware of your server, and is able to assign matches. So long as your server is running on the same machine as the sidecar you 
can now enter a match.

> See [Match Broker](Match-Broker.md) to read about how Catena manages servers.

You can see the new server in the list by listing servers from the API:

```cURL
curl --location '$CATENA_URL/api/v1/servers' \
--header 'Content-Type: application/json' \
--data '{}'
```

```json
{
    "servers": [
        {
            "serverId": "server-a622324b-fdce-4896-8229-16a9edcc1d7c",
            "requestedMatch": true,
            "matches": []
        }
    ]
}
```

## Testing it out

[//]: # (TODO)
Next we will download the sample game and test it out. In this case we will use a prebuilt version of the catena lyra demo. It can be found [here]().

So long as the server is still running from the last example`, we simply need to launch the game with a few arguments and we're good to go.

Grab a logged-in session token. (Run these in git bash)

```
SESSION_ID_1=$(curl -D - -sS --location '$CATENA_URL/api/v1/authentication/login' \
--header 'Content-Type: application/json' \
--data '{
"provider": "PROVIDER_UNSAFE",
"payload": "test01"
}' | grep "session-*" | cut -d' ' -f2)

curl --location '$CATENA_URL/api/v1/accounts' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{}'

echo $SESSION_ID

# session-ca8fb665-69df-462f-941e-3341275fd91c
```

Launch the game

```
LyraGame.exe -BackendURL $CATENA_URL -session-id $SESSION_ID -networkversionoverride=1234
```



Let's take a look at how this works and what the client interactions look like.

### Client Calls

First, Login and Create two Accounts. This may be easier in postman, but curl works too!

```cURL
SESSION_ID_1=$(curl -D - -sS --location '$CATENA_URL/api/v1/authentication/login' \
      --header 'Content-Type: application/json' \
      --data '{
      "provider": "PROVIDER_UNSAFE",
      "payload": "test53"
  }' | grep "session-*" | cut -d' ' -f2)

curl --location '$CATENA_URL/api/v1/accounts' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{}'
```

```cURL
SESSION_ID_2=$(curl -D - -sS --location '$CATENA_URL.catenatools.com/api/v1/authentication/login' \
      --header 'Content-Type: application/json' \
      --data '{
      "provider": "PROVIDER_UNSAFE",
      "payload": "test01"
  }' | grep "session-*" | cut -d' ' -f2)

curl --location '$CATENA_URL.catenatools.com/api/v1/accounts' \
--header "session-id: $SESSION_ID_2" \
--header 'Content-Type: application/json' \
--data '{}'
```

Take note of the two account ids.

Next, subscribe to events for one account, this is how we will get matchmaking notifications:

```cURL
curl --location --request GET '$CATENA_URL/api/v1/events/subscribe' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{}'
```

Finally, kick off matchmaking for the two connected accounts. Make sure to substitute your account ids.

```cURL
curl --location '$CATENA_URL/api/v1/matchmaking/start' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{
    "entity": {
        "id": "account-FILL_ACCOUNT_ID_HERE",
        "entity_type": "ENTITY_TYPE_ACCOUNT",
        "entities": [
        ],
        "metadata": {
            "queue_name": {
                "string_payload": "1v1"
            }
        }
    }
}'
```

```cURL
curl --location '$CATENA_URL/api/v1/matchmaking/start' \
--header "session-id: $SESSION_ID_2" \
--header 'Content-Type: application/json' \
--data '{
    "entity": {
        "id": "account-FILL_ACCOUNT_ID_HERE",
        "entity_type": "ENTITY_TYPE_ACCOUNT",
        "entities": [
        ],
        "metadata": {
            "queue_name": {
                "string_payload": "1v1"
            }
        }
    }
}'
```

If all goes well you should see your players found a match and were assigned to a server!

```
joeylyon@Joeys-MacBook-Pro catena-tools-core % curl --location --request GET '$CATENA_URL/api/v1/events/subscribe' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{}'
{"event":{"eventId":"event-0813d3d5-7033-4714-bddf-5a4e481985c4","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","emptyEvent":{}}}
{"event":{"eventId":"event-a6c101ec-e0b2-4707-a3c6-92edef38921d","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_IN_PROGRESS","message":"Ticket 0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1 started matchmaking","ticketId":"0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1"}}}
{"event":{"eventId":"event-c01c1210-c05e-497c-8539-560d393c3030","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_FINDING_SERVER","message":"Finding server for ticket ca4a506a-b279-4abb-b971-794864d99354","ticketId":"ca4a506a-b279-4abb-b971-794864d99354"}}}
{"event":{"eventId":"event-7faf0362-70ae-453b-b3ac-8656c0fb6888","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_FINDING_SERVER","message":"Finding server for ticket 0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1","ticketId":"0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1"}}}
{"event":{"eventId":"event-d25eb479-257e-462a-b51f-4f983b65e03d","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_COMPLETED","message":"Server found and matchmaking completed for match match-c2420756-a298-4066-8ee2-7f5fb9891900","ticketId":"","serverConnectionInfo":{"ip":"34.228.12.6:7777","port":0,"serverId":""}}}}
{"event":{"eventId":"event-d25eb479-257e-462a-b51f-4f983b65e03d","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_COMPLETED","message":"Server found and matchmaking completed for match match-c2420756-a298-4066-8ee2-7f5fb9891900","ticketId":"","serverConnectionInfo":{"ip":"34.228.12.6:7777","port":0,"serverId":""}}}}
```
