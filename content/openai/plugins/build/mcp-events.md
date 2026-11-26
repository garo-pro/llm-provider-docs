# MCP Events

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

MCP Events lets ChatGPT subscribe to updates from your MCP server, such as new messages, content updates, or status changes. Users choose what to monitor and what ChatGPT should do when an update arrives.

| Use case                         | User request                                                                                 | MCP event                                    |
| -------------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------- |
| Turn feedback into pull requests | Monitor #product-feedback for bug reports and open draft pull requests with fixes and tests. | `message.created`, filtered by `channel_id`  |
| Apply document feedback          | Watch this document for review comments and implement any requested edits.                   | `comment.created`, filtered by `document_id` |

## Before you start

MCP Events is available in Work chats on ChatGPT web, in Work chats in the desktop app with **Cloud** selected, and with [dots](https://learn.chatgpt.com/docs/dots). Workspace controls for plugins and event-triggered tasks apply.

ChatGPT supports MCP Events over Streamable HTTP using `initialize`, or over MCP 2.0 HTTP (protocol version `2026-07-28`). Configure your server in your plugin and provide persistent subscription storage and outbound HTTPS access to callback URLs.

ChatGPT supports webhook delivery and callback verification from the [draft MCP Events specification](https://github.com/modelcontextprotocol/experimental-ext-triggers-events/blob/main/docs/design-sketch-proposal.md). Polling, streaming, and the draft’s `gap` and `terminated` control notifications are not supported by this integration.

## How it works

1. Your server lists the events it supports.
2. The user tells ChatGPT what to monitor and how to respond.
3. ChatGPT subscribes through your MCP server and supplies a callback URL and signing secret.
4. Your server sends matching events to that URL.
5. ChatGPT receives the event in the subscribed chat and follows the user’s instructions for how to respond.

## Advertise event support

Event discovery starts with your server’s capabilities. For MCP versions that use initialization, add `events` to the capabilities returned by the `initialize` response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-03-26",
    "capabilities": {
      "tools": {},
      "events": {}
    },
    "serverInfo": {
      "name": "example-server",
      "version": "1.0.0"
    }
  }
}
```

For MCP 2.0 (protocol version `2026-07-28`), add `events` to the capabilities returned by the `server/discover` response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "supportedVersions": ["2026-07-28"],
    "capabilities": {
      "tools": {},
      "events": {}
    }
  }
}
```

Implement these three event methods on the same authenticated MCP endpoint as your tools:

| Method               | Server behavior                              |
| -------------------- | -------------------------------------------- |
| `events/list`        | Describe available events and their filters. |
| `events/subscribe`   | Create or refresh a subscription.            |
| `events/unsubscribe` | Stop a subscription.                         |

## Define an event

An event definition tells ChatGPT what users can subscribe to and which filters are available. Return these definitions from `events/list`, including the event name, supported delivery modes, subscription arguments, and payload schema.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "events": [
      {
        "name": "comment.created",
        "description": "A new review comment was added to the specified document.",
        "delivery": ["webhook"],
        "inputSchema": {
          "type": "object",
          "properties": {
            "document_id": {
              "type": "string",
              "description": "ID of the document to monitor for new review comments."
            }
          },
          "required": ["document_id"],
          "additionalProperties": false
        },
        "payloadSchema": {
          "type": "object",
          "properties": {
            "document_id": { "type": "string" },
            "comment_id": { "type": "string" },
            "text": { "type": "string" },
            "url": { "type": "string" }
          },
          "required": ["document_id", "comment_id", "text", "url"],
          "additionalProperties": false
        }
      }
    ]
  }
}
```

`inputSchema` describes the arguments passed when ChatGPT subscribes, while `payloadSchema` describes the `data` object in each delivered event.

Use stable event names and specific descriptions. Expose filters such as document, project, or queue IDs, and apply them on your server before delivery. Return only events the connected account is allowed to discover.

If the catalog spans multiple pages, return `nextCursor` and accept it as `cursor` on the next `events/list` request.

## Create a subscription

When a user asks to monitor an event, ChatGPT calls `events/subscribe` with the event name, filter arguments, and webhook destination:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "events/subscribe",
  "params": {
    "name": "comment.created",
    "arguments": {
      "document_id": "doc_123"
    },
    "delivery": {
      "mode": "webhook",
      "url": "https://receiver.example.com/mcp-events/callback_123",
      "secret": "whsec_<base64-encoded-signing-key>"
    },
    "cursor": null
  }
}
```

