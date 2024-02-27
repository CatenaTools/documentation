# Account Login and Linking

Catena's dashboard provides a few components to support the [Login Flow](Auth-Flow-Session-Management.md) and [Account Linking Flow](Account-Linking.md). These are supported through a combination of the Login and Link pages and the LoginForm component.

## Login Page

[//]: # (Note: The paths in this section and the following are important and must correspond to the paths in the frontend repo.)

When you first run the dashboard and navigate to it in a browser you will be greeted with a login page. It will look something like the following:

![](loginPage.png)

Which providers are shown here will vary depending on your configuration. In this case, clicking Google and Discord will both bring you through the authentication flow for those providers, and unsafe will allow you to login with an unsafe username.

The page itself is located within the frontend repo at `/src/app/login/page.tsx`. You are free to customize this page as needed in order to support your needs.

```Typescript
import LoginForm from "@/lib/components/PageComponents/LoginForm"
import { LoginFormProvider } from "@/lib/core/authentication/LoginFormProvider"
import { getEnabledProviders } from "@/lib/core/authentication/getEnabledProviders"
const catena_dashboard_url = process.env.NEXT_PUBLIC_DASHBOARD_URL

export default async function Login() {
  const providers = await getEnabledProviders();

  const enabledProviders = providers.enabledProviders.map((providerString) => {
    return LoginFormProvider[providerString] as LoginFormProvider
  })

  return (
    <>
      <main className="flex min-h-screen flex-col justify-center items-center w-full">
        <LoginForm
          enabledProviders={enabledProviders}
          targetURL={`${catena_dashboard_url}/dashboard/accounts`}
        ></LoginForm>
      </main>
    </>
  )
}
```

### Breaking Down the Code

In order to render this page, we first retrieve the providers which are enabled on the backend and convert them to an enum.

```Typescript
  const providers = await getEnabledProviders();

  const enabledProviders = providers.enabledProviders.map((providerString) => {
    return LoginFormProvider[providerString] as LoginFormProvider
  })
```

We then pass this data, along with a `targetURL` into the `LoginForm` component. The target url tells the LoginForm where to 
redirect the user after they have successfully logged into the platform.

```Typescript
<LoginForm
  enabledProviders={enabledProviders}
  targetURL={`${catena_dashboard_url}/dashboard/accounts`}
></LoginForm>
```

This means to customize the page itself you can add code to either `page.tsx` or to the `layout.tsx` in the same directory, however changing the form itself should be in the `LoginForm` component.

## LinkPage

The link page looks very similar to the login page, that is because it uses the same form component, this ensures we are consistent in how we apply styles across the platform.

Visit `/link` in order to view the link page on your dashboard. You should see a screen like the one below:

![linkPage.png](linkPage.png)

The code for the link page is located at `/src/app/link/page.tsx` and should look very similar to the login page.

The link page's code will look mostly similar to the login page, however the target url and layout is a bit different:

```Typescript
<div className={"w-full h-96 flex justify-center items-center"}>
  <LoginForm targetURL={`${catena_dashboard_url}/link/step2?` + urlProviders.toString()} enabledProviders={enabledProviders}></LoginForm>
</div>
```

Notice we are now sending the user to `/link/step2` once login is complete, we are also encoding the enabled providers as a url parameter so we do not have to re-fetch the enabled providers on the next page.

If you log in with a provider you will be sent to the `/link/step2` route which will allow you to complete the second part of the linking process.

## LinkPage Step 2

The code for this page is at `/src/app/link/step2/page.tsx`. This page allows the user to complete the linking flow by logging in with the provider they want to link.

The example below shows what a user may see if they have linked google and unsafe providers to the same account.

![linkPageStep2.png](linkPageStep2.png)

This page must juggle a few session tokens in order to complete the login step. It will do the following:

1. Complete the user's login flow 
2. Persist the user's login session token in local storage. It must retrieve this from the Next.js backend which maintains the state for the user. This token will be used in the next step.
```Typescript
    const jsonData: { account: Account } = await getLoginInfo.json()
    const getCatenaSessionID = await fetch(`/api/v1/authentication/session`, {
      credentials: "include",
    })
    const sessionData: { session: string } = await getCatenaSessionID.json()
```
3. Allow the user to log in with another provider.
4. Redirect the user to the final page `/link/step3`


## Link Page Step 3

This page uses a helper API route to perform the actual linking on the backend. The code for the page is at `src/app/link/step3/page.tsx`.

```Typescript
export default function LinkStep3() {
  const [linked, setLinked] = useState(false)

  async function CompleteLogin() {
    const getSessionIDToLink = await fetch(`/api/v1/authentication/session`, {
      credentials: "include"
    })

    const sessionDataToLink: { session: string } = await getSessionIDToLink.json()

    const originalSession = localStorage.getItem("OriginalSession")

    return {
      sessionToLink: sessionDataToLink.session,
      originalSession: originalSession ? originalSession : "",
    }
  }

  useEffect(() => {
    CompleteLogin().then((data) => {

      if (linked) {
        return () => {}
      }
      fetch(`/api/v1/accounts/link`, {
        method: "POST",
        body: JSON.stringify({
          session: data.originalSession,
          sessionToLink: data.sessionToLink,
        }),
      }).then((res) => {
        if (res.status !== 200) {
          console.error("Error linking account")
        } else {
          setLinked(true)
        }
      })
    })
    return () => {}
  }, [])

  return (
    <>
      <div className={"mt-1 ml-3"}>
        <CatenaPageHeader
          title={"Link Account Step 3"}
          subtitle={"Link flow complete"}
          button={<></>}
        ></CatenaPageHeader>
        <p>{linked ? "Account successfully linked." : "Failed to link account."} You can now close this page</p>
      </div>
    </>
  )
}
```

The important code here is the CompleteLogin() helper, which is responsible for sending the user's original session and new 
session which they retrieved from the provider to be linked to the original account. This relies on a helper defined in the Next.js backend defined as a route `src/app/api/v1/accounts/link/route.ts`. Code shown below:

```Typescript
// This helper exists to make the client experience easier for link account to provider, the backend
// request requires us to pass the session in the session-id header and the session to link in the request
// this wrapper takes both ids in the request so the client side doesn't have to shuffle them. It will also correctly set
// the client cookie to the original session.
export async function POST(request: Request, response: Response) {
  const URL = process.env.NEXT_PUBLIC_CATENA_URL
    ? process.env.NEXT_PUBLIC_CATENA_URL
    : "http://localhost:5001"

  const reqJson: { session?: string; sessionToLink?: string } =
    await request.json()

  if (!("session" in reqJson)) {
    return new Response("Bad request", {
      status: 400,
    })
  }

  if (!("sessionToLink" in reqJson)) {
    return new Response("Bad request, missing linkSession field", {
      status: 400,
    })
  }

  const session = await getIronSession<CatenaSession>(cookies(), {
    password: password,
    cookieName: cookieName,
    cookieOptions: {
      secure: false,
    },
  })

  const res = await fetch(URL + "/api/v1/accounts/link", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "session-id": reqJson.session!,
      "request-id": DashboardUUID(),
    },
    body: JSON.stringify({
      session_id: reqJson.sessionToLink,
    }),
  })

  if (response.status === 200) {
    console.log("Successfully linked account with session", reqJson.session!)
    session.sessionID = reqJson.session!

    await session.save()
  }

  return res
}

```

This code does a few things:

1. Pull the two session tokens from the request, `reqJson.session` is the original login session and `reqJson.sessionToLink` is the session id referring to the provider we'd like to link to the original account.
2. Constructs a request to the backend with the session ids in the correct place
3. Waits for a response and validates that response, if it is valid, we set the Next.js session cookie to point to the original session, logging the user in with the correct account.

## LoginForm

The LoginForm component defines the styles and function for all the dashboard pages which require auth. If you wish to customize the login page, this is the place to do it.

The render function for the form is shown here: 

```Typescript
return (
<>
  <div className="shadow-lg bg-black w-4/5 max-w-96 min-h-64 text-white rounded-md">
    <div className="h-16 flex justify-beginning items-center">
      <Image
        className="m-2 "
        height={48}
        width={48}
        src={logo.src}
        alt="CatenaTools"
      />
    </div>
    <div className="mx-4 my-2">
      {
        BuildFormFromProps()
      }
      <p className="text-red-500">{errorText}</p>
    </div>
  </div>
</>
)
```

`BuildFormFromProps()` is responsible for populating the buttons on the form, the surrounding code can be changed to style things however you wish.

The `<Image>` tag populates the logo and can be swapped for your logo.