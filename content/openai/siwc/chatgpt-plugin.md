# In your ChatGPT plugin

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Sign in with ChatGPT is currently available to selected commercial partners
  through a limited trial. To join the waitlist, contact your OpenAI
  representative.

Add Sign in with ChatGPT to let users sign in to your connector with their
ChatGPT account. They start the flow by selecting **Continue with ChatGPT** in
ChatGPT's plugin modal.

When a user starts this flow, ChatGPT begins your connector's OAuth flow. Your
application validates the OpenAI ID token, then completes connector
authorization.



> Illustration: A ChatGPT plugin page with a centered Connect Acme modal. ChatGPT and partner icons appear above the description, Plan projects and move work forward. The modal offers Sign in with Acme and a primary Continue with ChatGPT button, while the plugin page remains visible behind it.



## Understand the two flows

This integration contains two independent OAuth transactions. The outer
transaction authorizes your connector for ChatGPT. The inner transaction signs the user
in with ChatGPT and validates the OpenAI ID token. Keep their state, PKCE values, callbacks,
authorization codes, and credentials separate.

| Transaction                | OAuth client     | Authorization server | Result                                                                |
| -------------------------- | ---------------- | -------------------- | --------------------------------------------------------------------- |
| Outer: Connect your plugin | ChatGPT          | Your application     | Your connector authorization code, returned to ChatGPT.               |
| Inner: Sign in the user    | Your application | OpenAI               | An OpenAI authorization code, exchanged and verified by your backend. |

