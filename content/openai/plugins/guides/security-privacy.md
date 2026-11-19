# Security & Privacy

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

## Principles

Plugin tools can access user data, third-party APIs, and write actions. Treat
every MCP server and UI component as production software:

- **Least privilege:** Only request the scopes, storage access, and network permissions you need.
- **Explicit user consent:** Make sure users understand when they are linking
  accounts or granting write access. Use the host's confirmation prompts for
  destructive actions.
- **Defense in depth:** Assume prompt injection and malicious inputs will reach your server. Check every input and keep audit logs.

## Data handling

- **Structured content:** Include only the data required for the current prompt. Avoid embedding secrets or tokens in component props.
- **Storage:** Decide how long you keep user data and publish a retention policy. Respect deletion requests.
- **Logging:** Redact PII before writing to logs. Store correlation IDs for debugging but avoid storing raw prompt text unless necessary.

### Input schemas and task context

- Design the input schema to limit data collection by default, rather than a funnel for optional context.

- Input fields must be directly related to the tool’s stated purpose.

- Do not request the full conversation history, raw chat transcripts, or broad contextual fields “just in case.” A tool may request a brief, task-specific user intent field only when it meaningfully improves execution and does not expand data collection beyond what is reasonably necessary to respond to the user’s request and for the purposes described in your privacy policy.

- Do not request accumulated conversation context or prior-turn arrays.

### Response payloads

Tool responses must return only data that is directly relevant to the user’s request and the tool’s stated purpose. Do not include diagnostic, telemetry, or internal identifiers—such as session IDs, trace IDs, request IDs, timestamps, or logging metadata—unless they are strictly required to fulfill the user’s query.

### Chat-context boundaries

Your MCP server must not pull, reconstruct, or infer the full chat log from the client or elsewhere. Operate only on the explicit snippets and resources the client or model chooses to send. This separation can help prevent covert data expansion and keep analysis limited to intentionally shared content.

### Location

If needed, rely on the coarse geographic location shared by the system. Do not request precise user location data (for example, GPS coordinates or addresses).

Avoid requesting raw location fields (for example, city or coordinates) in your input schema. When location is needed, obtain it through the client’s controlled side channel (such as environment metadata or a referenced resource) so appropriate policy and consent controls can be applied. This reduces accidental PII capture, enforces least-privilege access, and keeps location handling auditable and revocable.

## Prompt injection and write actions

Custom MCP servers enable full MCP access, including write tools. Mitigate risk by:

- Reviewing tool descriptions regularly to discourage misuse (“Do not use to delete records”).
- Validating all inputs server-side even if the model provided them.
- Requiring human confirmation for irreversible operations.

Share your best prompts for testing injections with your QA team so they can probe weak spots early.

## Network access

Widgets run inside an isolated iframe with a strict Content Security Policy.
They cannot access privileged browser APIs such as `window.alert`,
`window.prompt`, `window.confirm`, or `navigator.clipboard`. The CSP controls
standard `fetch` requests. Nested frames are unavailable by default; enable
specific origins in resource CSP metadata such as
`_meta.ui.csp.frameDomains`. Plugins can embed pages from their MCP server's
own registrable domain, including existing editors and admin interfaces. See
the [iframe requirements](https://developers.openai.com/plugins/build/chatgpt-ui#content-security-policy-csp) for
domain ownership, required justifications, and review requirements.

The widget CSP restricts which iframe destinations can load. An embedded
page uses its own CSP; the widget `connectDomains` and `resourceDomains`
allowlists do not restrict network requests made inside that page. Keep iframe
origins specific and include the embedded experience in your security review.

Server-side code has no network restrictions beyond what your hosting environment enforces. Follow normal best practices for outbound calls (TLS verification, retries, timeouts).

## Authentication & authorization

- Use OAuth 2.1 authorization-code flows when integrating external accounts.
  Prefer Client ID Metadata Documents (CIMD) when your authorization server
  supports CIMD and the plugin builder chooses it. Use `none` for public-client
  token exchange or `private_key_jwt` when your authorization server requires
  client authentication. Support DCR when the plugin builder chooses it or CIMD
  is not available.
- Verify and enforce scopes on every tool call. Return a `401` response for expired or malformed tokens.
- For built-in identity, avoid storing long-lived secrets; use the provided auth context instead.

## Operational readiness

- Run security reviews before launch, especially if you handle regulated data.
- Monitor for anomalous traffic patterns and set up alerts for repeated errors or failed auth attempts.
- Keep third-party dependencies, libraries, and build tooling patched to mitigate supply chain risks.

Security and privacy are foundational to user trust. Bake them into your planning, implementation, and deployment workflows rather than treating them as an afterthought.