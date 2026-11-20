# Account security notification

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Account security notifications use the OpenID Connect (OIDC) Back-Channel Logout
protocol. OpenAI sends your backend a signed Logout Token to request user or
session logout. Your app must validate the token and end the sessions it identifies.

1. **OpenAI** sends a signed request.
2. **Your server** verifies the token.
3. **Your app** ends matching sessions.

## User and session logout

Notifications can request user logout or session logout. Check `sub` and `sid`
to determine which sessions to end.

| Scope          | Claims                        | Sessions to end                                              |
| -------------- | ----------------------------- | ------------------------------------------------------------ |
| User logout    | `sub`, without `sid`          | All of the user's app sessions, across browsers and devices. |
| Session logout | `sid`, with an optional `sub` | Only the sessions associated with that `sid`.                |

## What the token contains

A Logout Token is a signed JSON Web Token (JWT). Its claims identify your app
and the user or session to sign out. The server-to-server request doesn't include
a browser cookie.

This decoded example targets one session. A user logout has the same format
with `sub` and no `sid`. The IDs and timestamps are illustrative.

Header:

```json
{
  "alg": "RS256",
  "kid": "example-signing-key",
  "typ": "logout+jwt"
}
```

Payload:

```json
{
  "iss": "https://auth.openai.com",
  "aud": "oaiapp_EXAMPLE",
  "sub": "example-openai-subject",
  "sid": "example-openai-session",
  "iat": 1791442800,
  "exp": 1791442920,
  "jti": "unique-logout-token-id",
  "events": {
    "http://schemas.openid.net/event/backchannel-logout": {}
  }
}
```

| Claim         | Description                                                                               |
| ------------- | ----------------------------------------------------------------------------------------- |
| `iss`         | The OpenAI issuer that signed the token.                                                  |
| `aud`         | Your application's OpenAI client ID.                                                      |
| `sub`         | The user's subject from sign-in. Required for user logout and optional with `sid`.        |
| `sid`         | The session identifier from sign-in. Limits logout to matching sessions.                  |
| `iat` / `exp` | Issuance and expiration times in Unix seconds. The maximum token lifetime is 120 seconds. |
| `jti`         | A unique token ID for replay detection.                                                   |
| `events`      | The logout instruction. The event value is an empty object.                               |

A Logout Token is only an instruction to end sessions. It can't be used to sign
in or access account data. It contains no `nonce`, email address, conversation
content, or reason for the security action.

## 1. Create your endpoint

Create a public HTTPS endpoint that accepts POST requests. OpenAI sends the
signed token in the `logout_token` form field:

```http
POST /oidc/automatic-logout HTTP/1.1
Host: partner.example.com
Content-Type: application/x-www-form-urlencoded

logout_token=<signed-JWT>
```

Parse the body as form data and require exactly one nonempty `logout_token`.
Your endpoint must accept requests without a browser login or redirect.

## 2. Verify the token and end sessions

Verify the token's signature, issuer, audience, expiry, and logout event before
ending sessions. At sign-in, store the verified issuer, client ID, and `sub` with
the user's local account. Store the verified ID token's `sid` with the app
session when provided.

- **With `sid`:** Find sessions by issuer, client ID, and `sid`. If `sub` is also present, check that the sessions belong to that user. An unknown `sid` must not trigger user logout.
- **Without `sid`:** Find the user by issuer, client ID, and `sub`, and end all their app sessions.

Revoke matching sessions and credentials that can renew them. Enforce revocation
on every server that accepts sessions.

<details>
  <summary>Example: Validate a Logout Token</summary>

Both examples allow five seconds of clock tolerance. Set `CLIENT_ID` to your registered app’s client ID.




Install `PyJWT[crypto]` for Python.







For TypeScript, install `jose`. Save the complete source as `automatic-logout.ts` and pass the signed token through standard input to `node --experimental-strip-types automatic-logout.ts`. The command verifies the token; session revocation belongs in your application.