The inner transaction uses the [website sign-in flow](https://developers.openai.com/siwc/website). After
verifying the OpenAI ID token, your application identifies or creates a local
account and resumes the saved outer transaction.

## 1. Accept ChatGPT's authorization request

ChatGPT initiates a connector authorization request to your endpoint:

```text
GET /oauth/authorize?
  response_type=code
  &client_id=CHATGPT_CONNECTOR_CLIENT_ID
  &redirect_uri=REGISTERED_CHATGPT_CALLBACK
  &scope=REQUESTED_CONNECTOR_SCOPES
  &state=CHATGPT_GENERATED_STATE
  &code_challenge=CHATGPT_PKCE_CHALLENGE
  &code_challenge_method=S256
  &resource=CONNECTOR_RESOURCE
  &login_hint=USER_EMAIL_HINT
  &target_flow=chatgpt_siwc
```

Validate the registered ChatGPT connector client, exact redirect URI, supported
scopes, and outer PKCE challenge. Validate `resource` when your connector
implements resource indicators.

`target_flow=chatgpt_siwc` indicates a Sign in with ChatGPT-enabled connector
experience. Use `login_hint`, which contains the user's email, to suggest an
account or route to enterprise SSO.

When the hint matches an existing authenticated session and enterprise sign-in
requirements are satisfied, continue to connector consent without showing another
account selector. If the accounts differ, let the user choose. The hint does not verify the user's identity.

## 2. Save the outer connector transaction

Persist the validated connector request before starting inner sign-in. The
example below uses an application-owned registry of the connector client,
callback, scopes, and resource.

```typescript
export function saveConnectorTransaction(
  browserSessionId: string,
  requestUrl: URL
) {
  const params = requestUrl.searchParams;
  const scope = (params.get("scope") ?? "").split(" ").filter(Boolean);
  const codeChallenge = params.get("code_challenge") ?? "";
  if (
    params.get("response_type") !== "code" ||
    params.get("client_id") !== registeredConnector.clientId ||
    params.get("redirect_uri") !== registeredConnector.redirectUri ||
    params.get("code_challenge_method") !== "S256" ||
    !/^[A-Za-z0-9_-]{43}$/.test(codeChallenge) ||
    !params.get("state") ||
    scope.some((value: string) => !registeredConnector.scopes.has(value)) ||
    params.get("resource") !== registeredConnector.resource
  ) {
    throw new Error("The connector authorization request is invalid.");
  }

  const transaction = {
    id: randomValue(),
    browserSessionId,
    clientId: registeredConnector.clientId,
    redirectUri: registeredConnector.redirectUri,
    state: params.get("state"),
    codeChallenge,
    codeChallengeMethod: "S256",
    scope: scope.join(" "),
    resource: registeredConnector.resource,
    loginHint: params.get("login_hint"),
    targetFlow: params.get("target_flow"),
    expiresAt: Date.now() + 10 * 60 * 1000,
  };
  transactions.set(transaction.id, transaction);
  return transaction;
}
```

```python
def save_connector_transaction(
    browser_session_id: str, request_url: str
) -> ConnectorTransaction:
    """Validate the registered request before accepting its redirect destination."""
    _required(browser_session_id, "browser session")
    if not isinstance(request_url, str) or len(request_url) > _MAX_INPUT_BYTES:
        raise ValueError("The connector authorization request is invalid.")
    url = urlsplit(request_url)
    if url.scheme != "https" or not url.netloc or url.fragment:
        raise ValueError("An HTTPS authorization URL without a fragment is required.")
    params = _unique_form(url.query)
    registration = REGISTERED_CONNECTOR
    scope = [value for value in params.get("scope", "").split(" ") if value]
    challenge = params.get("code_challenge", "")
    if (
        params.get("response_type") != "code"
        or params.get("client_id") != registration.client_id
        or params.get("redirect_uri") != registration.redirect_uri
        or params.get("code_challenge_method") != "S256"
        or not re.fullmatch(r"[A-Za-z0-9_-]{43}", challenge)
        or not params.get("state")
        or len(params["state"]) > 2048
        or any(value not in registration.scopes for value in scope)
        or params.get("resource") != registration.resource
    ):
        raise ValueError("The connector authorization request is invalid.")

    with _lock:
        now = time.time()  # Python timestamps use seconds, not milliseconds.
        _prune(now)
        if len(_transactions) >= _MAX_PENDING:
            raise ValueError("Too many pending connector transactions.")
        transaction = ConnectorTransaction(
            id=secrets.token_urlsafe(32),
            browser_session_id=browser_session_id,
            client_id=registration.client_id,
            redirect_uri=registration.redirect_uri,
            state=params["state"],
            code_challenge=challenge,
            code_challenge_method="S256",
            scope=" ".join(scope),
            resource=registration.resource,
            login_hint=params.get("login_hint"),
            target_flow=params.get("target_flow"),
            expires_at=now + 10 * 60,
        )
        _transactions[transaction.id] = transaction
        return transaction
```

```go
import (
	"crypto/rand"
	"crypto/sha256"
	"crypto/subtle"
	"encoding/base64"
	"encoding/json"
	"errors"
	"fmt"
	"net/url"
	"os"
	"regexp"
	"strings"
	"sync"
	"time"
)

const clientID = "chatgpt_connector_example"
const redirectURI = "https://chatgpt.com/connector-callback/example"
const resourceURI = "https://example.com/mcp"
const maxPending = 1024

var challengePattern = regexp.MustCompile(`^[A-Za-z0-9_-]{43}$`)
var verifierPattern = regexp.MustCompile(`^[A-Za-z0-9._~-]{43,128}$`)

type Transaction struct {
	ID, BrowserSessionID, ClientID, RedirectURI, State  string
	CodeChallenge, CodeChallengeMethod, Scope, Resource string
	LoginHint, TargetFlow                               string
	ExpiresAt                                           time.Time
}
type Grant struct {
	UserID, ClientID, RedirectURI, CodeChallenge, Scope, Resource string
	ExpiresAt                                                     time.Time
}
type User struct{ ID, Email string }
type Experience struct {
	Action         string `json:"action"`
	ReturnTo       string `json:"returnTo,omitempty"`
	UserID         string `json:"userId,omitempty"`
	SuggestedEmail string `json:"suggestedEmail,omitempty"`
}

// One process owns this bounded store. A multi-instance deployment needs a shared
// store with atomic consume operations and equivalent expiration/capacity limits.
type Connector struct {
	mu           sync.Mutex
	transactions map[string]Transaction
	codes        map[string]Grant
}

func newConnector() *Connector {
	return &Connector{transactions: make(map[string]Transaction), codes: make(map[string]Grant)}
}
func randomValue() (string, error) {
	var b [32]byte
	if _, err := rand.Read(b[:]); err != nil {
		return "", err
	}
	return base64.RawURLEncoding.EncodeToString(b[:]), nil
}
func (c *Connector) prune(now time.Time) {
	for id, t := range c.transactions {
		if !now.Before(t.ExpiresAt) {
			delete(c.transactions, id)
		}
	}
	for code, g := range c.codes {
		if !now.Before(g.ExpiresAt) {
			delete(c.codes, code)
		}
	}
}

func (c *Connector) SaveConnectorTransaction(browserSessionID string, requestURL *url.URL) (Transaction, error) {
	invalid := errors.New("the connector authorization request is invalid")
	if requestURL == nil || strings.TrimSpace(browserSessionID) == "" || len(requestURL.RawQuery) > 16384 {
		return Transaction{}, invalid
	}
	params, err := url.ParseQuery(requestURL.RawQuery)
	if err != nil {
		return Transaction{}, invalid
	}
	// Reject duplicate parameters rather than interpreting ambiguous OAuth inputs.
	for _, values := range params {
		if len(values) != 1 {
			return Transaction{}, invalid
		}
	}
	scope := []string{}
	for _, value := range strings.Split(params.Get("scope"), " ") {
		if value != "" {
			scope = append(scope, value)
		}
	}
	for _, value := range scope {
		if value != "documents.read" {
			return Transaction{}, invalid
		}
	}
	challenge := params.Get("code_challenge")
	if params.Get("response_type") != "code" || params.Get("client_id") != clientID ||
		params.Get("redirect_uri") != redirectURI || params.Get("code_challenge_method") != "S256" ||
		!challengePattern.MatchString(challenge) || params.Get("state") == "" || params.Get("resource") != resourceURI {
		return Transaction{}, invalid
	}
	id, err := randomValue()
	if err != nil {
		return Transaction{}, err
	}
	now := time.Now()
	transaction := Transaction{
		ID: id, BrowserSessionID: browserSessionID, ClientID: clientID, RedirectURI: redirectURI,
		State: params.Get("state"), CodeChallenge: challenge, CodeChallengeMethod: "S256",
		Scope: strings.Join(scope, " "), Resource: resourceURI, LoginHint: params.Get("login_hint"),
		TargetFlow: params.Get("target_flow"), ExpiresAt: now.Add(10 * time.Minute),
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	c.prune(now)
	if len(c.transactions) >= maxPending {
		return Transaction{}, errors.New("connector transaction capacity reached")
	}
	c.transactions[id] = transaction
	return transaction, nil
}
```

```java
import com.fasterxml.jackson.core.JsonParser;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.URI;
import java.net.URLDecoder;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.time.Clock;
import java.util.Base64;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

// Replace these illustrative values with an approved connector registration.
private static final String CLIENT_ID = "chatgpt_connector_example";
private static final String REDIRECT_URI = "https://chatgpt.com/connector-callback/example";
private static final String RESOURCE = "https://example.com/mcp";
private static final Set<String> SCOPES = Set.of("documents.read");
private static final int MAX_PENDING = 10_000;
private final Map<String, Transaction> transactions = new HashMap<>();
private final Map<String, Grant> authorizationCodes = new HashMap<>();
private final SecureRandom random = new SecureRandom();
private final Clock clock;

public SignInWithChatGptConnector() {
  this(Clock.systemUTC());
}

public SignInWithChatGptConnector(Clock clock) {
  this.clock = java.util.Objects.requireNonNull(clock);
}

public record Transaction(
    String id,
    String browserSessionId,
    String clientId,
    String redirectUri,
    String state,
    String codeChallenge,
    String codeChallengeMethod,
    String scope,
    String resource,
    String loginHint,
    String targetFlow,
    long expiresAt) {}

public record User(String id, String email) {
  public User {
    if (!nonempty(id) || !nonempty(email)) {
      throw new IllegalArgumentException("Authenticated user ID and email are required.");
    }
  }
}

public record Grant(
    String userId,
    String clientId,
    String redirectUri,
    String codeChallenge,
    String scope,
    String resource,
    long expiresAt) {}

public synchronized Transaction saveConnectorTransaction(
    String browserSessionId, URI requestUrl) {
  Map<String, String> params = parseForm(requestUrl.getRawQuery());
  String scope = params.getOrDefault("scope", "");
  List<String> scopes = scope.isBlank() ? List.of() : List.of(scope.strip().split(" +"));
  String codeChallenge = params.getOrDefault("code_challenge", "");
  if (!nonempty(browserSessionId)
      || !"code".equals(params.get("response_type"))
      || !CLIENT_ID.equals(params.get("client_id"))
      || !REDIRECT_URI.equals(params.get("redirect_uri"))
      || !"S256".equals(params.get("code_challenge_method"))
      || !codeChallenge.matches("[A-Za-z0-9_-]{43}")
      || !nonempty(params.get("state"))
      || !SCOPES.containsAll(scopes)
      || !RESOURCE.equals(params.get("resource"))) {
    throw new IllegalArgumentException("The connector authorization request is invalid.");
  }
  purgeExpired();
  if (transactions.size() >= MAX_PENDING) {
    throw new IllegalStateException("Too many pending connector transactions.");
  }
  Transaction transaction =
      new Transaction(
          randomValue(),
          browserSessionId,
          CLIENT_ID,
          REDIRECT_URI,
          params.get("state"),
          codeChallenge,
          "S256",
          String.join(" ", scopes),
          RESOURCE,
          params.get("login_hint"),
          params.get("target_flow"),
          clock.millis() + 10 * 60 * 1000);
  transactions.put(transaction.id(), transaction);
  return transaction;
}
```

```csharp
using System.Collections.Specialized;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Text.RegularExpressions;
using System.Web;

internal static Transaction SaveConnectorTransaction(string browserSessionId, Uri requestUrl)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(browserSessionId);
    if (requestUrl.OriginalString.Length > 16384)
        throw new ArgumentException("The connector authorization request is too large.");
    NameValueCollection parameters = HttpUtility.ParseQueryString(requestUrl.Query);
    string? Read(string key) => Single(parameters, key);
    string[] scopes = (Read("scope") ?? "").Split(' ', StringSplitOptions.RemoveEmptyEntries);
    string challenge = Read("code_challenge") ?? "";
    string? state = Read("state");
    if (Read("response_type") != "code" || Read("client_id") != ClientId ||
        Read("redirect_uri") != RedirectUri || Read("code_challenge_method") != "S256" ||
        !Regex.IsMatch(challenge, "\\A[A-Za-z0-9_-]{43}\\z") ||
        string.IsNullOrEmpty(state) || state.Length > 4096 ||
        scopes.Any(scope => !AllowedScopes.Contains(scope)) || Read("resource") != Resource)
    {
        throw new ArgumentException("The connector authorization request is invalid.");
    }
    Transaction transaction = new(RandomValue(), browserSessionId, ClientId, RedirectUri,
        state, challenge, "S256", string.Join(' ', scopes), Resource,
        Read("login_hint"), Read("target_flow"), DateTimeOffset.UtcNow.AddMinutes(10));
    lock (Gate)
    {
        PruneExpired();
        if (Transactions.Count >= Capacity)
            throw new InvalidOperationException("Too many pending connector transactions.");
        Transactions.Add(transaction.Id, transaction);
    }
    return transaction;
}
```

```ruby
def save_connector_transaction(browser_session_id, request_url)
  raise ArgumentError, "Missing browser session" unless nonempty?(browser_session_id)

  url = URI.parse(request_url)
  params = unique_parameters(url.query.to_s)
  scope = params.fetch("scope", "").split(" ")
  challenge = params.fetch("code_challenge", "")
  unless params["response_type"] == "code" &&
         params["client_id"] == REGISTRATION[:client_id] &&
         params["redirect_uri"] == REGISTRATION[:redirect_uri] &&
         params["code_challenge_method"] == "S256" &&
         /\A[A-Za-z0-9_-]{43}\z/.match?(challenge) &&
         nonempty?(params["state"]) &&
         (scope - REGISTRATION[:scopes]).empty? &&
         params["resource"] == REGISTRATION[:resource]
    raise ArgumentError, "The connector authorization request is invalid."
  end

  transaction = {
    id: SecureRandom.urlsafe_base64(32, false),
    browser_session_id: browser_session_id.dup.freeze,
    client_id: REGISTRATION[:client_id],
    redirect_uri: REGISTRATION[:redirect_uri],
    state: params.fetch("state"),
    code_challenge: challenge,
    code_challenge_method: "S256",
    scope: scope.join(" "),
    resource: REGISTRATION[:resource],
    login_hint: params["login_hint"],
    target_flow: params["target_flow"],
    expires_at: Time.now.to_f + 10 * 60
  }
  transaction.each_value { |value| value.freeze }
  transaction.freeze
  @lock.synchronize do
    remove_expired
    raise "Too many connector transactions" if @transactions.length >= MAX_ENTRIES

    @transactions[transaction[:id]] = transaction
  end
  transaction
end
```


Bind the transaction to a server-generated browser session, set a short lifetime,
and protect the session identifier with an `HttpOnly`, `Secure` cookie. Use your
production session store instead of the example's in-memory map.

## 3. Choose the account experience

Use `target_flow`, the existing first-party session, and `login_hint` to select
the next experience. Check enterprise policy before continuing authorization.

```typescript
export function chooseAccountExperience(
  currentUser: { id: string; email: string } | null,
  connector: ReturnType<typeof saveConnectorTransaction>,
  enterpriseSso: boolean
) {
  const loginHint = connector.loginHint;
  if (enterpriseSso) {
    return { action: "enterprise-sign-in", returnTo: connector.id };
  }
  if (currentUser && (!loginHint || currentUser.email === loginHint)) {
    return { action: "connector-consent", userId: currentUser.id };
  }
  if (currentUser && loginHint) {
    return { action: "account-selector", suggestedEmail: loginHint };
  }
  if (connector.targetFlow !== "chatgpt_siwc") {
    return { action: "application-sign-in", returnTo: connector.id };
  }
  return { action: "sign-in-with-chatgpt", returnTo: connector.id };
}
```

```python
def choose_account_experience(
    current_user: User | None,
    connector: ConnectorTransaction,
    enterprise_sso: bool,
) -> dict[str, str]:
    """Choose the UI route; a login hint never authenticates or links a user."""
    login_hint = connector.login_hint
    if enterprise_sso:
        return {"action": "enterprise-sign-in", "returnTo": connector.id}
    if current_user and (not login_hint or current_user.email == login_hint):
        return {"action": "connector-consent", "userId": current_user.id}
    if current_user and login_hint:
        return {"action": "account-selector", "suggestedEmail": login_hint}
    if connector.target_flow != "chatgpt_siwc":
        return {"action": "application-sign-in", "returnTo": connector.id}
    return {"action": "sign-in-with-chatgpt", "returnTo": connector.id}
```

```go
func ChooseAccountExperience(currentUser *User, connector Transaction, enterpriseSSO bool) Experience {
	if enterpriseSSO {
		return Experience{Action: "enterprise-sign-in", ReturnTo: connector.ID}
	}
	if currentUser != nil && (connector.LoginHint == "" || currentUser.Email == connector.LoginHint) {
		return Experience{Action: "connector-consent", UserID: currentUser.ID}
	}
	if currentUser != nil && connector.LoginHint != "" {
		return Experience{Action: "account-selector", SuggestedEmail: connector.LoginHint}
	}
	if connector.TargetFlow != "chatgpt_siwc" {
		return Experience{Action: "application-sign-in", ReturnTo: connector.ID}
	}
	return Experience{Action: "sign-in-with-chatgpt", ReturnTo: connector.ID}
}
```

```java
public static Map<String, String> chooseAccountExperience(
    User currentUser, Transaction connector, boolean enterpriseSso) {
  String loginHint = connector.loginHint();
  if (enterpriseSso) {
    return Map.of("action", "enterprise-sign-in", "returnTo", connector.id());
  }
  if (currentUser != null && (!nonempty(loginHint) || currentUser.email().equals(loginHint))) {
    return Map.of("action", "connector-consent", "userId", currentUser.id());
  }
  if (currentUser != null && nonempty(loginHint)) {
    return Map.of("action", "account-selector", "suggestedEmail", loginHint);
  }
  if (!"chatgpt_siwc".equals(connector.targetFlow())) {
    return Map.of("action", "application-sign-in", "returnTo", connector.id());
  }
  return Map.of("action", "sign-in-with-chatgpt", "returnTo", connector.id());
}
```

```csharp
internal static Experience ChooseAccountExperience(User? currentUser,
    Transaction connector, bool enterpriseSso)
{
    if (enterpriseSso)
        return new("enterprise-sign-in", ReturnTo: connector.Id);
    if (currentUser is not null &&
        (string.IsNullOrEmpty(connector.LoginHint) || currentUser.Email == connector.LoginHint))
        return new("connector-consent", UserId: currentUser.Id);
    if (currentUser is not null && !string.IsNullOrEmpty(connector.LoginHint))
        return new("account-selector", SuggestedEmail: connector.LoginHint);
    if (connector.TargetFlow != "chatgpt_siwc")
        return new("application-sign-in", ReturnTo: connector.Id);
    return new("sign-in-with-chatgpt", ReturnTo: connector.Id);
}
```

```ruby
def choose_account_experience(current_user, connector, enterprise_sso)
  login_hint = connector[:login_hint]
  if enterprise_sso
    {
      action: "enterprise-sign-in",
      returnTo: connector[:id]
    }
  elsif current_user && (!nonempty?(login_hint) || current_user.fetch("email") == login_hint)
    {
      action: "connector-consent",
      userId: current_user.fetch("id")
    }
  elsif current_user && nonempty?(login_hint)
    {
      action: "account-selector",
      suggestedEmail: login_hint
    }
  elsif connector[:target_flow] != "chatgpt_siwc"
    {
      action: "application-sign-in",
      returnTo: connector[:id]
    }
  else
    {
      action: "sign-in-with-chatgpt",
      returnTo: connector[:id]
    }
  end
end
```


Use the returned action to show the account selector, start your existing
application or enterprise sign-in, show connector consent, or start Sign in with ChatGPT.
Label the sign-in action **Continue with ChatGPT**.

- If the active account and hint differ, let the person choose the account.
- Honor enterprise-managed domains, mandatory SSO, organization access, and account-provisioning rules.
- Avoid exposing whether an email address or enterprise domain has an account through publicly distinguishable errors.
- If you offer manual linking, explain how the person can sign in and connect the existing account.

## 4. Reuse website sign-in

Start the complete [website sign-in flow](https://developers.openai.com/siwc/website) with its own values:

- A new OpenAI sign-in `state`.
- A new PKCE verifier and S256 challenge.
- A new OpenAI nonce.
- Your registered OpenAI client ID and callback URI.
- The identity scopes `openid profile email`.

A client's registered scopes define the maximum that it may request; they aren't
granted automatically. Requesting `openid profile email` is an identity-only
authorization.

Verify the inner state, exchange the OpenAI code on your backend, and verify the
OpenAI ID token before identifying or linking the local account. Resume the
browser's saved connector transaction after establishing the local session.

## 5. Return your connector authorization code

After authenticating the user and completing your connector's consent and access
checks, issue an authorization code for your connector. The code must be
short-lived, single-use, and bound to the original connector request.

```typescript
export function completeConnectorAuthorization(
  transactionId: string,
  browserSessionId: string,
  authenticatedUser: { id: string } | null,
  consentGranted: boolean
) {
  const connector = transactions.get(transactionId);
  if (
    !connector ||
    connector.browserSessionId !== browserSessionId ||
    connector.expiresAt <= Date.now() ||
    !authenticatedUser
  ) {
    throw new Error("The connector transaction could not be verified.");
  }
  transactions.delete(transactionId);

  const callback = new URL(connector.redirectUri);
  callback.searchParams.set("state", connector.state);
  if (!consentGranted) {
    callback.searchParams.set("error", "access_denied");
    return callback.toString();
  }

  const connectorCode = randomValue();
  authorizationCodes.set(connectorCode, {
    userId: authenticatedUser.id,
    clientId: connector.clientId,
    redirectUri: connector.redirectUri,
    codeChallenge: connector.codeChallenge,
    scope: connector.scope,
    resource: connector.resource,
    expiresAt: Date.now() + 60 * 1000,
  });
  callback.searchParams.set("code", connectorCode);
  return callback.toString();
}
```

```python
def complete_connector_authorization(
    transaction_id: str,
    browser_session_id: str,
    authenticated_user: User | None,
    consent_granted: bool,
) -> str:
    """Resume only the transaction belonging to this verified browser session."""
    _required(browser_session_id, "browser session")
    if type(consent_granted) is not bool:
        raise ValueError("An explicit consent decision is required.")
    with _lock:
        now = time.time()
        _prune(now)
        connector = _transactions.get(transaction_id)
        if (
            connector is None
            or not secrets.compare_digest(
                connector.browser_session_id.encode(), browser_session_id.encode()
            )
            or authenticated_user is None
        ):
            raise ValueError("The connector transaction could not be verified.")
        _required(authenticated_user.id, "authenticated user")
        if consent_granted and len(_authorization_codes) >= _MAX_PENDING:
            raise ValueError("Too many pending connector authorization codes.")

        callback = urlsplit(connector.redirect_uri)
        params = dict(parse_qsl(callback.query, keep_blank_values=True))
        # A denial must not return a stale code from a registered query string.
        params.pop("code", None)
        params.pop("error", None)
        params["state"] = connector.state
        if not consent_granted:
            params["error"] = "access_denied"
        else:
            connector_code = secrets.token_urlsafe(32)
            _authorization_codes[connector_code] = ConnectorGrant(
                user_id=authenticated_user.id,
                client_id=connector.client_id,
                redirect_uri=connector.redirect_uri,
                code_challenge=connector.code_challenge,
                scope=connector.scope,
                resource=connector.resource,
                expires_at=now + 60,
            )
            params["code"] = connector_code
        del _transactions[transaction_id]
        return urlunsplit(callback._replace(query=urlencode(params)))
```

```go
func (c *Connector) CompleteConnectorAuthorization(
	transactionID, browserSessionID string,
	authenticatedUser *User,
	consentGranted bool,
) (string, error) {
	c.mu.Lock()
	defer c.mu.Unlock()
	now := time.Now()
	c.prune(now)
	connector, ok := c.transactions[transactionID]
	if !ok ||
		connector.BrowserSessionID != browserSessionID ||
		authenticatedUser == nil ||
		strings.TrimSpace(authenticatedUser.ID) == "" {
		return "", errors.New("the connector transaction could not be verified")
	}
	callback, err := url.Parse(connector.RedirectURI)
	if err != nil {
		return "", err
	}
	params := callback.Query()
	params.Set("state", connector.State)
	if !consentGranted {
		delete(c.transactions, transactionID)
		params.Set("error", "access_denied")
	} else {
		if len(c.codes) >= maxPending {
			return "", errors.New("connector code capacity reached")
		}
		code, err := randomValue()
		if err != nil {
			return "", err
		}
		c.codes[code] = Grant{UserID: authenticatedUser.ID, ClientID: connector.ClientID,
			RedirectURI: connector.RedirectURI, CodeChallenge: connector.CodeChallenge,
			Scope: connector.Scope, Resource: connector.Resource, ExpiresAt: now.Add(time.Minute)}
		delete(c.transactions, transactionID)
		params.Set("code", code)
	}
	callback.RawQuery = params.Encode()
	return callback.String(), nil
}
```

```java
public synchronized String completeConnectorAuthorization(
    String transactionId,
    String browserSessionId,
    User authenticatedUser,
    boolean consentGranted) {
  Transaction connector = transactions.get(transactionId);
  if (connector == null
      || !connector.browserSessionId().equals(browserSessionId)
      || connector.expiresAt() <= clock.millis()
      || authenticatedUser == null
      || !nonempty(authenticatedUser.id())) {
    throw new IllegalArgumentException("The connector transaction could not be verified.");
  }
  purgeExpired();
  if (consentGranted && authorizationCodes.size() >= MAX_PENDING) {
    throw new IllegalStateException("Too many pending authorization codes.");
  }
  transactions.remove(transactionId);
  Map<String, String> callback = new LinkedHashMap<>();
  callback.put("state", connector.state());
  if (!consentGranted) {
    callback.put("error", "access_denied");
  } else {
    String connectorCode = randomValue();
    authorizationCodes.put(
        connectorCode,
        new Grant(
            authenticatedUser.id(),
            connector.clientId(),
            connector.redirectUri(),
            connector.codeChallenge(),
            connector.scope(),
            connector.resource(),
            clock.millis() + 60 * 1000));
    callback.put("code", connectorCode);
  }
  String redirect = connector.redirectUri();
  int fragmentIndex = redirect.indexOf('#');
  String fragment = fragmentIndex < 0 ? "" : redirect.substring(fragmentIndex);
  String target = fragmentIndex < 0 ? redirect : redirect.substring(0, fragmentIndex);
  String separator = URI.create(target).getRawQuery() == null ? "?" : "&";
  return target + separator + encodeForm(callback) + fragment;
}
```

```csharp
internal static string CompleteConnectorAuthorization(string transactionId,
    string browserSessionId, User? authenticatedUser, bool consentGranted)
{
    lock (Gate)
    {
        PruneExpired();
        if (!Transactions.TryGetValue(transactionId, out Transaction? connector) ||
            connector.BrowserSessionId != browserSessionId ||
            authenticatedUser is null || string.IsNullOrWhiteSpace(authenticatedUser.Id))
        {
            throw new InvalidOperationException("The connector transaction could not be verified.");
        }
        if (consentGranted && AuthorizationCodes.Count >= Capacity)
            throw new InvalidOperationException("Too many pending connector codes.");
        Transactions.Remove(transactionId);
        UriBuilder callback = new(connector.RedirectUri);
        NameValueCollection query = HttpUtility.ParseQueryString(callback.Query);
        query["state"] = connector.State;
        if (!consentGranted)
        {
            query["error"] = "access_denied";
        }
        else
        {
            string code = RandomValue();
            AuthorizationCodes.Add(code, new(authenticatedUser.Id, connector.ClientId,
                connector.RedirectUri, connector.CodeChallenge, connector.Scope,
                connector.Resource, DateTimeOffset.UtcNow.AddSeconds(60)));
            query["code"] = code;
        }
        callback.Query = query.ToString() ?? "";
        return callback.Uri.AbsoluteUri;
    }
}
```

```ruby
def complete_connector_authorization(transaction_id, browser_session_id, authenticated_user, consent_granted)
  unless authenticated_user.is_a?(Hash) && nonempty?(authenticated_user["id"]) &&
         [true, false].include?(consent_granted)
    raise ArgumentError, "Authentication and an explicit consent decision are required."
  end

  @lock.synchronize do
    remove_expired
    connector = @transactions[transaction_id]
    unless connector && connector[:browser_session_id] == browser_session_id
      raise ArgumentError, "The connector transaction could not be verified."
    end
    if consent_granted && @authorization_codes.length >= MAX_ENTRIES
      raise "Too many connector authorization codes"
    end

    @transactions.delete(transaction_id)
    callback = URI.parse(connector[:redirect_uri])
    params = URI.decode_www_form(callback.query.to_s)
    params.reject! { |key, _| ["state", "code", "error"].include?(key) }
    params << ["state", connector[:state]]
    if consent_granted
      code = SecureRandom.urlsafe_base64(32, false)
      @authorization_codes[code] = {
        user_id: authenticated_user.fetch("id").dup.freeze,
        client_id: connector[:client_id],
        redirect_uri: connector[:redirect_uri],
        code_challenge: connector[:code_challenge],
        scope: connector[:scope],
        resource: connector[:resource],
        expires_at: Time.now.to_f + 60
      }.freeze
      params << ["code", code]
    else
      params << ["error", "access_denied"]
    end
    callback.query = URI.encode_www_form(params)
    callback.to_s
  end
end
```


Redirect the browser to the returned callback URL. Pass `authenticatedUser` only
from a verified application session and `consentGranted` only after your
connector's consent and access checks. In production, consume transactions and
codes atomically in shared storage.

If the user denies access or authorization fails, return an appropriate OAuth
error to the validated, registered connector callback with the original state.
Don't redirect to a callback until you've validated it.

## 6. Complete connector token exchange

ChatGPT redeems your connector authorization code at your connector token
endpoint. That endpoint must:

1. Validate the connector client.
2. Validate the authorization code, original connector redirect URI, and expiration.
3. Verify the outer `code_verifier` against the stored connector `code_challenge`.
4. Enforce the granted connector scopes and resource audience.
5. Issue only the application-owned connector tokens required for the integration.
6. Store and revoke connector credentials according to your existing security policy.

## Handle common integration problems

### The plugin opens the wrong account

Compare the hint to the current session for user experience only. Offer an
account selector instead of silently replacing a session or trusting an
unverified email.

### Enterprise SSO is required

Route the person through the existing enterprise identity provider. Resume the
saved outer request only after your application establishes a valid local
session.

### ChatGPT rejects the callback

Confirm that the redirect URI exactly matches the registered ChatGPT connector
callback. Return the original outer `state` and a connector authorization code.
Don't send an OpenAI code or ID token to ChatGPT as the connector result.

### PKCE verification fails

Check which transaction the verifier belongs to. The OpenAI verifier redeems the
inner OpenAI code; the ChatGPT verifier redeems your outer connector code.

### A returning user can't reconnect

Keep account-linking and reconnection paths available. If an existing account
must be linked manually, show actionable steps instead of an unexplained
conflict or generic error.

## Enable Continue with ChatGPT

After completing your implementation, contact your OpenAI representative on the
partnerships team to enable **Continue with ChatGPT** in your ChatGPT plugin's
modal.

## Test before launch

- Validate the connector's registered client, callback, scopes, and PKCE challenge.
- Confirm `target_flow=chatgpt_siwc` selects the expected experience.
- Test existing-account, switched-account, new-account, and enterprise SSO paths.
- Confirm the connector token endpoint verifies its own outer PKCE challenge.
- Keep the inner OpenAI code, ID token, and verifier out of the outer flow.
- Check that expired or reused connector transactions and authorization codes are rejected.
- Return denial errors only to the validated callback, with the original state.