Before accepting the subscription:

1. Check that the user is authorized for the requested event and arguments.
2. Validate the event name and arguments against your event definition. Require a `whsec_` signing secret whose base64 value decodes to 24–64 bytes.
3. [Validate and verify the callback URL](#verify-the-callback).
4. Store the subscription, its owner, filters, callback URL, signing secret, and expiration.

Derive a deterministic subscription ID from the authenticated principal, callback URL, event name, and arguments. Return it with the granted expiration:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "id": "sub_123",
    "refreshBefore": "2026-10-02T12:00:00Z",
    "cursor": null,
    "truncated": false
  }
}
```

Set `refreshBefore` to the expiration your server grants. Return `cursor: null` for event types that do not support replay.

Make subscription creation idempotent: update the existing subscription when its identity matches. Compare arguments using canonical JSON so object key order does not create duplicate subscriptions.

### Verify the callback

Before sending application data, verify the callback by sending a signed request with a fresh, single-use, short-lived challenge:

```json
{
  "type": "verification",
  "challenge": "a-single-use-random-value"
}
```

Assign the verification request a unique `webhook-id`, such as `msg_verification_123`, and sign its body with the subscription’s secret. Include `webhook-timestamp`, `webhook-signature`, and `X-MCP-Subscription-Id`. ChatGPT echoes the challenge in a successful response:

```json
{
  "challenge": "a-single-use-random-value"
}
```

Require a `2xx` response and compare the returned challenge in constant time before activating delivery. Cache successful verification by authenticated principal and callback URL for a bounded period so repeated subscription requests do not trigger repeated challenges. If verification fails, return JSON-RPC error `-32015` (`CallbackEndpointError`) with a categorized `data.reason`, such as `challenge_failed` or `timeout`.

Require HTTPS for callbacks. Resolve and validate destination addresses at connection time, then connect to the validated address while preserving the original hostname for TLS verification. Block private, local, and other non-public addresses, and do not follow redirects. Apply these checks to verification requests as well as event deliveries.

## Send an event

When a new comment matches an active subscription, POST one event object to that subscription's callback URL:

```json
{
  "eventId": "evt_456",
  "name": "comment.created",
  "timestamp": "2026-10-01T12:05:00Z",
  "data": {
    "document_id": "doc_123",
    "comment_id": "comment_456",
    "text": "Can we add the rollout dates to this section?",
    "url": "https://docs.example.com/doc_123#comment_456"
  },
  "cursor": null
}
```

Use a unique event ID and preserve it across retries. Set `timestamp` to the event’s occurrence time as an ISO 8601 timestamp with a timezone. The `name` must match the subscribed event, and the `data` object must match its `payloadSchema`. Keep application fields inside `data`; a top-level `type` identifies a protocol control notification.

For large records, send a summary and expose a read tool to retrieve the full record. Treat comments and other user-authored text as data; do not add instructions telling the model how to behave inside the event payload.

### Sign the request

ChatGPT verifies deliveries using Standard Webhooks. Send these headers:

| Header                  | Value                                   |
| ----------------------- | --------------------------------------- |
| `Content-Type`          | `application/json`                      |
| `webhook-id`            | The same value as the body's `eventId`. |
| `webhook-timestamp`     | The signing time as Unix seconds.       |
| `webhook-signature`     | The Standard Webhooks HMAC signature.   |
| `X-MCP-Subscription-Id` | The ID returned by `events/subscribe`.  |

Sign deliveries using Standard Webhooks and the subscription’s signing secret. Because the signature covers the event ID, signing timestamp, and exact request body bytes, serialize the body once and send those same bytes.

### Send a signed event with Node.js

Install the [Standard Webhooks library](https://github.com/standard-webhooks/standard-webhooks/tree/main/libraries/javascript):

```bash
npm install standardwebhooks
```

Pass a `webhookFetch` function with the same interface as `fetch` that validates callback addresses on each connection and blocks redirects.

Send a signed event

```typescript
import { Webhook } from "standardwebhooks";