Fetch signing keys from the configured OpenAI JWKS endpoint
(`https://auth.openai.com/.well-known/jwks.json`) and support key rotation.
Never use a key URL supplied in the token. For staging, use
`https://auth.api.openai.org` as the issuer and
`https://auth.api.openai.org/.well-known/jwks.json` as its JWKS endpoint.

```typescript
import { createRemoteJWKSet, decodeProtectedHeader, jwtVerify } from "jose";

const ISSUER = "https://auth.openai.com";
const CLIENT_ID = "oaiapp_example";
const CLOCK_TOLERANCE = 5;
const LOGOUT_EVENT = "http://schemas.openid.net/event/backchannel-logout";
const jwks = createRemoteJWKSet(
  new URL("https://auth.openai.com/.well-known/jwks.json")
);

const nonempty = (value: unknown): value is string =>
  typeof value === "string" && value.trim().length > 0;

export async function verifyLogoutToken(logoutToken: string) {
  const header = decodeProtectedHeader(logoutToken);
  if (
    header.alg !== "RS256" ||
    header.typ !== "logout+jwt" ||
    !nonempty(header.kid)
  ) {
    throw new Error("Invalid Logout Token header");
  }
  // Only the configured JWKS is trusted; never follow token-supplied key URLs.
  const { payload } = await jwtVerify(logoutToken, jwks, {
    algorithms: ["RS256"],
    issuer: ISSUER,
    audience: CLIENT_ID,
    requiredClaims: ["iss", "aud", "iat", "exp", "jti", "events"],
    clockTolerance: CLOCK_TOLERANCE,
    maxTokenAge: 120,
  });
  const { iat, exp, events } = payload;
  const now = Date.now() / 1000;
  if (
    typeof iat !== "number" ||
    !Number.isFinite(iat) ||
    typeof exp !== "number" ||
    !Number.isFinite(exp) ||
    iat > now + CLOCK_TOLERANCE ||
    exp <= iat ||
    exp > iat + 120 ||
    (!("sub" in payload) && !("sid" in payload)) ||
    ("sub" in payload && !nonempty(payload.sub)) ||
    ("sid" in payload && !nonempty(payload.sid)) ||
    !nonempty(payload.jti) ||
    !events ||
    typeof events !== "object" ||
    Array.isArray(events) ||
    !(LOGOUT_EVENT in events) ||
    "nonce" in payload
  ) {
    throw new Error("Invalid Logout Token claims");
  }
  const event = (events as Record<string, unknown>)[LOGOUT_EVENT];
  if (
    !event ||
    typeof event !== "object" ||
    Array.isArray(event) ||
    Object.keys(event).length
  ) {
    throw new Error("Invalid logout event");
  }
  return payload;
}
// Read the sensitive token from stdin, not command-line arguments or logs.
let token = "";
for await (const chunk of process.stdin) {
  token += chunk;
  if (Buffer.byteLength(token) > 65536)
    throw new Error("Logout Token is too large");
}
await verifyLogoutToken(token.trim());
console.log(
  "Logout Token verified; use the returned claims to revoke app sessions."
);
```

