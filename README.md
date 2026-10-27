# AuthHero SDK

AuthHero exposes a JSON HTTP API usable from any language and a Node.js SDK for server integrations. The SDK requires Node.js 18 or newer and has no runtime dependencies.

## Install

After this package is published:

```sh
npm install @authhero/node-sdk
```

To try it from this repository before publishing:

```sh
npm install ./packages/node-sdk
```

## Configure

Set the project details in your server environment. Do not put credentials in browser code or commit `.env` files.

```env
AUTHHERO_URL=http://localhost:4000
AUTHHERO_PUBLISHABLE_KEY=ah_pub_your_project_key
```

Create one client per user/device session and keep it on the server:

```js
import { AuthHeroClient } from '@authhero/node-sdk';

const auth = new AuthHeroClient({
  baseUrl: process.env.AUTHHERO_URL,
  publishableKey: process.env.AUTHHERO_PUBLISHABLE_KEY,
});
```

The SDK generates a URL-safe device ID for each client instance. Supply `deviceId` when constructing the client if the application needs to keep the same device identity across instances.

## Run the Flow Studio authentication experience

Configure and save the flow and authenticated destination in **Flow Studio**. Add the application host to the AuthHero server's `ALLOWED_REDIRECT_HOSTS` setting. The destination may be a relative path (such as `/dashboard`) or an allowed absolute URL.

In a browser application, load the browser SDK and start the project flow:

```html
<script src="https://auth.example.com/auth_hero.js?api=ah_pub_your_project_key"></script>
<script>
  AuthHero.startAuthFlow(window.location.href);
</script>
```

The **Open live flow** action uses AuthHero's hosted sign-in page at `https://auth.example.com/flow.html`. It runs the saved steps, sends verification/magic-link/reset emails, then returns to the validated requested URL or the Studio destination. Email-verification steps can select a saved project email template in Flow Studio.

### Use a custom sign-in UI

You can keep your own page design and use the saved Flow Studio steps as the authentication flow. Load the browser SDK on your page, retrieve the saved flow, and submit form values for the active Studio step:

```html
<script src="https://auth.example.com/auth_hero.js?api=ah_pub_your_project_key"></script>
<div id="authhero-attribution"></div>
<script type="module">
  const attribution = document.querySelector('#authhero-attribution');
  AuthHero.renderPoweredBy(attribution);

  const { flow } = await AuthHero.getAuthFlow({ returnTo: window.location.href });
  let activeStep = flow.nodes[0];

  async function submitCurrentStep(values) {
    try {
      const result = await AuthHero.submitFlowStep(activeStep.id, values);
      if (result.pending) {
        // Keep the current step open while the user checks their email.
        return result;
      }
      activeStep = result.nextStep;
      if (activeStep?.type === 'authenticated') {
        window.location.assign(flow.destination || '/');
      }
      return result;
    } catch (error) {
      // The Studio step's onFailure connection identifies the retry/alternate step.
      if (activeStep.onFailure) {
        activeStep = flow.nodes.find((step) => step.id === activeStep.onFailure);
      }
      throw error;
    }
  }
</script>
```

Render the fields for each step type in your own UI and pass their values to `submitCurrentStep`, for example `{ email, password }` for sign-in/sign-up, `{ email }` to request an email code, and `{ email, code }` to verify it. Magic-link requests use `{ email }` and confirmations use `{ token }`; password-recovery requests use `{ email }` and completions use `{ token, newPassword }`. The SDK handles API requests, applies the Studio-selected verification template, and stores successful browser sessions. Errors are rejected to your UI instead of being silently converted into success.

`AuthHero.renderPoweredBy()` adds a “Powered by AuthHero” link to the element you provide. Keep this attribution visible on custom authentication pages. The hosted AuthHero page remains available alongside custom UI. Magic-link and password-reset email links currently open the AuthHero-hosted completion page.

Browser sessions are held in `sessionStorage` for the tab. `haveAuth()` calls the AuthHero API to validate the access token; it is asynchronous and should not be treated as server-side authorization:

```js
if (!(await AuthHero.haveAuth())) {
  AuthHero.startAuthFlow(window.location.href);
}
```