export async function sendEvent(
  subscription: { id: string; url: string; secret: string },
  event: { eventId: string },
  webhookFetch: typeof fetch
) {
  const body = JSON.stringify(event);
  if (Buffer.byteLength(body, "utf8") > 256 * 1024) {
    throw new Error("Event payload exceeds 256 KiB");
  }

  const signedAt = new Date();
  const signer = new Webhook(subscription.secret);
  const response = await webhookFetch(subscription.url, {
    method: "POST",
    redirect: "error",
    signal: AbortSignal.timeout(10_000),
    headers: {
      "Content-Type": "application/json",
      "webhook-id": event.eventId,
      "webhook-timestamp": String(Math.floor(signedAt.getTime() / 1000)),
      "webhook-signature": signer.sign(event.eventId, signedAt, body),
      "X-MCP-Subscription-Id": subscription.id,
    },
    body,
  });

  return { accepted: response.ok, status: response.status };
}
```

```python
import base64
import hashlib
import hmac
import json
import time
from collections.abc import Callable
from dataclasses import dataclass
from urllib.parse import urlsplit


@dataclass(frozen=True)
class Subscription:
    id: str
    url: str
    secret: str


def encode_event(event: dict[str, object]) -> bytes:
    body = json.dumps(
        event, ensure_ascii=False, separators=(",", ":"), allow_nan=False
    ).encode("utf-8")
    if len(body) > 256 * 1024:
        raise ValueError("Event payload exceeds 256 KiB")
    return body


def prepare_event(
    subscription: Subscription, event: dict[str, object]
) -> tuple[bytes, dict[str, str]]:
    event_id = event.get("eventId")
    if not isinstance(event_id, str) or not event_id or not subscription.id:
        raise ValueError(
            "Event ID must be a nonempty string and subscription ID must not be empty"
        )
    body = encode_event(event)
    timestamp = str(int(time.time()))
    secret = subscription.secret.removeprefix("whsec_")
    key = base64.b64decode(secret, validate=True)
    if not key:
        raise ValueError("Webhook signing key must not be empty")
    signed = f"{event_id}.{timestamp}.".encode() + body
    signature = base64.b64encode(hmac.digest(key, signed, hashlib.sha256)).decode(
        "ascii"
    )
    return body, {
        "Content-Type": "application/json",
        "webhook-id": event_id,
        "webhook-timestamp": timestamp,
        "webhook-signature": f"v1,{signature}",
        "X-MCP-Subscription-Id": subscription.id,
    }


# The caller supplies a transport that validates public destination addresses on
# every connection, preserves TLS hostname verification, and refuses redirects.
# No default transport can safely make this decision for arbitrary subscribers.
ValidatedTransport = Callable[[str, bytes, dict[str, str]], int]


def send_event(
    subscription: Subscription,
    event: dict[str, object],
    *,
    validated_transport: ValidatedTransport,
) -> dict[str, object]:
    target = urlsplit(subscription.url)
    if target.scheme != "https" or not target.hostname:
        raise ValueError("Subscription URL must be an absolute HTTPS URL")
    if target.username is not None or target.password is not None or target.fragment:
        raise ValueError("Subscription URL must not contain credentials or a fragment")
    body, headers = prepare_event(subscription, event)
    status = validated_transport(subscription.url, body, headers)
    if 300 <= status < 400:
        raise RuntimeError("Webhook redirects are not allowed")
    return {"accepted": 200 <= status < 300, "status": status}
```

```go
package main

