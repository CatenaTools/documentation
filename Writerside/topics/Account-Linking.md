# Account Linking

<include from="AuthenticationLibrary.topic" element-id="requires_user"/>

Catena's Account Linking system allows users to log into one account with multiple login providers. For example a user may wish
to log into their account with either their Playstation account or Xbox account depending on the platform they are on. The data from both providers 
post-linking will be the same. Related to this is [Account Merging](Account-Merging.md) which happens when a user has two accounts with different providers that they wish to combine.

The Account Linking flow Builds Upon the [Authentication](Authentication.md) flow. At a high level a user must first retrieve a <tooltip term="Provider Login Payload">Provider Login Payload</tooltip>
for both a provider associated with the account they want to add a new provider to and the provider they want to link in order to make a `LinkAccountToProviderRequest.` Prior to making this request the user must be 
logged in with the provider associated with the account.

## Account Linking Flow

First, the user must log in to the platform. This is done using the same login flow used everywhere else in the platform. It consists the following steps:

<include from="AuthenticationLibrary.topic" element-id="login-with-twitch"></include>

The LoginWithProviderRequest above will set a `session-id` header in the response. This header will contain a <tooltip term="session_token">Session Token.</tooltip> 
Once the login is complete, this token will refer to the User's Account, Login Payload, and other data relevant to a user's login state.

After the user has logged in, they will then need to perform a LoginWithProviderRequest for the provider they wish to link into the original account. This will create a second session containing the login
provider data for the second provider. This will be used in the LinkAccountToProvider Request.

[//]: # (Writerside requires an openapi spec, but we are definining the full spec here, so pass in a NONE tag to force it to render just what we want)
<api-doc openapi-path="../apispec/catenaapi.json" tag="NONE">
    <api-endpoint endpoint="/api/v1/accounts/link" method="POST">
        <request>
            <sample lang="javascript" title="payload">
                fetch(http://localhost:5000/api/v1/accounts/link, {
                    method: "POST",
                    body: {
                        "session-id": "session-5e213093-8b01-4290-9d96-69674d405e89"
                    }
                    headers: {
                        "session-id": "session-32b8b50f-829f-49e5-9cb6-2d1cb8f8561a"
                    }
                },
            </sample>
        </request>
        <response type="200">
            <sample lang="JSON" title="success">
                {
                    "account_linked": true
                }
            </sample>
        </response>
        <response type=""></response>
    </api-endpoint>
</api-doc>

When calling this endpoint, the user should set the `session-id` header to their account's session ID and the body `session-id` to the ID to link. In the example above: 
`session-5e213093-8b01-4290-9d96-69674d405e89` is the session to link and `session-32b8b50f-829f-49e5-9cb6-2d1cb8f8561a` is the user's login session.