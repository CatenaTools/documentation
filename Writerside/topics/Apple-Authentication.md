# Apple Authentication

## Developing/Testing Locally

Apple doesn’t allow their redirect URLs to be `http` or contain `[localhost](http://localhost)` which obviously has issues when working locally, the easiest way around this is use something like [Caddy](https://caddyserver.com/) and setup a reverse-proxy.

### Setting up Caddy using Docker

- **Basic Setup**

  [Caddy Documentation](https://caddyserver.com/docs/running#docker-compose)

  Using the two files below you be able to stand up a docker container running a Caddy server that will take all requests on `apple.localhost` and forward them to your host machine on port `5001`

  *At the time of writing this the port for gRPC and HTTP for the Catena launcher were still a bit up in the air, so you might have to change the port to match the HTTP port the launcher ends up using*

  **docker-compose.yml**

    ```docker
    version: "3.9"
    
    services:
      caddy:
        image: caddy:2.7
        restart: unless-stopped
        ports:
          - "443:443"
        volumes:
          - ./Caddyfile:/etc/caddy/Caddyfile
    ```

  You will need to change `volumes` to match where ever your `Caddyfile` , below, will be located on disk

  **Caddyfile**

    ```
    apple.localhost {
    	reverse_proxy * {
    		to host.docker.internal:5001
    	}
    }
    ```

- **SSL Certificate**

  Since Apple requires the the redirect URI to be `https` we’ll need a certificate for `apple.localhost`

  Luckily Caddy handles all this for us and you just need to download the certificate off the docker container and add it to your machine as a trusted certificate. The root.crt on the container is located here: `/data/caddy/pki/authorities/local/root.crt`

  Grab that file off the container and add it to your machine as a trusted certificate

- **Last Steps**

  After starting up the container and making sure you’ve added the certificate to your trust store, you should be able to sign in to your Apple ID. You will be redirected back to `[https://apple.localhost/api/v1/authentication/PROVIDER_APPLE/callback](https://apple.localhost/api/v1/authentication/PROVIDER_APPLE/callback)` where Caddy is waiting to forward you to `localhost`

  Just as a precaution keep the Caddy logs running incase something does go wrong and authentication is successful

  `docker compose logs caddy -n=1000 -f`


### Apple Auth Gotchas

- **User Info**

  The First and Last name of a user is only sent once by Apple when the user authenticates with the Catena App — This is a apparently a “security” precaution set by Apple

  If we plan to have this information moving forward, we’re going to have to consider storing it permanently.

  This information comes back in Apples form post redirect after authorizing the with our App. So if there is a network blip or our services goes down, or has some unforeseen error we will lose this information forever.

- **Client Secret**

  In order to grab a token from Apple after a successful login you need to use your Client Secret

  This is JWT signed with a private key you create in Apple Developer, and it is only good for 6 months.

  **We’re going to need to think about how we’re going to handle this expiration**

- **Their Login Button**

  The method we implemented other auth providers is that we request the redirect URI from the backend. Apple’s Sign In button ships with JavaScript that is intended to build this URI on the frontend.

  It does this with some metadata tags in HTML `head` tags, see [documentation](https://developer.apple.com/documentation/sign_in_with_apple/displaying_sign_in_with_apple_buttons_on_the_web#3333109).

  In an effort to keep things consistent with our other, already, implemented providers this is circumvented with some additional JS and the request is still made to the backend.

  It would have been nice if we could have removed those `meta` tags, but the button ultimately doesn’t render without them

  Another option here, although not fully explored, is to use their button generation API and use an svg of their button, and hopefully not need to use `meta` tags or their JS