import (
	"bytes"
	"context"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net"
	"net/http"
	"net/netip"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

type Subscription struct{ ID, URL, Secret string }
type DeliveryResult struct {
	Accepted bool `json:"accepted"`
	Status   int  `json:"status"`
}

func sendEvent(ctx context.Context, subscription Subscription, event map[string]any) (DeliveryResult, error) {
	body, err := json.Marshal(event)
	if err != nil {
		return DeliveryResult{}, err
	}
	if len(body) > 256*1024 {
		return DeliveryResult{}, errors.New("Event payload exceeds 256 KiB")
	}
	id, ok := event["eventId"].(string)
	if !ok || id == "" {
		return DeliveryResult{}, errors.New("Missing eventId")
	}
	target, err := url.Parse(subscription.URL)
	if err != nil || target.Host == "" || target.Scheme != "https" || target.User != nil || target.Fragment != "" {
		return DeliveryResult{}, errors.New("Expected an HTTPS subscription URL without credentials or a fragment")
	}
	key, err := base64.StdEncoding.Strict().DecodeString(strings.TrimPrefix(subscription.Secret, "whsec_"))
	if err != nil || len(key) == 0 {
		return DeliveryResult{}, errors.New("Invalid webhook secret")
	}
	timestamp := strconv.FormatInt(time.Now().Unix(), 10)
	mac := hmac.New(sha256.New, key)
	mac.Write([]byte(id + "." + timestamp + "."))
	mac.Write(body)
	request, err := http.NewRequestWithContext(ctx, http.MethodPost, target.String(), bytes.NewReader(body))
	if err != nil {
		return DeliveryResult{}, err
	}
	request.Header.Set("Content-Type", "application/json")
	request.Header.Set("webhook-id", id)
	request.Header.Set("webhook-timestamp", timestamp)
	request.Header.Set("webhook-signature", "v1,"+base64.StdEncoding.EncodeToString(mac.Sum(nil)))
	request.Header.Set("X-MCP-Subscription-Id", subscription.ID)
	response, err := webhookClient.Do(request)
	if err != nil {
		return DeliveryResult{}, err
	}
	defer response.Body.Close()
	if response.StatusCode >= 300 && response.StatusCode < 400 {
		return DeliveryResult{}, errors.New("Webhook redirects are not allowed")
	}
	return DeliveryResult{Accepted: response.StatusCode >= 200 && response.StatusCode < 300, Status: response.StatusCode}, nil
}

// Resolve and validate at connection time, then dial that exact IP. TLS still
// verifies the original URL hostname. Proxies and redirects are not permitted.
func publicAddress(ip netip.Addr) bool {
	ip = ip.Unmap()
	if !ip.IsGlobalUnicast() || ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() {
		return false
	}
	for _, cidr := range []string{"0.0.0.0/8", "100.64.0.0/10", "192.0.0.0/24", "192.0.2.0/24", "192.88.99.0/24", "198.18.0.0/15", "198.51.100.0/24", "203.0.113.0/24", "240.0.0.0/4", "2001::/23", "2001:db8::/32", "2002::/16", "3fff::/20"} {
		if netip.MustParsePrefix(cidr).Contains(ip) {
			return false
		}
	}
	return ip.Is4() || netip.MustParsePrefix("2000::/3").Contains(ip)
}

type lookupIPs func(context.Context, string, string) ([]netip.Addr, error)
type dialIP func(context.Context, string, string) (net.Conn, error)

func validatedDial(lookup lookupIPs, dial dialIP) dialIP {
	return func(ctx context.Context, network, address string) (net.Conn, error) {
		host, port, err := net.SplitHostPort(address)
		if err != nil {
			return nil, err
		}
		ips, err := lookup(ctx, "ip", host)
		if err != nil {
			return nil, err
		}
		if len(ips) == 0 {
			return nil, errors.New("Webhook hostname has no addresses")
		}
		for _, ip := range ips {
			if !publicAddress(ip) {
				return nil, errors.New("Webhook destination is not public")
			}
		}
		// Do not resolve the hostname again in the dialer (DNS rebinding).
		var failures []error
		for _, ip := range ips {
			connection, err := dial(ctx, network, net.JoinHostPort(ip.String(), port))
			if err == nil {
				return connection, nil
			}
			failures = append(failures, err)
			if ctx.Err() != nil {
				break
			}
		}
		return nil, errors.Join(failures...)
	}
}

var webhookClient = &http.Client{
	Timeout:       10 * time.Second,
	CheckRedirect: func(*http.Request, []*http.Request) error { return http.ErrUseLastResponse },
	Transport: &http.Transport{
		Proxy:               nil,
		DialContext:         validatedDial(net.DefaultResolver.LookupNetIP, (&net.Dialer{Timeout: 5 * time.Second}).DialContext),
		TLSHandshakeTimeout: 5 * time.Second,
	},
}

// Supply one JSON object on standard input with subscription (ID, URL, Secret)
// and event fields. Keep the secret out of command-line arguments and logs.
func run() error {
	var input struct {
		Subscription Subscription   `json:"subscription"`
		Event        map[string]any `json:"event"`
	}
	if err := json.NewDecoder(io.LimitReader(os.Stdin, 2*1024*1024)).Decode(&input); err != nil {
		return err
	}
	result, err := sendEvent(context.Background(), input.Subscription, input.Event)
	if err != nil {
		return err
	}
	return json.NewEncoder(os.Stdout).Encode(result)
}
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.net.URI;
import java.net.http.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.time.*;
import java.util.*;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

record Subscription(String id, URI url, String secret) {}

static Map<String, Object> sendEvent(Subscription subscription, Map<String, String> event)
    throws Exception {
  byte[] body = new ObjectMapper().writeValueAsBytes(event);
  if (body.length > 256 * 1024)
    throw new IllegalArgumentException("Event payload exceeds 256 KiB");
  String timestamp = Long.toString(Instant.now().getEpochSecond());
  String eventId = Objects.requireNonNull(event.get("eventId"));
  String encodedSecret = subscription.secret().replaceFirst("^whsec_", "");
  var signer = Mac.getInstance("HmacSHA256");
  signer.init(new SecretKeySpec(Base64.getDecoder().decode(encodedSecret), "HmacSHA256"));
  signer.update((eventId + "." + timestamp + ".").getBytes(StandardCharsets.UTF_8));
  String signature = "v1," + Base64.getEncoder().encodeToString(signer.doFinal(body));
  var client = HttpClient.newBuilder().followRedirects(HttpClient.Redirect.NEVER).build();
  var request =
      HttpRequest.newBuilder(subscription.url())
          .timeout(Duration.ofSeconds(10))
          .header("Content-Type", "application/json")
          .header("webhook-id", eventId)
          .header("webhook-timestamp", timestamp)
          .header("webhook-signature", signature)
          .header("X-MCP-Subscription-Id", subscription.id())
          .POST(HttpRequest.BodyPublishers.ofByteArray(body))
          .build();
  var response = client.send(request, HttpResponse.BodyHandlers.discarding());
  if (response.statusCode() >= 300 && response.statusCode() < 400)
    throw new java.io.IOException("Webhook redirects are not allowed");
  return Map.of(
      "accepted",
      response.statusCode() >= 200 && response.statusCode() < 300,
      "status",
      response.statusCode());
}
```

```csharp
using System.Globalization;
using System.Net.Http.Headers;
using System.Security.Cryptography;
using System.Text;
using System.Text.Encodings.Web;
using System.Text.Json;
using System.Text.Json.Serialization;

internal sealed record Subscription(string Id, Uri Url, string Secret);
internal sealed record DeliveryResult(
    [property: JsonPropertyName("accepted")] bool Accepted,
    [property: JsonPropertyName("status")] int Status);

// Supply an application-owned transport that requires HTTPS and validates public
// destination addresses at every connection, including DNS changes, while preserving
// TLS hostname verification. Never follow redirects or use an unrestricted client.
// Honor the cancellation token and complete after response headers. The sender owns
// and disposes the request and returned response. The caller owns the transport.
internal delegate Task<HttpResponseMessage> WebhookTransport(
    HttpRequestMessage request, CancellationToken cancellationToken);

private static readonly JsonSerializerOptions JsonOptions = new()
{
    Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping
};

internal static byte[] EncodeEvent(IReadOnlyDictionary<string, object?> eventData)
{
    byte[] body = JsonSerializer.SerializeToUtf8Bytes(eventData, JsonOptions);
    if (body.Length > 256 * 1024)
        throw new ArgumentException("Event payload exceeds 256 KiB");
    return body;
}

internal static async Task<DeliveryResult> SendEventAsync(
    Subscription subscription, IReadOnlyDictionary<string, object?> eventData,
    WebhookTransport validatedTransport, CancellationToken cancellationToken = default)
{
    ArgumentNullException.ThrowIfNull(validatedTransport);
    byte[] body = EncodeEvent(eventData);
    if (!eventData.TryGetValue("eventId", out object? identity) || identity is not string eventId)
        throw new ArgumentException("Event eventId must be a nonempty string.", nameof(eventData));
    ArgumentException.ThrowIfNullOrWhiteSpace(eventId);
    ArgumentException.ThrowIfNullOrWhiteSpace(subscription.Id);
    if (!subscription.Url.IsAbsoluteUri
        || subscription.Url.Scheme != "https"
        || subscription.Url.UserInfo.Length != 0 || subscription.Url.Fragment.Length != 0)
        throw new ArgumentException("Subscription URL must be an absolute HTTPS URL without credentials or a fragment");
    string encodedSecret = subscription.Secret.StartsWith("whsec_", StringComparison.Ordinal)
        ? subscription.Secret[6..] : subscription.Secret;
    byte[] key = Convert.FromBase64String(encodedSecret);
    if (key.Length == 0)
        throw new ArgumentException("Webhook signing key must not be empty");
    string timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(CultureInfo.InvariantCulture);
    byte[] prefix = Encoding.UTF8.GetBytes($"{eventId}.{timestamp}.");
    byte[] signature;
    try
    {
        using IncrementalHash signer = IncrementalHash.CreateHMAC(HashAlgorithmName.SHA256, key);
        signer.AppendData(prefix);
        signer.AppendData(body);
        signature = signer.GetHashAndReset();
    }
    finally
    {
        CryptographicOperations.ZeroMemory(key);
    }

    using HttpRequestMessage request = new(HttpMethod.Post, subscription.Url);
    request.Content = new ByteArrayContent(body);
    request.Content.Headers.ContentType = new MediaTypeHeaderValue("application/json");
    request.Headers.Add("webhook-id", eventId);
    request.Headers.Add("webhook-timestamp", timestamp);
    request.Headers.Add("webhook-signature", $"v1,{Convert.ToBase64String(signature)}");
    request.Headers.Add("X-MCP-Subscription-Id", subscription.Id);
    // Headers-only completion avoids buffering an untrusted response body.
    using CancellationTokenSource timeout = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
    timeout.CancelAfter(TimeSpan.FromSeconds(10));
    using HttpResponseMessage response = await validatedTransport(request, timeout.Token);
    int status = (int)response.StatusCode;
    if (status is >= 300 and < 400)
        throw new InvalidOperationException("Webhook redirects are not allowed");
    return new DeliveryResult(status is >= 200 and < 300, status);
}
```

```ruby
require "standardwebhooks"
require "net/http"
require "json"
require "uri"

def send_event(subscription, event)
  body = JSON.generate(event)
  raise ArgumentError, "Event payload exceeds 256 KiB" if body.bytesize > 256 * 1024

  signed_at = Time.now
  event_id = event.fetch(:eventId)
  signer = StandardWebhooks::Webhook.new(subscription.fetch(:secret))
  uri = URI.parse(subscription.fetch(:url))
  raise ArgumentError, "Expected an HTTP or HTTPS subscription URL" unless uri.is_a?(URI::HTTP)

  request = Net::HTTP::Post.new(uri)
  request["Content-Type"] = "application/json"
  request["webhook-id"] = event_id
  request["webhook-timestamp"] = signed_at.to_i.to_s
  request["webhook-signature"] = signer.sign(event_id, signed_at.to_i, body)
  request["X-MCP-Subscription-Id"] = subscription.fetch(:id)
  request.body = body
  response = Net::HTTP.start(
    uri.host, uri.port, use_ssl: uri.scheme == "https",
    open_timeout: 10, read_timeout: 10, write_timeout: 10
  ) do |http|
    http.request(request)
  end
  raise "Webhook redirects are not allowed" if response.is_a?(Net::HTTPRedirection)

  {
    accepted: response.is_a?(Net::HTTPSuccess),
    status: response.code.to_i
  }
end
```


Set `subscription.url` and `subscription.secret` from the subscription request’s `delivery` object.

### Handle delivery responses

A `2xx` response acknowledges webhook receipt. ChatGPT processes the event asynchronously.

Send one event per request, with a complete request body no larger than 256 KiB (262,144 bytes). ChatGPT can group separately delivered events into one task run according to the task’s batching settings.

Retry transient failures with exponential backoff and bounded attempts. Preserve the event ID and generate a fresh signing timestamp and signature for each attempt. Do not retry deliveries that return `410` or `413`.

Events can arrive out of order. Make write tools idempotent so repeated calls do not duplicate changes.

## Manage subscriptions

Retain subscription state for the lifetime you grant, including across server restarts. Recheck the user’s access during the subscription’s lifetime and stop delivery if access is revoked.

### Refresh a subscription

ChatGPT refreshes expiring subscriptions by calling `events/subscribe` before `refreshBefore`, using the same subscription identity and the last saved cursor. Update the existing subscription and return its new expiration in `refreshBefore`.

When `ttlMs` is omitted, use your server’s default subscription lifetime. When provided, it specifies the requested lifetime in milliseconds. Grant no more than that duration, except when enforcing a minimum lifetime to prevent excessive refresh requests.

`ttlMs: null` requests a subscription without expiration. Return `refreshBefore: null` only when granting that request. Otherwise, return a finite expiration and stop delivery when it passes.

When a refresh supplies a replacement signing secret, replace the stored secret. During a short rotation window, sign with both the old and new keys using Standard Webhooks’ space-separated signatures.

For replayable events, use the request’s `cursor` to resume after an expired subscription or server restart. In subscription responses and event payloads, return a cursor that does not skip events still awaiting delivery. Return `truncated: true` when the requested history is no longer available. For event types without replay, return `cursor: null`; events missed during an interruption cannot be recovered through the protocol.

### Stop a subscription

Handle `events/unsubscribe` using the original event name, arguments, and callback URL:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "events/unsubscribe",
  "params": {
    "name": "comment.created",
    "arguments": {
      "document_id": "doc_123"
    },
    "delivery": {
      "mode": "webhook",
      "url": "https://receiver.example.com/mcp-events/callback_123"
    }
  }
}
```

Stop sending events for the matching subscription and return an empty result:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {}
}
```

Make unsubscribe idempotent and authorize it against the connected account.

## Test in ChatGPT

With the event methods and webhook delivery in place, connect your MCP server to ChatGPT through a plugin and test the full subscription lifecycle:

1. Confirm your server advertises `events` through `initialize` or `server/discover`, then receives `events/list` and returns the expected event definitions.
2. Verify that your events appear on your plugin page alongside your tools. Rescan your MCP server whenever you change its tools or events.

   <img src="https://cdn.openai.com/devhub/docs/plugins/mcp-events/plugin-tools-and-events.webp"
     alt="Plugin details showing message.created and comment.created events alongside MCP tools."
     width="612"
     height="738"
     className="my-6 h-auto w-full max-w-[459px] rounded-xl"
     loading="lazy"
   />

   **_Plugin events._** _Discovered events appear alongside tools on the plugin page._

3. Start a Work chat on ChatGPT web, or select **Work** and **Cloud** in the desktop app. Ask ChatGPT to subscribe to one of your events and specify what it should do when an event arrives. You can also test with a [dot](https://learn.chatgpt.com/docs/dots).
4. Confirm your server receives `events/subscribe` with the expected event name and arguments.
5. Check that callback verification succeeds and the subscription is stored.
6. Trigger a matching event in your app and confirm that the webhook delivery receives a `2xx` response.
7. Verify that ChatGPT receives the expected event data and responds as instructed.
8. For filtered subscriptions, trigger an event that does not match the filters and confirm it is not delivered to the subscription.
9. Stop monitoring in ChatGPT. Confirm that your server processes `events/unsubscribe` and stops delivery.

Also test repeated subscription requests, expiration and refresh across a server restart, account disconnection, revoked access to a subscribed resource, invalid signatures, duplicate deliveries, and bursts with batching enabled and disabled. If the requested action changes data in the source app, verify that the resulting events do not create a feedback loop.

Protocol reference: [MCP Events design sketch](https://github.com/modelcontextprotocol/experimental-ext-triggers-events/blob/main/docs/design-sketch-proposal.md).