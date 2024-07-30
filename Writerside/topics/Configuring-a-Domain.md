# Configuring a Domain

Catena requires a properly configured domain in order to run in production. This is both for simplicity and to allow us to configure routing to your services 
across all components of catena.

Depending on your infrastructure provider, you will need to use their specific instructions to setup a domain.

## AWS

The catena quickstart and other aws configurations use a Hosted Zone to create and manage DNS routes.

AWS maintains this documentation here: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html