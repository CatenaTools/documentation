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
* A Game with a client/dedicated game server model or the [Catena Lyra Demo](https://github.com/catenaTools/catena-lyra-demo)
* The Catena Tools [AWS Infrastructure Template](https://github.com/CatenaTools/infrastructure/tree/main/aws/catena-core)
* [Terraform](https://www.terraform.io) V1.71 or higher
* An AWS account
* Git bash (optional)
* A domain name that you control and can configure in [AWS Route 53](https://aws.amazon.com/route53/)

## Setup the Infrastructure

Prior to provisioning any infrastructure, you will need to set up a hosted zone in Route 53 for the domain that you will use. In the example, this will be `catenatools.com.`

See [Creating a public hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html). The Terraform configuration in the AWS Infrastructure Template will create a few subdomain routes on this URL.

Next, clone the `infrastructure` repo and open it in your favorite editor. We will be working out of the `aws/catena-core/` directory.

**Git Bash**
```bash
git clone git@github.com:CatenaTools/infrastructure.git
cd infrastructure/aws/catena-core/
```

This template sets up a working [dokku](https://dokku.com) instance on the domain you select, along with a few additional settings to simplify deploying and running the platform, including an IAM role for `aws ssm` logins and an Elastic IP.

> We set things up this way to make deploying your Catena instance as simple as running `git push`. You don't need to worry too much about all of the infrastructure if you don't want to. This tutorial will outline the pieces that are important for you to understand in order to get up and running.

### Configure Terraform

The Terraform configuration defaults to using an S3 bucket to store it's state. You may create an S3 bucket in your account with the AWS Console and update the configuration or remove the S3 configuration to store state locally.

Open `terraform_config.tf` and modify the following portion to match your S3 bucket if you have created one.

```
  backend "s3" {
      bucket  = "catena-terraform-state"
      key     = "updated-infra/terraform.tfstte"
      region  = "us-east-1"
      profile = "catena_admin"
  }
```

Be sure to replace `bucket` with your bucket's name, `region` with your region, and `profile` with your local [AWS Profile](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/configure/index.html).

> If you do not need to back up or share your Terraform state, you can remove the "backend" block and Terraform will store it's state locally.

Next, we need to set some configurations specific to your game.

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

Then add a new remote and do a git push, this will deploy the `HEAD` of the repo. The command to do so is the value of the `add_doku_remote_command` output. The push command is platform specific, use the `{powershell,unix,windows}_deploy_command` output above.

**Git Bash**
```bash
git remote add dokku dokku@dev.catenatools.com:platform ## Replace this with the "add_dokku_remote_command" output by Terraform
git push dokku HEAD:main ## Replace this with your platform's deploy command
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

The full Postman collection can be found [here](https://github.com/CatenaTools/catena-tools-core/tree/main/scripts). Set the http-host environment variable to your domain prior to use.

## Running your game in a multiplayer configuration

Next we will use the server [Sidecar](Sidecar.md) along with our backend to support a multiplayer workflow.

> The sidecar enables you to configure your game to work with one API interface, and deploy it on multiple fleet managers. It can also make requests on behalf of the game server reducing the burden to set up a dedicated game server.

We are making some assumptions and trade-offs to simplify this guide. There is an example implementation in our [catena-lyra-demo repo.](https://github.com/CatenaTools/catena-lyra-demo/tree/main)

1. The server will be "always open" and accepting players
2. There will be only one type of match

### Setup our Game Server

We will be running our game on an AWS EC2 Instance running Windows Server, these instructions also work with a Linux server using minor modifications.

<note>
If you don't have an ssh key, you will need to generate one.

<code-block lang="bash">
ssh-keygen -t rsa -b 2048 -m PEM -f catena_deploy_key

</code-block>

This will generate two files
* `catena_deploy_key`
* `catena_deploy_key.pub`

Reference the files in the  `vars.tfvars` file below.
</note>

Next, configure terraform to provision an ec2 instance where we will run the server. we have provided a template for this in the `aws/ec2-gameserver` directory of the infrastructure repo.

Similar to the previous infrastructure, we must set some variables. Create another `vars.tfvar` file with the following contents:

```hcl
aws_profile="catena_admin"
aws_region="us-east-1"
game_server_os="windows"                               # Defaults to linux
ec2_instance_size="t2.medium"
ssh_private_key_path="~/.ssh/catena_deploy_key"        # Your private key
ssh_public_key_path="~/.ssh/catena_deploy_key.pub"     # Your public key
```

Next plan and apply your infrastructure

```
terraform plan -var-file="vars.tfvar" 
terraform apply -var-file="vars.tfvar"
```

Once Terraform finishes, you can go to the AWS EC2 Console, click your instance, and click "Connect to instance." Under the RDP tab, download the
Remote Desktop File. 

You can retrieve the password for the administrator account by running: `terraform output -raw ec2_windows_password`.

At this point you can connect to the Windows server using RDP by opening the file and logging into the machine with the "Administrator" account and password
from the AWS console. Subsequently, you should download and prepare to run your game on the server. Instructions for the Lyra Demo are below.

<procedure title="Lyra Game Server Installation" id="lyra_setup" collapsible="true">
    <p>These instructions use the <a href="https://github.com/catenatools/catena-lyra-demo">catena-lyra-demo</a> server.</p>
    <step>Download the server onto the machine. The download can be found <a href="https://catena-public-content.S3.amazonaws.com/WindowsServer.zip">here.</a></step>
    <step>Extract the zip file.</step>
    <step>Run the game server from command prompt: <pre>LyraServer.exe -networkversionoverride=1234 -BackendUrl=localhost:8085</pre></step>
    <step>Now the game server will communicate with the backend through the sidecar.</step>
    
> To copy your game server to the remote server, either copy and paste through remote desktop if you are using windows, or use `scp` if you are using a linux machine. `scp -i ~/.ssh/catena_deploy_key gameserver.zip user@remote_host:/remote/directory/`
</procedure>

## Run the Server Sidecar

The server sidecar enables Catena to be aware of your game server without requiring any changes to the code of the game server itself. Additionally, it is able to make requests on behalf of your game server
if you so choose. The sidecar will augment connection details and can translate requests between different backends so your server only needs to implement one SDK interface. See [Sidecar](Sidecar.md) for more details.

First download the Sidecar to the machine you are running the game server on. The latest release can be found [here](https://github.com/CatenaTools/sidecar/releases/). Be sure to download the version for your platform. If you are following this tutorial, use the Windows x64 Version. 

Alternatively, if you so choose, you can clone and build the sidecar.

In the same directory where the sidecar compiled binary is located, add a `config.json.` There is an example in the [sidecar repo](https://github.com/CatenaTools/sidecar/blob/main/config.json).

```json
{
  "motd": "Hello Catena",
  "app": {
    "logging": {
      "level": "debug",
      "encoder": "console"
    },
    "event_source": {
      "use": "catena_api_event_source",
      "configs": {
        "catena_api_event_source": {
          "GrpcListenAddr": "0.0.0.0:8080",
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
          "url": "$CATENA_URL"
        },
        "noop_backend": {}
      },
      "use": "catena_backend"
    },
    "sidecar": {
      "DebugPanelAddr": "localhost:8081",
      "MatchManagementStrategy": {
        "OverrideConnectionDetails": true,
        "AutoPopulateIDs": true
      }
    }
  }
}
```

The key is to set the following fields:

`app.backend.catena_backend.url` The url of your deployed backend.
`app.resolver.configs.ec2_resolver.Min` - The min port to use for clients to connect.
`app.resolver.configs.ec2_resolver.Max` - The max port to use for clients to connect.

_For this demo, min and max port should be the same, and be the port your server runs on. If you are following this demo, it should be `7777`_

> Also set is the `app.event_source.use` field, which is `catena_api_event_source.` If your server implements the [Catena Server Manager](https://github.com/CatenaTools/catena-tools-core/blob/main/Protos/api/v1/catena_server_manager.proto) interface
> it can make requests to this backend.

After which you can launch the sidecar using `cmd.exe`

<procedure collapsible="true" title="Run the sidecar">
<step>
Press <shortcut>Ctrl+R</shortcut> or Start -> Run
</step>
<step>Type cmd.exe and click run</step>
<step>Change to the directory you are running the sidecar from <code>cd C:\Path\To\sidecar-windows-amd64.exe</code></step>
<step>Launch the sidecar <code>sidecar-windows-amd64.exe</code></step>
</procedure>

At this point you should see the sidecar start up. It will resolve connection details and make requests on behalf of the game server. Our lyra demo is setup to request matches through the sidecar.

> If you don't want to interface with the sidecar or want the quickest path to a working setup, the sidecar can pretend to be a server and request matches on behalf of your server. To do this, simply change `app.event_source.use` to `virtual_server_event_source`. This tells the sidecar to act as a server and request new matches
> every 5 seconds. So long as your server is listening on port 7777 clients will be able to connect to your server.
> 
> If you restart the sidecar with this config, you will see it requesting matches repeatedly.
> 
> See [Match Broker](Match-Broker.md) to read about how Catena manages servers.


## Testing it out

Next we will download the sample game and test it out. In this case we will use a prebuilt version of the Catena Lyra Demo. It can be found [here](https://catena-public-content.s3.amazonaws.com/WindowsClient.zip).

So long as the server is still running from the last example, we simply need to launch the game with a few arguments, and we're good to go.

Grab a logged-in session token. Catena has a multi step login process, since not all operations will require a full account.

<procedure title="Log into Catena">
<step>
    Grab a session token.
<code-block lang="bash">
SESSION_ID_1=$(curl -D - -sS --location '$CATENA_URL/api/v1/authentication/login' \
    --header 'Content-Type: application/json' \
    --data '{
    "provider": "PROVIDER_UNSAFE",
    "payload": "test01"
}' | grep "session-*" | cut -d' ' -f2)
    </code-block>
</step>
<step>
Create an account for that session.

<code-block lang="bash">
curl --location '$CATENA_URL/api/v1/accounts' \
    --header "session-id: $SESSION_ID_1" \
    --header 'Content-Type: application/json' \
    --data '{}'
</code-block>
</step>
<step>View the new session token.
<code-block lang="bash">
echo $SESSION_ID_1
# session-ca8fb665-69df-462f-941e-3341275fd91c
</code-block>
</step>
</procedure>

Launch the game

```
LyraGame.exe -BackendURL $CATENA_URL -networkversionoverride=1234
```

> By default, catena will log in with a test account, if you wish to login with another account a login token can be provided on the command line by passing a `-session-id $SESSION_ID` argument.

From here, you can navigate through Play Lyra -> Matchmaking -> Elimination. You should see your game enter matchmaking.

If you enter another client into the matchmaking queue (or enter one via curl or postman) you will see your game and the other client connect to the same server and the game should start!

### What calls are we making?

In order to more deeply understand the workflow, We can examine the workflow by using two command line clients.

Firstly, lets setup our sidecar to constantly request new matches. If you followed the instructions above, the sidecar will not request a match unless the game server requests, one, lets set it up to
request matches in a loop instead.

Terminate the sidecar on the Game Server Using <shortcut>Ctrl+C</shortcut>.

Next, modify `config.json` to use the `virtual_server_event_source` by changing the "use" field of the `event_source` config.


```
// -- Truncated
    "event_source": {
      "use": "virtual_server_event_source",
      "configs": {
        "catena_api_event_source": {
          "GrpcListenAddr": "0.0.0.0:8080",
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
// -- Truncated
```

Restart the sidecar. You should see it repeatedly requesting new matches.

You can also see the sidecar's virtual server in the list by listing servers from the API:

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

Next, lets test out matchmaking!

First, Login and Create two Accounts. This may be easier in postman, but curl works too. Be sure to do it in two terminal windows, or ensure you are applying session ids consistently.

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
SESSION_ID_2=$(curl -D - -sS --location '$CATENA_URL/api/v1/authentication/login' \
      --header 'Content-Type: application/json' \
      --data '{
      "provider": "PROVIDER_UNSAFE",
      "payload": "test01"
  }' | grep "session-*" | cut -d' ' -f2)

curl --location '$CATENA_URL/api/v1/accounts' \
--header "session-id: $SESSION_ID_2" \
--header 'Content-Type: application/json' \
--data '{}'
```

Take note of the two account ids which will be returned by the second curl command.

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
curl --location --request GET '$CATENA_URL/api/v1/events/subscribe' \
--header "session-id: $SESSION_ID_1" \
--header 'Content-Type: application/json' \
--data '{}'

{"event":{"eventId":"event-0813d3d5-7033-4714-bddf-5a4e481985c4","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","emptyEvent":{}}}
{"event":{"eventId":"event-7faf0362-70ae-453b-b3ac-8656c0fb6888","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_FINDING_SERVER","message":"Finding server for ticket 0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1","ticketId":"0b4a0ca2-7da6-4d5c-ae9d-e3340f5487c1"}}}
{"event":{"eventId":"event-d25eb479-257e-462a-b51f-4f983b65e03d","recipientId":"account-f378e3c2-8760-4059-ba91-69f5343dfb48","matchmakingStatusUpdateEvent":{"type":"MATCHMAKING_STATUS_UPDATE_TYPE_COMPLETED","message":"Server found and matchmaking completed for match match-c2420756-a298-4066-8ee2-7f5fb9891900","ticketId":"","serverConnectionInfo":{"ip":"34.228.12.6:7777","port":0,"serverId":""}}}}
```

### Wrap it 

This tutorial covered several concepts in Catena's Ecosystem. To read about them in more detail use the links below:

* [Matchmaking](Matchmaking.md)
* [Match Broker](Match-Broker.md)
* [Sidecar](Sidecar.md)
* [Accounts](Accounts.md)