```python
import math
import time

import jwt
from jwt import PyJWKClient

ISSUER = "https://auth.openai.com"
CLIENT_ID = "oaiapp_example"
CLOCK_TOLERANCE = 5  # Example: allow five seconds of clock skew.
LOGOUT_EVENT = "http://schemas.openid.net/event/backchannel-logout"
jwks = PyJWKClient("https://auth.openai.com/.well-known/jwks.json", timeout=5)


def nonempty_string(value):
    return isinstance(value, str) and bool(value.strip())


def numeric_date(value):
    return type(value) is int or (type(value) is float and math.isfinite(value))


def verify_logout_token(logout_token):
    header = jwt.get_unverified_header(logout_token)
    if (
        header.get("alg") != "RS256"
        or header.get("typ") != "logout+jwt"
        or not nonempty_string(header.get("kid"))
    ):
        raise jwt.InvalidTokenError("Invalid Logout Token header")

    # Read keys only from the configured JWKS URL, then verify the signature.
    signing_key = jwks.get_signing_key(header["kid"])
    claims = jwt.decode(
        logout_token,
        signing_key.key,
        algorithms=["RS256"],
        issuer=ISSUER,
        audience=CLIENT_ID,
        options={
            "require": ["iss", "aud", "iat", "exp", "jti", "events"],
            # Check time claims below without coercing strings or booleans.
            "verify_iat": False,
            "verify_exp": False,
            "verify_nbf": False,
        },
    )

    now = time.time()
    issued_at, expires_at = claims["iat"], claims["exp"]
    not_before = claims.get("nbf", now)
    events = claims["events"]
    if (
        not numeric_date(issued_at)
        or not numeric_date(expires_at)
        or not now - 120 - CLOCK_TOLERANCE <= issued_at <= now + CLOCK_TOLERANCE
        or not issued_at < expires_at <= issued_at + 120
        or expires_at <= now - CLOCK_TOLERANCE
        or not numeric_date(not_before)
        or not_before > now + CLOCK_TOLERANCE
        or not ("sub" in claims or "sid" in claims)
        or ("sub" in claims and not nonempty_string(claims["sub"]))
        or ("sid" in claims and not nonempty_string(claims["sid"]))
        or not nonempty_string(claims["jti"])
        or not isinstance(events, dict)
        or events.get(LOGOUT_EVENT) != {}
        or "nonce" in claims
    ):
        raise jwt.InvalidTokenError("Invalid Logout Token")

    return claims
```


After validation, use the returned claims to select and revoke sessions. In your
endpoint, parse the form body, handle duplicate requests, and return HTTP
responses.




In Python, catch
`jwt.InvalidTokenError` for invalid tokens. An unknown signing key can raise
`jwt.PyJWKClientError`; distinguish an unknown key in a successfully fetched key
set from a failure to fetch keys or a storage failure. Treat both failures as
server errors.







In TypeScript, reject tokens that fail validation without revoking sessions. Handle JWKS fetch failures as temporary server failures and distinguish them from an invalid token or an unknown signing key. Do not return verifier diagnostics or tokens to callers.




</details>

### Return a response

Return an HTTP status code to indicate the outcome of processing the notification.

| Status         | When to return it                                                                    |
| -------------- | ------------------------------------------------------------------------------------ |
| `200` or `204` | Logout is complete, the token was already processed, or no matching sessions remain. |
| `400`          | The token is invalid. Leave sessions unchanged.                                      |
| `503`          | A temporary server failure prevents validation or logout.                            |

## 3. Configure your endpoint

Configure your endpoint in OpenAI Platform to receive account security notifications.

1. In [OpenAI Platform](https://platform.openai.com/), select **OAuth apps** in the sidebar, select your client, and open **Security notifications**.
2. Select **Add logout endpoint** and enter your endpoint URL.
3. Select **Save endpoint**.
4. Select **Enable automatic logout** and confirm the action.
5. Check that **Automatic logout** shows **On** and the saved endpoint URL is correct.

If either change requires review, submit it and publish it after approval before
continuing. Existing settings stay active until publication. Saving the endpoint
alone doesn't enable production notifications.

If **Security notifications** is unavailable, contact OpenAI about access.

## 4. Test your integration

Use **Test endpoint** to send a signed Logout Token to your saved endpoint. The
test checks delivery; it doesn't verify that your app logs users out or enable
production notifications.

1. In **Security notifications**, select **Test endpoint**.
2. Enter any `sub` value in **Test user ID** and select **Send test**.
3. Confirm your endpoint returns `200` or `204`.

Test notifications automatically add `test-` to the `sub` value. For example,
`user-123` becomes `test-user-123`. To test logout for an existing user, validate
the Logout Token, then remove the leading `test-` from the received `sub` before
looking up the user and ending their sessions. Only remove this prefix when
processing test notifications.