Protect sensitive data and APIs on your server as well. A browser check alone cannot protect server resources.

## Sign up and request verification

Signing up creates an account and session; it does not automatically send a verification email. Request the verification email explicitly:

```js
const session = await auth.signUp({
  email: 'alex@example.com',
  password: 'a-long-user-password',
});

await auth.requestEmailVerification(session.user.email);
```

After the user enters the emailed code:

```js
const result = await auth.verifyEmail(session.user.email, code);
```

## Sign in and access the current session

```js
const session = await auth.signIn({
  email: 'alex@example.com',
  password: 'a-long-user-password',
});

const user = await auth.getCurrentUser();
const { currentSessionId, sessions } = await auth.getSessions();

await auth.refresh();
await auth.signOut();
```

Access and refresh tokens are held in memory on the Node client instance. Store them only in an appropriately protected server-side session if they need to persist between requests; never expose them to the browser or logs.

To check a current Node client session with the AuthHero API and obtain the hosted-flow URL:

```js
if (!(await auth.haveAuth())) {
  const loginUrl = auth.getAuthFlowUrl({ returnTo: 'https://app.example.com/dashboard' });
}
```

## Protect Express routes

Install Express and your chosen session middleware in your application. After sign-in, keep the AuthHero session in your server-side session store, then install the middleware on protected routes:

```js
import { createAuthMiddleware } from '@authhero/node-sdk/express';

app.get('/login', (req, res) => {
  res.redirect(auth.getAuthFlowUrl({ returnTo: '/dashboard' }));
});

app.get('/protected', createAuthMiddleware({
  baseUrl: process.env.AUTHHERO_URL,
  publishableKey: process.env.AUTHHERO_PUBLISHABLE_KEY,
  redirectTo: '/login',
  getAccessToken: (req) => req.session?.authhero?.accessToken,
}), (req, res) => {
  res.send(`Hello ${req.authhero.email}`);
});
```

The middleware validates the bearer token against AuthHero on each protected request, adds the verified user to `req.authhero`, redirects unauthenticated requests, and passes network/server errors to Express `next(error)`.

## Email links and password recovery

```js
await auth.requestMagicLink('alex@example.com');
const session = await auth.confirmMagicLink(tokenFromEmail);

await auth.requestPasswordReset('alex@example.com');
await auth.resetPassword(resetTokenFromEmail, 'a-new-long-password');
```

For an authenticated password change:

```js
await auth.changePassword('current-password', 'new-long-password');
```

## Errors

API errors reject with `AuthHeroError`, which includes the HTTP status and parsed server response:

```js
import { AuthHeroClient, AuthHeroError } from '@authhero/node-sdk';

try {
  await auth.signIn({ email, password });
} catch (error) {
  if (error instanceof AuthHeroError) {
    console.error(`AuthHero request failed with status ${error.status}`);
  }
  throw error;
}
```

Network failures are surfaced as the errors from Node.js `fetch`.

## HTTP API (any language)

Send the project publishable key in `X-AuthHero-Api-Key`. For session-protected routes, also send `Authorization: Bearer <accessToken>`. Auth requests accept JSON and return `{ "success": true, "data": ... }`:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/auth/register` | Create an account and session |
| `POST` | `/auth/login` | Sign in and create a session |
| `POST` | `/auth/email-verification/request` | Send verification code; optional `templateId` |
| `POST` | `/auth/email-verification/verify` | Verify email code |
| `POST` | `/auth/magic-link/request` | Send sign-in link |
| `POST` | `/auth/magic-link/confirm` | Confirm a link and create session |
| `POST` | `/auth/password-reset/request` | Send password reset link |
| `POST` | `/auth/password-reset/confirm` | Set a password with reset token |
| `GET` | `/auth/me` | Validate token and return current user |
| `POST` | `/auth/logout` | Revoke current session |
| `GET` | `/auth/flow` | Read saved public flow configuration |

Example email verification request:

```sh
curl -X POST "$AUTHHERO_URL/auth/email-verification/request" \
  -H "Content-Type: application/json" \
  -H "X-AuthHero-Api-Key: $AUTHHERO_PUBLISHABLE_KEY" \
  -d '{"email":"alex@example.com","templateId":"your-template-id"}'
```
