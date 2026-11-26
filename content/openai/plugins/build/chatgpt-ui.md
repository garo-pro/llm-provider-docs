# Add UI to your MCP server

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

## Overview

Custom UI is optional. Add it when a plugin use case requires people to
inspect, compare, edit, confirm, or navigate structured information. Keep the
MCP tools useful without a component so ChatGPT and Codex can complete the
workflow without UI.

The MCP server returns UI resources for selected tools. Components run inside
an iframe in ChatGPT, communicate with the host through the MCP Apps bridge
(JSON-RPC over `postMessage`), and render alongside the conversation. The open
MCP Apps standard lets the UI run across compatible hosts.

## Start with MCP Apps

ChatGPT implements the open [MCP Apps
standard](https://modelcontextprotocol.io/docs/extensions/apps) for UI returned
by an MCP server. MCP Apps defines how your server associates tools with UI
resources and how the iframe communicates with its host.

For new UI:

1. Declare the UI resource with `_meta.ui.resourceUri`.
2. Use the `ui/*` JSON-RPC bridge over `postMessage` for initialization,
   notifications, tool calls, messages, and model-visible context.
3. Keep tools useful without UI so the model can complete the workflow in
   clients that do not render components.

This standards-first foundation lets the same UI run in ChatGPT and other
compatible MCP Apps hosts.

When you're ready to implement the standard, use the [MCP Apps
specification](https://modelcontextprotocol.io/docs/extensions/apps).

For sidebar apps, file viewers, and richer forms, see
[Extensions](https://developers.openai.com/plugins/build/extensions).

## Layer on ChatGPT extensions

After the MCP Apps flow works, use `window.openai` only for capabilities that
the shared specification does not cover. These optional extensions can improve
the experience in ChatGPT without making them part of the portable UI
foundation.

### Prefer shared fields and methods

Use the MCP Apps field or method whenever the shared specification covers the
capability:

| Goal                         | MCP Apps standard                               | ChatGPT compatibility alias         |
| ---------------------------- | ----------------------------------------------- | ----------------------------------- |
| Link a tool to a UI resource | `_meta.ui.resourceUri`                          | `_meta["openai/outputTemplate"]`    |
| Receive tool input           | `ui/initialize` + `ui/notifications/tool-input` | `window.openai.toolInput`           |
| Receive tool results         | `ui/notifications/tool-result`                  | `window.openai.toolOutput`          |
| Call a tool from the UI      | `tools/call`                                    | `window.openai.callTool`            |
| Send a follow-up message     | `ui/message`                                    | `window.openai.sendFollowUpMessage` |

The compatibility aliases remain available for existing integrations. New UI
should use the shared fields and bridge methods in the middle column.

Examples include:

- Instant Checkout with `window.openai.requestCheckout`.
- ChatGPT file handling with `window.openai.uploadFile`,
  `window.openai.selectFiles`, and `window.openai.getFileDownloadUrl`.
- Host-controlled modals with `window.openai.requestModal`.
- Widget-state persistence with `window.openai.widgetState` and
  `window.openai.setWidgetState`.

Feature-detect each extension and provide a fallback when practical:

```typescript
const openai = typeof window !== "undefined" ? window.openai : undefined;

if (openai?.requestModal) {
  await openai.requestModal({
    template: "ui://widget/details.html",
  });
} else {
  document.body.textContent = "This host does not support modals.";
}
```


Avoid branching on a host or product name. Test for the capability your UI
needs.

For extension signatures and examples, see the [`window.openai` component
bridge reference](https://developers.openai.com/plugins/reference#windowopenai-component-bridge).

### Optional OpenAI component library

The
[`@openai/apps-sdk-ui`](https://openai.github.io/apps-sdk-ui/) component
library provides ready-made buttons, cards, input controls, and layout
primitives that match ChatGPT's container. Use it when you want consistent
styling without rebuilding base components.

You can also explore the [UI examples repository on
GitHub](https://github.com/openai/openai-apps-sdk-examples).

## Choose a presentation

Start with inline UI and request more space only when the workflow needs it.
Choose the smallest presentation that lets people understand the result or
complete the task.

### Inline card

Use an inline card for a focused result, confirmation, or small set of actions.
Keep it self-contained and avoid deep navigation.

![Examples of inline cards](https://developers.openai.com/images/apps-sdk/inline_cards.png)

### Inline carousel

Use an inline carousel when people need to scan and choose from a small set of
similar, visually rich options.

![Example of an inline carousel](https://developers.openai.com/images/apps-sdk/inline_carousel.png)

### fullscreen

Use fullscreen for rich tasks that need more room, such as maps, editing
canvases, or detailed browsing. Design the experience to work with ChatGPT's
composer, which remains available in fullscreen.

![Example of fullscreen UI](https://developers.openai.com/images/apps-sdk/fullscreen.png)

### Picture-in-picture

Use picture-in-picture for an ongoing activity that should remain visible while
the conversation continues, such as a live session, game, or video.

![Example of picture-in-picture UI](https://developers.openai.com/images/apps-sdk/pip.png)

For detailed layout, interaction, visual design, and accessibility guidance,
see [UI guidelines](https://developers.openai.com/plugins/concepts/ui-guidelines).

## Separate data processing from UI rendering

### Decoupled pattern

If you attach a widget template to every tool call, ChatGPT can re-render your
iframe too often. A better pattern is to separate data-processing tools from
render tools:

- **Data tools** fetch, compute, or mutate data and return only tool results.
- **Render tools** take final data and return the widget template.

This allows the model to apply its intelligence to data it fetched before
choosing to render UI to the user, making it much more likely that it will
accomplish the user's specific expressed goal.

This pattern is part of the MCP Apps architecture.

In practice, many UI integrations use this split:

- **Search/fetch tools (data-first):** Return IDs plus metadata with no widget
  template attached.
- **Render tools (for example, `render_listings_widget`):** Take a prepared list
  of IDs and render the widget.

Only the render tool should include `_meta.ui.resourceUri`.

### Decoupled call flow

Recommended call flow:

1. The model calls the data tool (for example, `roll_dice`).
2. The model receives `structuredContent` from the data tool.
3. The model calls the render tool with that data.
4. The widget renders once with final, model-checked context.

### Example: Real estate follow-up queries

Suppose your plugin shows listing cards and a map, but your server-side `search` tool
only supports broad filters (city, price, beds, baths) and cannot filter by
school zone.

If a user asks, “Which of these are in the Richmond Primary School zone?”
decoupling helps:

1. `search` runs broadly and returns candidate listing IDs plus metadata.
2. The model refines that candidate set for the follow-up question.
3. The model calls `render_listings_widget` with only the filtered IDs.
4. The widget renders the final filtered set.

Best practices:

- Keep data tools reusable. Return complete `structuredContent` for chaining.
- Keep render tools focused on presentation. Don't mix business logic into the
  render handler.
- State the dependency in the render tool description (for example, “Always
  call `roll_dice` first”).
- Keep reruns intentional. Let the UI call data tools directly for local
  interactions like “Re-roll,” without remounting the widget.

### Decoupled example




For Go, use `github.com/modelcontextprotocol/go-sdk/mcp` at version `v1.8.0`. Install `esbuild` and `@modelcontextprotocol/ext-apps` with your browser dependencies. Bundle the browser component below with `npm exec -- esbuild dice-widget.ts --bundle --format=esm --outfile=dice-widget.js`, then run the Go server from that directory.







For the Ruby server, install the `mcp` gem at version `1.10.0`. Build the browser component as described below, then run the server from the project directory.




Example (decoupled dice tools):

Register a widget resource

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod/v3";
import { build } from "esbuild";
import { fileURLToPath } from "node:url";

const TEMPLATE_URI = "ui://widget/dice.html";

const server = new McpServer(
  { name: "Decoupled dice", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Bundle the same browser source that is type-checked and tested directly.
const widgetBundle = await build({
  entryPoints: [fileURLToPath(new URL("./dice-widget.ts", import.meta.url))],
  bundle: true,
  format: "esm",
  platform: "browser",
  write: false,
});
const widgetScript = widgetBundle.outputFiles[0].text.replace(
  /<\/script/gi,
  "<\\/script"
);
const widgetHtml = `<!doctype html><html lang="en"><meta charset="utf-8"><title>Dice widget</title><body><script type="module">${widgetScript}</script></body></html>`;

server.registerResource("dice-widget", TEMPLATE_URI, {}, async () => ({
  contents: [
    {
      uri: TEMPLATE_URI,
      mimeType: "text/html;profile=mcp-app",
      text: widgetHtml,
      _meta: { ui: { prefersBorder: true } },
    },
  ],
}));

// 1) Data tool: no output template, returns chainable structuredContent.
server.registerTool(
  "roll_dice",
  {
    title: "Roll dice",
    description: "Roll an N-sided die and return { sides, value }.",
    inputSchema: { sides: z.number().int().min(2) },
    outputSchema: {
      sides: z.number().int().min(2),
      value: z.number().int().min(1),
    },
    _meta: {
      "openai/toolInvocation/invoking": "Rolling…",
      "openai/toolInvocation/invoked": "Rolled.",
    },
  },
  async ({ sides }) => {
    const value = 1 + Math.floor(Math.random() * sides);
    return {
      structuredContent: { sides, value },
      content: [{ type: "text", text: `Rolled ${value} on ${sides} sides.` }],
    };
  }
);

// 2) Render tool: owns the template and requires data from roll_dice.
server.registerTool(
  "render_dice_widget",
  {
    title: "Render dice widget",
    description:
      "Render the dice widget from roll data. First call roll_dice, then pass its sides and value to this tool.",
    inputSchema: {
      sides: z.number().int().min(2),
      value: z.number().int().min(1),
    },
    outputSchema: {
      sides: z.number().int().min(2),
      value: z.number().int().min(1),
    },
    _meta: {
      ui: { resourceUri: TEMPLATE_URI },
      "openai/toolInvocation/invoking": "Rendering…",
      "openai/toolInvocation/invoked": "Rendered.",
    },
  },
  async ({ sides, value }) => ({
    structuredContent: { sides, value },
    content: [
      {
        type: "text",
        text: `Showing a ${sides}-sided roll: ${value}.`,
      },
    ],
  })
);

export default server;
```

```python
# From src/content/platform/python-examples, after installing TypeScript dependencies
# ../typescript-examples/node_modules/.bin/esbuild \
#   ../typescript-examples/apps-sdk-chatgpt-ui/dice-widget.ts \
#   --bundle --format=esm --outfile=dice-widget.js
import re
from pathlib import Path
from secrets import randbelow
from typing import Any

import uvicorn
from mcp import types
from mcp.server.lowlevel import Server
from mcp.server.lowlevel.helper_types import ReadResourceContents
from mcp.server.streamable_http_manager import StreamableHTTPSessionManager
from mcp.server.transport_security import TransportSecuritySettings
from pydantic import AnyUrl
from starlette.applications import Starlette
from starlette.routing import Route

server = Server("Decoupled dice", version="1.0.0")
RESOURCE_URI = "ui://widget/dice.html"
MIME_TYPE = "text/html;profile=mcp-app"


def component_html() -> str:
    script = Path("dice-widget.js").read_text(encoding="utf-8")
    script = re.sub(r"</script", lambda _: r"<\/script", script, flags=re.IGNORECASE)
    return (
        '<!doctype html><html lang="en"><meta charset="utf-8">'
        '<title>Dice</title><body><div id="root"></div>'
        '<script type="module">' + script + "</script></body></html>"
    )


@server.list_resources()
async def list_resources() -> list[types.Resource]:
    return [types.Resource(uri=AnyUrl(RESOURCE_URI), name="widget", mimeType=MIME_TYPE)]


@server.read_resource()
async def read_resource(uri: AnyUrl) -> list[ReadResourceContents]:
    if str(uri) != RESOURCE_URI:
        raise ValueError("Unknown component resource")
    return [ReadResourceContents(content=component_html(), mime_type=MIME_TYPE)]


@server.list_tools()
async def list_tools() -> list[types.Tool]:
    sides = {"type": "integer", "minimum": 2, "maximum": 2147483647}
    return [
        types.Tool(
            name="roll_dice",
            description="Roll a die without opening a widget.",
            inputSchema={
                "type": "object",
                "properties": {"sides": sides},
                "required": ["sides"],
            },
        ),
        types.Tool(
            name="render_dice_widget",
            description="Show a previously rolled die.",
            inputSchema={
                "type": "object",
                "properties": {
                    "sides": sides,
                    "value": {"type": "integer", "minimum": 1},
                },
                "required": ["sides", "value"],
            },
            _meta={"ui": {"resourceUri": RESOURCE_URI}},
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> types.CallToolResult:
    if name not in {"roll_dice", "render_dice_widget"}:
        raise ValueError("Unknown tool")
    sides = arguments.get("sides")
    if type(sides) is not int or not 2 <= sides <= 2147483647:
        raise ValueError("sides must be an integer between 2 and 2147483647")
    if name == "roll_dice":
        value = randbelow(sides) + 1
    else:
        value = arguments.get("value")
        if type(value) is not int or not 1 <= value <= sides:
            raise ValueError("value must be an integer between 1 and sides")
    return types.CallToolResult(
        content=[], structuredContent={"sides": sides, "value": value}
    )


# For an HTTPS tunnel, stop this server and append your tunnel's exact
# hostname to allowed_hosts below, for example "your-tunnel.ngrok.app".
# Use only the hostname, without https://, a path, or a wildcard. Keep the
# loopback entries for local testing. Update the hostname if the tunnel changes.
# Restart from src/content/platform/python-examples with:
# python apps-sdk-chatgpt-ui/decoupled.py
# Connect ChatGPT to the tunnel's HTTPS URL with /mcp appended.
# Keep allowed_origins restricted to trusted browser origins. ChatGPT's server
# connection does not require adding the tunnel URL to allowed_origins.
# Keep DNS-rebinding protection enabled; other hostnames must stay rejected.
sessions = StreamableHTTPSessionManager(
    server,
    json_response=True,
    stateless=True,
    security_settings=TransportSecuritySettings(
        allowed_hosts=["127.0.0.1:*", "localhost:*"],
        allowed_origins=["http://localhost:6274", "http://127.0.0.1:6274"],
    ),
)


class McpEndpoint:
    async def __call__(self, scope, receive, send):
        await sessions.handle_request(scope, receive, send)


app = Starlette(
    lifespan=lambda app: sessions.run(),
    routes=[Route("/mcp", McpEndpoint(), methods=["GET", "POST", "DELETE"])],
)


if __name__ == "__main__":
    # Fail before transport startup if the required built browser asset is missing.
    component_html()
    uvicorn.run(app, host="127.0.0.1", port=3000)
```

```go
package main

import (
	"context"
	"fmt"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
	"math/rand/v2"
	"os"
	"regexp"
)

type rollInput struct {
	Sides int `json:"sides"`
}
type die struct {
	Sides int `json:"sides"`
	Value int `json:"value"`
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "Decoupled dice", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "roll_dice", Description: "Roll a die without opening a widget.", InputSchema: map[string]any{"type": "object", "properties": map[string]any{"sides": map[string]any{"type": "integer", "minimum": 2}}, "required": []string{"sides"}}}, func(_ context.Context, _ *mcp.CallToolRequest, input rollInput) (*mcp.CallToolResult, die, error) {
		return &mcp.CallToolResult{Content: []mcp.Content{}}, die{Sides: input.Sides, Value: rand.IntN(input.Sides) + 1}, nil
	})
	mcp.AddTool(server, &mcp.Tool{Name: "render_dice_widget", Description: "Show a previously rolled die.", Meta: mcp.Meta{"ui": map[string]any{"resourceUri": "ui://widget/dice.html"}}, InputSchema: map[string]any{"type": "object", "properties": map[string]any{"sides": map[string]any{"type": "integer", "minimum": 2}, "value": map[string]any{"type": "integer", "minimum": 1}}, "required": []string{"sides", "value"}}}, func(_ context.Context, _ *mcp.CallToolRequest, input die) (*mcp.CallToolResult, die, error) {
		if input.Value > input.Sides {
			return nil, die{}, fmt.Errorf("value exceeds the die's sides")
		}
		return &mcp.CallToolResult{Content: []mcp.Content{}}, input, nil
	})
	// Build the browser component first, then run from the project root.
	script, err := os.ReadFile("dice-widget.js")
	if err != nil {
		log.Fatal(err)
	}
	escaped := regexp.MustCompile(`(?i)</script`).ReplaceAllLiteralString(string(script), `<\/script`)
	html := `<!doctype html><html lang="en"><meta charset="utf-8"><title>Dice</title><body><div id="root"></div><script type="module">` + escaped + `</script></body></html>`
	server.AddResource(&mcp.Resource{URI: "ui://widget/dice.html", Name: "widget", MIMEType: "text/html;profile=mcp-app"}, func(context.Context, *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: "ui://widget/dice.html", MIMEType: "text/html;profile=mcp-app", Text: html}}}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```java
import com.fasterxml.jackson.databind.json.JsonMapper;
import io.modelcontextprotocol.json.jackson2.JacksonMcpJsonMapper;
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpStatelessServerFeatures;
import io.modelcontextprotocol.server.transport.DefaultServerTransportSecurityValidator;
import io.modelcontextprotocol.server.transport.HttpServletStatelessServerTransport;
import io.modelcontextprotocol.spec.McpSchema;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ThreadLocalRandom;

// UI asset build from src/content/platform/java-examples:
// ../typescript-examples/node_modules/.bin/esbuild \
//   ../typescript-examples/apps-sdk-chatgpt-ui/dice-widget.ts \
//   --bundle --format=esm --outfile=dice-widget.js
public final class AppsSdkDecoupledDice {
  public static void main(String[] args) throws Exception {
    var mapper = new JacksonMcpJsonMapper(JsonMapper.builder().build());
    // Build the existing TypeScript component first; run from the asset directory.
    String script = Files.readString(Path.of("dice-widget.js"));
    script =
        script.replaceAll("(?i)</script", java.util.regex.Matcher.quoteReplacement("<\\/script"));
    String html =
        "<!doctype html><html lang=\"en\"><meta charset=\"utf-8\">"
            + "<title>Dice</title><body><div id=\"root\"></div><script type=\"module\">"
            + script
            + "</script></body></html>";
    var transport =
        HttpServletStatelessServerTransport.builder()
            .jsonMapper(mapper)
            .messageEndpoint("/mcp")
            .securityValidator(
                DefaultServerTransportSecurityValidator.builder()
                    .allowedHosts(List.of("localhost:*", "127.0.0.1:*"))
                    .allowedOrigins(List.of("http://localhost:6274", "http://127.0.0.1:6274"))
                    .build())
            .build();
    var resource =
        McpSchema.Resource.builder()
            .name("widget")
            .uri("ui://widget/dice.html")
            .mimeType("text/html;profile=mcp-app")
            .build();
    var builder =
        McpServer.sync(transport)
            .serverInfo("Decoupled dice", "1.0.0")
            .capabilities(
                McpSchema.ServerCapabilities.builder().tools(false).resources(false, false).build())
            .resources(
                new McpStatelessServerFeatures.SyncResourceSpecification(
                    resource,
                    (exchange, request) ->
                        new McpSchema.ReadResourceResult(
                            List.of(
                                new McpSchema.TextResourceContents(
                                    resource.uri(), resource.mimeType(), html)))));
    String rollSchema =
"""
{
  "type": "object",
  "properties": {
    "sides": {
      "type": "integer",
      "minimum": 2,
      "maximum": 2147483647
    }
  },
  "required": [
    "sides"
  ]
}
""";
    String renderSchema =
"""
{
  "type": "object",
  "properties": {
    "sides": {
      "type": "integer",
      "minimum": 2,
      "maximum": 2147483647
    },
    "value": {
      "type": "integer",
      "minimum": 1
    }
  },
  "required": [
    "sides",
    "value"
  ]
}
""";
    builder.toolCall(
        McpSchema.Tool.builder()
            .name("roll_dice")
            .description("Roll a die without opening a widget.")
            .inputSchema(mapper, rollSchema)
            .build(),
        (exchange, request) -> diceResult(request.arguments(), false));
    builder.toolCall(
        McpSchema.Tool.builder()
            .name("render_dice_widget")
            .description("Show a previously rolled die.")
            .inputSchema(mapper, renderSchema)
            .meta(Map.of("ui", Map.of("resourceUri", resource.uri())))
            .build(),
        (exchange, request) -> diceResult(request.arguments(), true));
    var server = builder.build();
    var httpServer = new org.eclipse.jetty.server.Server();
    var connector = new org.eclipse.jetty.server.ServerConnector(httpServer);
    connector.setHost("127.0.0.1");
    connector.setPort(3000);
    httpServer.addConnector(connector);
    var context = new org.eclipse.jetty.ee10.servlet.ServletContextHandler();
    context.setContextPath("/");
    context.addServlet(new org.eclipse.jetty.ee10.servlet.ServletHolder(transport), "/mcp");
    httpServer.setHandler(context);
    try {
      httpServer.start();
      httpServer.join();
    } finally {
      try {
        server.closeGracefully().block();
      } finally {
        httpServer.stop();
      }
    }
  }

  private static int positiveInteger(Object value) {
    if (!(value instanceof Number number)) return -1;
    double numeric = number.doubleValue();
    if (!Double.isFinite(numeric)
        || numeric != Math.rint(numeric)
        || numeric < 1
        || numeric > Integer.MAX_VALUE) return -1;
    return (int) numeric;
  }

  private static McpSchema.CallToolResult diceResult(
      Map<String, Object> arguments, boolean render) {
    int sides = positiveInteger(arguments.get("sides"));
    if (sides < 2)
      return McpSchema.CallToolResult.builder()
          .isError(true)
          .addTextContent("sides must be an integer between 2 and 2147483647")
          .build();
    int value =
        render
            ? positiveInteger(arguments.get("value"))
            : ThreadLocalRandom.current().nextInt(sides) + 1;
    if (value < 1 || value > sides)
      return McpSchema.CallToolResult.builder()
          .isError(true)
          .addTextContent("value must be an integer between 1 and sides")
          .build();
    return McpSchema.CallToolResult.builder()
        .content(List.of())
        .structuredContent(Map.of("sides", sides, "value", value))
        .isError(false)
        .build();
  }
}
```

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Text.RegularExpressions;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

// From ../typescript-examples, run npm ci, then
// npx --no-install esbuild apps-sdk-chatgpt-ui/dice-widget.ts --bundle --format=esm \
//   --outfile=../dotnet-examples/dice-widget.js
// Run this program from the dotnet-examples directory.
string script = await File.ReadAllTextAsync("dice-widget.js");
script = Regex.Replace(script, "</script", _ => "<\\/script", RegexOptions.IgnoreCase);
string html = "<!doctype html><html lang=\"en\"><meta charset=\"utf-8\">"
    + "<title>Dice</title><body><div id=\"root\"></div><script type=\"module\">"
    + script + "</script></body></html>";
const string resourceUri = "ui://widget/dice.html";
const string mimeType = "text/html;profile=mcp-app";
WebApplicationBuilder builder = WebApplication.CreateBuilder();
builder.WebHost.UseUrls("http://127.0.0.1:3000");
builder.Logging.ClearProviders();
builder.Services.AddMcpServer(options =>
{
    options.ServerInfo = new Implementation { Name = "Decoupled dice", Version = "1.0.0" };
})
.WithHttpTransport(options => options.Stateless = true)
.WithListToolsHandler((_, _) => ValueTask.FromResult(new ListToolsResult
{
    Tools =
    [
        new Tool
        {
            Name = "roll_dice",
            Description = "Roll a die without opening a widget.",
            InputSchema = JsonSerializer.Deserialize<JsonElement>("""
                {"type":"object","properties":{"sides":{"type":"integer","minimum":2,"maximum":2147483647}},"required":["sides"]}
                """),
        },
        new Tool
        {
            Name = "render_dice_widget",
            Description = "Show a previously rolled die.",
            InputSchema = JsonSerializer.Deserialize<JsonElement>("""
                {"type":"object","properties":{"sides":{"type":"integer","minimum":2,"maximum":2147483647},"value":{"type":"integer","minimum":1}},"required":["sides","value"]}
                """),
            Meta = new JsonObject { ["ui"] = new JsonObject { ["resourceUri"] = resourceUri } },
        },
    ],
}))
.WithCallToolHandler((request, _) =>
{
    var parameters = request.Params ?? throw new ArgumentException("Missing tool parameters");
    if (parameters.Name is not ("roll_dice" or "render_dice_widget"))
    {
        throw new ArgumentException("Unknown tool");
    }
    var arguments = parameters.Arguments;
    if (arguments is null || !arguments.TryGetValue("sides", out var sidesJson)
        || sidesJson.ValueKind != JsonValueKind.Number || !sidesJson.TryGetInt32(out int sides) || sides < 2)
    {
        return ValueTask.FromResult(Error("sides must be an integer between 2 and 2147483647"));
    }
    int value;
    if (parameters.Name == "roll_dice")
    {
        value = Random.Shared.Next(sides) + 1;
    }
    else if (!arguments.TryGetValue("value", out var valueJson)
        || valueJson.ValueKind != JsonValueKind.Number || !valueJson.TryGetInt32(out value) || value < 1 || value > sides)
    {
        return ValueTask.FromResult(Error("value must be an integer between 1 and sides"));
    }
    return ValueTask.FromResult(new CallToolResult
    {
        Content = [],
        StructuredContent = JsonSerializer.SerializeToElement(new { sides, value }),
    });
})
.WithListResourcesHandler((_, _) => ValueTask.FromResult(new ListResourcesResult
{
    Resources = [new Resource { Uri = resourceUri, Name = "widget", MimeType = mimeType }],
}))
.WithReadResourceHandler((request, _) =>
{
    if (request.Params?.Uri != resourceUri) throw new ArgumentException("Unknown component resource");
    return ValueTask.FromResult(new ReadResourceResult
    {
        Contents = [new TextResourceContents
        {
            Uri = resourceUri,
            MimeType = mimeType,
            Text = html,
        }],
    });
});
WebApplication app = builder.Build();
app.Use(async (context, next) =>
{
    if (context.Request.Host.Host is not ("localhost" or "127.0.0.1"))
    {
        context.Response.StatusCode = StatusCodes.Status421MisdirectedRequest;
        return;
    }
    var origin = context.Request.Headers.Origin;
    if (origin.Count > 0 && (origin.Count != 1 || origin[0] is not
        ("http://localhost:6274" or "http://127.0.0.1:6274")))
    {
        context.Response.StatusCode = StatusCodes.Status403Forbidden;
        return;
    }
    context.Response.Headers.Vary = "Origin";
    if (origin.Count == 1)
    {
        context.Response.Headers.AccessControlAllowOrigin = origin[0];
        context.Response.Headers.AccessControlExposeHeaders = "Mcp-Session-Id";
    }
    if (HttpMethods.IsOptions(context.Request.Method))
    {
        context.Response.Headers.AccessControlAllowMethods = "GET, POST, DELETE, OPTIONS";
        context.Response.Headers.AccessControlAllowHeaders =
            "Content-Type, Mcp-Session-Id, Mcp-Protocol-Version";
        context.Response.StatusCode = StatusCodes.Status204NoContent;
        return;
    }
    await next(context);
});
app.MapMcp("/mcp");

await using (app)
{
    await app.RunAsync();
}

static CallToolResult Error(string message) => new()
{
    IsError = true,
    Content = [new TextContentBlock { Text = message }],
};
```

```ruby
require "mcp"

class RollDice < MCP::Tool
  tool_name "roll_dice"
  description "Roll a die without opening a widget."
  input_schema(
    properties: {
      sides: {
        type: "integer",
        minimum: 2
      }
    }, required: ["sides"]
  )

  def self.call(sides:, server_context:)
    MCP::Tool::Response.new(
      [], structured_content: {
        sides: sides,
        value: rand(1..sides)
      }
    )
  end
end

class RenderDiceWidget < MCP::Tool
  tool_name "render_dice_widget"
  description "Show a previously rolled die."
  input_schema(
    properties: {
      sides: {
        type: "integer",
        minimum: 2
      },
      value: {
        type: "integer",
        minimum: 1
      }
    },
    required: ["sides", "value"]
  )
  meta(ui: { resourceUri: "ui://widget/dice.html" })

  def self.call(sides:, value:, server_context:)
    if value > sides
      return MCP::Tool::Response.new(
        [
          {
            type: "text",
            text: "Value exceeds the die's sides."
          }
        ], error: true
      )
    end

    MCP::Tool::Response.new(
      [], structured_content: {
        sides: sides,
        value: value
      }
    )
  end
end

# Build the browser component first, then run this server from the project root.
script = File.read("dice-widget.js").gsub(%r{</script}i) { '<\/script' }
html = '<!doctype html><html lang="en"><meta charset="utf-8">' \
       '<title>Dice</title><body><div id="root"></div>' \
       "<script type=\"module\">#{script}</script></body></html>"
resource = MCP::Resource.define(uri: "ui://widget/dice.html", name: "widget", mime_type: "text/html;profile=mcp-app") do
  [
    MCP::Resource::TextContents.new(
      uri: "ui://widget/dice.html", mime_type: "text/html;profile=mcp-app", text: html
    )
  ]
end

server = MCP::Server.new(
  name: "Decoupled dice", version: "1.0.0", tools: [RollDice, RenderDiceWidget],
  resources: [
    resource
  ]
)
MCP::Server::Transports::StdioTransport.new(server).open
```


Install `esbuild` with the server dependencies, then save this browser code as
`dice-widget.ts` beside the server file:

```typescript
import { App } from "@modelcontextprotocol/ext-apps";

const container = document.createElement("div");
container.style.cssText = "font-family: system-ui; padding: 8px;";
const resultLine = document.createElement("div");
resultLine.style.cssText = "font-size: 20px; margin-bottom: 6px;";
const output = document.createElement("span");
output.id = "out";
output.textContent = "—";
resultLine.append("Result: ", output);
const rerollButton = document.createElement("button");
rerollButton.id = "reroll";
rerollButton.textContent = "Re-roll";
rerollButton.disabled = true;
container.append(resultLine, rerollButton);
document.body.append(container);

const app = new App({ name: "Dice widget", version: "1.0.0" });
let latestInput: Record<string, unknown> = {};
let latestOutput: Record<string, unknown> = {};

function render(result: Record<string, unknown>) {
  latestOutput = result;
  output.textContent = String(result.value ?? "—");
}

app.ontoolinput = ({ arguments: input }) => {
  latestInput = input ?? {};
};
app.ontoolresult = (result) => {
  render(result.structuredContent ?? {});
};

// Call the data tool without remounting the widget.
rerollButton.onclick = async () => {
  const candidate = latestOutput.sides ?? latestInput.sides;
  const sides =
    typeof candidate === "number" &&
    Number.isInteger(candidate) &&
    candidate >= 2
      ? candidate
      : 6;
  const next = await app.callServerTool({
    name: "roll_dice",
    arguments: { sides },
  });
  if (next.structuredContent) render(next.structuredContent);
};

await app.connect();
rerollButton.disabled = false;
```





For Ruby, bundle `dice-widget.ts` into `dice-widget.js` with `npm exec -- esbuild dice-widget.ts --bundle --format=esm --outfile=dice-widget.js`. Install `esbuild` and `@modelcontextprotocol/ext-apps` with your browser dependencies. The Ruby server reads that compiled file.




## Manage state

UI from an MCP server works with three kinds of state:

| State type                        | Owner                          | Lifetime                             | Examples                                 |
| --------------------------------- | ------------------------------ | ------------------------------------ | ---------------------------------------- |
| **Business data (authoritative)** | MCP server or external service | Long-lived                           | Tasks, tickets, documents                |
| **UI state (ephemeral)**          | UI instance                    | Active UI instance                   | Selected row, expanded panel, sort order |
| **Cross-session state (durable)** | Storage you control            | Cross-session and cross-conversation | Saved filters, view mode, workspace      |

Keep each value with the system that owns it. The UI should render
authoritative data from tool results and layer temporary presentation state on
top.

```text
MCP server or external service
│
├── Authoritative business data
│
▼
UI
│
├── Ephemeral presentation state
│
└── Rendered view = business data + UI state
```

### Keep business data on the server

Business data is the source of truth. Do not store it only in the UI. When a
user takes an action:

1. The UI calls an MCP tool.
2. The server validates the request and updates the data.
3. The server returns the updated authoritative snapshot.
4. The UI renders the snapshot while preserving compatible presentation
   state.

Return enough structured content for both the model and UI to understand the
new state. This also lets the conversation remain useful if the UI cannot
load.

### Keep temporary UI state in the UI

Use framework state for values that only affect presentation, such as a
selected item, open panel, or draft filter. Each rendered UI instance has its
own state.

When the model needs to know about a selection or staged edit, send that
information through `ui/update-model-context`. This is the portable MCP Apps
mechanism for updating model-visible context.

ChatGPT also provides optional widget-scoped persistence:

- Read the current snapshot from `window.openai.widgetState`.
- Write a new snapshot with `window.openai.setWidgetState(state)`.

`setWidgetState` is synchronous. Call it after each meaningful UI-state change;
there is nothing to `await`.

```tsx
import { useState } from "react";

export function TaskList({
  tasks,
}: {
  tasks: { id: string; title: string }[];
}) {
  const [state, setState] = useState(
    window.openai?.widgetState ?? { selectedId: null }
  );

  function selectTask(selectedId: string) {
    const nextState = { ...state, selectedId };
    setState(nextState);
    window.openai?.setWidgetState?.(nextState);
  }

  return (
    <ul>
      {tasks.map((task) => (
        <li key={task.id}>
          <button
            type="button"
            aria-pressed={state.selectedId === task.id}
            onClick={() => selectTask(task.id)}
          >
            {task.title}
          </button>
        </li>
      ))}
    </ul>
  );
}
```


Widget state belongs to one rendered UI instance. Do not use it as the source
of truth for business data or as durable storage.

#### Make images visible to the model

For UI that works with images, use the structured widget-state shape:

- `modelContent`: Text or JSON the model should see.
- `privateContent`: UI-only state the model should not see.
- `imageIds`: File IDs the model should receive on later turns.

```tsx
const selectedFiles = await window.openai.selectFiles();

window.openai.setWidgetState({
  modelContent: "Review the currently selected images.",
  privateContent: {
    currentView: "image-viewer",
    filters: ["crop", "sharpen"],
  },
  imageIds: selectedFiles.map((file) => file.fileId),
});
```


Only include file IDs uploaded with `window.openai.uploadFile`, selected with
`window.openai.selectFiles`, received through tool input file parameters, or
returned through tool result file references.

### Store cross-session state on your server

Store preferences and data that must survive across conversations, devices, or
sessions in storage you control. Authenticate the user so the MCP server can
map each request to the correct account.

When you add durable storage:

- Keep latency low enough for interactive UI.
- Protect private data with server-side authorization.
- Plan for data residency and compliance requirements.
- Apply rate limits to traffic from retries or concurrent UI instances.
- Version stored objects so you can migrate them without breaking existing
  conversations.

Avoid `localStorage` for core state. UI runs in an isolated iframe, and browser
storage does not provide a reliable cross-device or cross-session data layer.

## Scaffold the component project

Now that you understand the MCP Apps bridge (and optional ChatGPT extensions),
it’s time to scaffold your component project.

As best practice, keep the component code separate from your server logic. A common layout is:

```text
plugin-ui/
  server/            # MCP server (Python or Node)
  web/               # Component bundle source
    package.json
    tsconfig.json
    src/component.tsx
    dist/component.js   # Build output
```

Create the project and install dependencies (Node 18+ recommended):

```bash
cd plugin-ui/web
npm init -y
npm install react@^19 react-dom@^19
npm install -D typescript esbuild
```

If your component requires drag-and-drop, charts, or other libraries, add them now. Keep the dependency set lean to reduce bundle size.

## Author the React component

Your entry file should mount a component into a `root` element and render from
the latest tool result delivered over the MCP Apps bridge (for example,
`ui/notifications/tool-result`).

The [examples page](https://developers.openai.com/plugins/build/examples) includes sample UI, such as the Pizzaz list of
pizza restaurants.

### Explore the Pizzaz component gallery

The [UI examples](https://developers.openai.com/plugins/build/examples) include example components. Treat them as blueprints when shaping your own UI:

- **Pizzaz List:** Ranked card list with favorites and call-to-action buttons.  
  ![Screenshot of the Pizzaz list component](https://developers.openai.com/images/apps-sdk/pizzaz-list.png)
- **Pizzaz Carousel:** Embla-powered horizontal scroller that demonstrates media-heavy layouts.  
  ![Screenshot of the Pizzaz carousel component](https://developers.openai.com/images/apps-sdk/pizzaz-carousel.png)
- **Pizzaz Map:** Mapbox integration with fullscreen inspector and host state sync.  
  ![Screenshot of the Pizzaz map component](https://developers.openai.com/images/apps-sdk/pizzaz-map.png)
- **Pizzaz Album:** Stacked gallery view built for deep dives on a single place.  
  ![Screenshot of the Pizzaz album component](https://developers.openai.com/images/apps-sdk/pizzaz-album.png)
- **Pizzaz Video:** Scripted player with overlays and fullscreen controls.

Each example shows how to bundle assets, wire host APIs, and structure state for real conversations. Copy the one closest to your use case and adapt the data layer for your tool responses.

### React helper hooks

A small helper to subscribe to `ui/notifications/tool-result`:

```tsx
import { useEffect, useState } from "react";

type ToolResult = { structuredContent?: unknown } | null;

export function useToolResult() {
  const [toolResult, setToolResult] = useState<ToolResult>(null);

  useEffect(() => {
    const onMessage = (event: MessageEvent) => {
      if (event.source !== window.parent) return;
      const message = event.data;
      if (!message || message.jsonrpc !== "2.0") return;
      if (message.method !== "ui/notifications/tool-result") return;
      setToolResult(message.params ?? null);
    };

    window.addEventListener("message", onMessage, { passive: true });
    return () => window.removeEventListener("message", onMessage);
  }, []);

  return toolResult;
}
```


Render from `toolResult?.structuredContent`, and treat it as untrusted input.

## Widget localization

The host mirrors the locale to `document.documentElement.lang`. Use that locale
to load translations and format dates/numbers. Run `npm install react-intl`
in the `web/` directory before using this example:

```tsx
import { IntlProvider, FormattedMessage } from "react-intl";

const messages: Record<string, Record<string, string>> = {
  "en-US": { greeting: "Hello" },
  "es-ES": { greeting: "Hola" },
};

export function PluginUI() {
  const locale = document.documentElement.lang || "en-US";
  return (
    

);
}
```


## Bundle for the iframe

Once you finish writing your React component, you can build it into a single JavaScript module that the server can inline:

```json
// package.json
{
  "scripts": {
    "build": "esbuild src/component.tsx --bundle --format=esm --jsx=automatic --outfile=dist/component.js"
  }
}
```

Run `npm run build` to produce `dist/component.js`. If esbuild complains about missing dependencies, confirm you ran `npm install` in the `web/` directory and that your imports match installed package names (for example, `@react-dnd/html5-server-side` vs `react-dnd-html5-server-side`).

## Embed the component in the server response

Expose the component as an MCP resource with the MCP Apps UI MIME type
(`text/html;profile=mcp-app`). If you use
`@modelcontextprotocol/ext-apps/server`, prefer `RESOURCE_MIME_TYPE` instead of
embedding the string:

Return a widget with a tool result

```typescript
import {
  registerAppResource,
  RESOURCE_MIME_TYPE,
} from "@modelcontextprotocol/ext-apps/server";
import { readFileSync } from "node:fs";

const component = readFileSync("web/dist/component.js", "utf8");

registerAppResource(
  server,
  "project-board",
  "ui://project-board/v1.html",
  {},
  async () => ({
    contents: [
      {
        uri: "ui://project-board/v1.html",
        mimeType: RESOURCE_MIME_TYPE,
        text: `<div id="root"></div><script type="module">${component}</script>`,
        _meta: {
          ui: {
            prefersBorder: true,
            domain: "https://example.com",
            csp: {
              connectDomains: ["https://api.example.com"],
              resourceDomains: ["https://static.example.com"],
            },
          },
        },
      },
    ],
  })
);
```

```python
# From src/content/platform/python-examples, after installing TypeScript dependencies
# ../typescript-examples/node_modules/.bin/esbuild \
#   ../typescript-examples/apps-sdk-chatgpt-ui/keep-temporary-ui-state-in-the-ui.tsx \
#   --bundle --format=esm --outfile=web/dist/component.js
import re
from pathlib import Path
from typing import Any

import uvicorn
from mcp import types
from mcp.server.lowlevel import Server
from mcp.server.lowlevel.helper_types import ReadResourceContents
from mcp.server.streamable_http_manager import StreamableHTTPSessionManager
from mcp.server.transport_security import TransportSecuritySettings
from pydantic import AnyUrl
from starlette.applications import Starlette
from starlette.routing import Route

server = Server("docs-example", version="1.0.0")
RESOURCE_URI = "ui://project-board/v1.html"
MIME_TYPE = "text/html;profile=mcp-app"


def component_html() -> str:
    script = Path("web/dist/component.js").read_text(encoding="utf-8")
    script = re.sub(r"</script", lambda _: r"<\/script", script, flags=re.IGNORECASE)
    return (
        '<!doctype html><html lang="en"><meta charset="utf-8">'
        '<title>Project board</title><body><div id="root"></div>'
        '<script type="module">' + script + "</script></body></html>"
    )


@server.list_resources()
async def list_resources() -> list[types.Resource]:
    return [types.Resource(uri=AnyUrl(RESOURCE_URI), name="widget", mimeType=MIME_TYPE)]


@server.read_resource()
async def read_resource(uri: AnyUrl) -> list[ReadResourceContents]:
    if str(uri) != RESOURCE_URI:
        raise ValueError("Unknown component resource")
    return [
        ReadResourceContents(
            content=component_html(),
            mime_type=MIME_TYPE,
            meta={
                "ui": {
                    "prefersBorder": True,
                    "domain": "https://example.com",
                    "csp": {
                        "connectDomains": ["https://api.example.com"],
                        "resourceDomains": ["https://static.example.com"],
                    },
                }
            },
        )
    ]


@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="open_project_board",
            title="Open project board",
            inputSchema={"type": "object", "properties": {}},
            _meta={"ui": {"resourceUri": RESOURCE_URI}},
        )
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> types.CallToolResult:
    if name != "open_project_board":
        raise ValueError("Unknown tool")
    return types.CallToolResult(
        content=[
            types.TextContent(
                type="text", text="Review the tasks in the project board."
            )
        ]
    )


# For an HTTPS tunnel, stop this server and append your tunnel's exact
# hostname to allowed_hosts below, for example "your-tunnel.ngrok.app".
# Use only the hostname, without https://, a path, or a wildcard. Keep the
# loopback entries for local testing. Update the hostname if the tunnel changes.
# Restart from src/content/platform/python-examples with:
# python apps-sdk-chatgpt-ui/embed-the-component-in-the-server-response.py
# Connect ChatGPT to the tunnel's HTTPS URL with /mcp appended.
# Keep allowed_origins restricted to trusted browser origins. ChatGPT's server
# connection does not require adding the tunnel URL to allowed_origins.
# Keep DNS-rebinding protection enabled; other hostnames must stay rejected.
sessions = StreamableHTTPSessionManager(
    server,
    json_response=True,
    stateless=True,
    security_settings=TransportSecuritySettings(
        allowed_hosts=["127.0.0.1:*", "localhost:*"],
        allowed_origins=["http://localhost:6274", "http://127.0.0.1:6274"],
    ),
)


class McpEndpoint:
    async def __call__(self, scope, receive, send):
        await sessions.handle_request(scope, receive, send)


app = Starlette(
    lifespan=lambda app: sessions.run(),
    routes=[Route("/mcp", McpEndpoint(), methods=["GET", "POST", "DELETE"])],
)


if __name__ == "__main__":
    # Fail before transport startup if the required built browser asset is missing.
    component_html()
    uvicorn.run(app, host="127.0.0.1", port=3000)
```

```go
package main

import (
	"context"
	"log"
	"os"
	"regexp"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "open_project_board", Title: "Open project board", Meta: mcp.Meta{"ui": map[string]any{"resourceUri": "ui://project-board/v1.html"}}}, func(context.Context, *mcp.CallToolRequest, struct{}) (*mcp.CallToolResult, any, error) {
		return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: "Review the tasks in the project board."}}}, nil, nil
	})
	// Build the browser component first, then run from the project root.
	script, err := os.ReadFile("web/dist/component.js")
	if err != nil {
		log.Fatal(err)
	}
	escaped := regexp.MustCompile(`(?i)</script`).ReplaceAllLiteralString(string(script), `<\/script`)
	html := `<!doctype html><html lang="en"><meta charset="utf-8"><title>Project board</title><body><div id="root"></div><script type="module">` + escaped + `</script></body></html>`
	server.AddResource(&mcp.Resource{URI: "ui://project-board/v1.html", Name: "widget", MIMEType: "text/html;profile=mcp-app"}, func(context.Context, *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: "ui://project-board/v1.html", MIMEType: "text/html;profile=mcp-app", Text: html, Meta: mcp.Meta{"ui": map[string]any{"prefersBorder": true, "domain": "https://example.com", "csp": map[string]any{"connectDomains": []string{"https://api.example.com"}, "resourceDomains": []string{"https://static.example.com"}}}}}}}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```java
import com.fasterxml.jackson.databind.json.JsonMapper;
import io.modelcontextprotocol.json.jackson2.JacksonMcpJsonMapper;
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpStatelessServerFeatures;
import io.modelcontextprotocol.server.transport.DefaultServerTransportSecurityValidator;
import io.modelcontextprotocol.server.transport.HttpServletStatelessServerTransport;
import io.modelcontextprotocol.spec.McpSchema;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

// UI asset build from src/content/platform/java-examples:
// ../typescript-examples/node_modules/.bin/esbuild \
//   ../typescript-examples/apps-sdk-chatgpt-ui/keep-temporary-ui-state-in-the-ui.tsx \
//   --bundle --format=esm --outfile=web/dist/component.js
public final class AppsSdkProjectBoard {
  public static void main(String[] args) throws Exception {
    var mapper = new JacksonMcpJsonMapper(JsonMapper.builder().build());
    // Build the existing TypeScript component first; run from the asset directory.
    String script = Files.readString(Path.of("web/dist/component.js"));
    script =
        script.replaceAll("(?i)</script", java.util.regex.Matcher.quoteReplacement("<\\/script"));
    String html =
        "<!doctype html><html lang=\"en\"><meta charset=\"utf-8\">"
            + "<title>Project board</title><body><div id=\"root\"></div><script type=\"module\">"
            + script
            + "</script></body></html>";
    var transport =
        HttpServletStatelessServerTransport.builder()
            .jsonMapper(mapper)
            .messageEndpoint("/mcp")
            .securityValidator(
                DefaultServerTransportSecurityValidator.builder()
                    .allowedHosts(List.of("localhost:*", "127.0.0.1:*"))
                    .allowedOrigins(List.of("http://localhost:6274", "http://127.0.0.1:6274"))
                    .build())
            .build();
    var resource =
        McpSchema.Resource.builder()
            .name("widget")
            .uri("ui://project-board/v1.html")
            .mimeType("text/html;profile=mcp-app")
            .build();
    var builder =
        McpServer.sync(transport)
            .serverInfo("docs-example", "1.0.0")
            .capabilities(
                McpSchema.ServerCapabilities.builder().tools(false).resources(false, false).build())
            .resources(
                new McpStatelessServerFeatures.SyncResourceSpecification(
                    resource,
                    (exchange, request) ->
                        new McpSchema.ReadResourceResult(
                            List.of(
                                new McpSchema.TextResourceContents(
                                    resource.uri(),
                                    resource.mimeType(),
                                    html,
                                    Map.of(
                                        "ui",
                                        Map.of(
                                            "prefersBorder",
                                            true,
                                            "domain",
                                            "https://example.com",
                                            "csp",
                                            Map.of(
                                                "connectDomains",
                                                    List.of("https://api.example.com"),
                                                "resourceDomains",
                                                    List.of("https://static.example.com")))))))));
    builder.toolCall(
        McpSchema.Tool.builder()
            .name("open_project_board")
            .title("Open project board")
            .inputSchema(mapper, "{\"type\":\"object\",\"properties\":{}}")
            .meta(Map.of("ui", Map.of("resourceUri", resource.uri())))
            .build(),
        (exchange, request) ->
            McpSchema.CallToolResult.builder()
                .addTextContent("Review the tasks in the project board.")
                .isError(false)
                .build());
    var server = builder.build();
    var httpServer = new org.eclipse.jetty.server.Server();
    var connector = new org.eclipse.jetty.server.ServerConnector(httpServer);
    connector.setHost("127.0.0.1");
    connector.setPort(3000);
    httpServer.addConnector(connector);
    var context = new org.eclipse.jetty.ee10.servlet.ServletContextHandler();
    context.setContextPath("/");
    context.addServlet(new org.eclipse.jetty.ee10.servlet.ServletHolder(transport), "/mcp");
    httpServer.setHandler(context);
    try {
      httpServer.start();
      httpServer.join();
    } finally {
      try {
        server.closeGracefully().block();
      } finally {
        httpServer.stop();
      }
    }
  }
}
```

```csharp
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Text.RegularExpressions;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

// From ../typescript-examples, run npm ci, then
// npx --no-install esbuild apps-sdk-chatgpt-ui/keep-temporary-ui-state-in-the-ui.tsx --bundle --format=esm \
//   --outfile=../dotnet-examples/web/dist/component.js
// Run this program from the dotnet-examples directory.
string script = await File.ReadAllTextAsync("web/dist/component.js");
script = Regex.Replace(script, "</script", _ => "<\\/script", RegexOptions.IgnoreCase);
string html = "<!doctype html><html lang=\"en\"><meta charset=\"utf-8\">"
    + "<title>Project board</title><body><div id=\"root\"></div><script type=\"module\">"
    + script + "</script></body></html>";
const string resourceUri = "ui://project-board/v1.html";
const string mimeType = "text/html;profile=mcp-app";
WebApplicationBuilder builder = WebApplication.CreateBuilder();
builder.WebHost.UseUrls("http://127.0.0.1:3000");
builder.Logging.ClearProviders();
builder.Services.AddMcpServer(options =>
{
    options.ServerInfo = new Implementation { Name = "docs-example", Version = "1.0.0" };
})
.WithHttpTransport(options => options.Stateless = true)
.WithListToolsHandler((_, _) => ValueTask.FromResult(new ListToolsResult
{
    Tools =
    [
        new Tool
        {
            Name = "open_project_board",
            Title = "Open project board",
            InputSchema = JsonSerializer.SerializeToElement(new { type = "object", properties = new { } }),
            Meta = new JsonObject { ["ui"] = new JsonObject { ["resourceUri"] = resourceUri } },
        },
    ],
}))
.WithCallToolHandler((request, _) =>
{
    if (request.Params?.Name != "open_project_board")
    {
        throw new ArgumentException("Unknown tool");
    }
    return ValueTask.FromResult(new CallToolResult
    {
        Content = [new TextContentBlock { Text = "Review the tasks in the project board." }],
    });
})
.WithListResourcesHandler((_, _) => ValueTask.FromResult(new ListResourcesResult
{
    Resources = [new Resource { Uri = resourceUri, Name = "widget", MimeType = mimeType }],
}))
.WithReadResourceHandler((request, _) =>
{
    if (request.Params?.Uri != resourceUri) throw new ArgumentException("Unknown component resource");
    return ValueTask.FromResult(new ReadResourceResult
    {
        Contents = [new TextResourceContents
        {
            Uri = resourceUri,
            MimeType = mimeType,
            Text = html,
            Meta = new JsonObject
            {
                ["ui"] = new JsonObject
                {
                    ["prefersBorder"] = true,
                    ["domain"] = "https://example.com",
                    ["csp"] = new JsonObject
                    {
                        ["connectDomains"] = new JsonArray("https://api.example.com"),
                        ["resourceDomains"] = new JsonArray("https://static.example.com"),
                    },
                },
            },
        }],
    });
});
WebApplication app = builder.Build();
app.Use(async (context, next) =>
{
    if (context.Request.Host.Host is not ("localhost" or "127.0.0.1"))
    {
        context.Response.StatusCode = StatusCodes.Status421MisdirectedRequest;
        return;
    }
    var origin = context.Request.Headers.Origin;
    if (origin.Count > 0 && (origin.Count != 1 || origin[0] is not
        ("http://localhost:6274" or "http://127.0.0.1:6274")))
    {
        context.Response.StatusCode = StatusCodes.Status403Forbidden;
        return;
    }
    context.Response.Headers.Vary = "Origin";
    if (origin.Count == 1)
    {
        context.Response.Headers.AccessControlAllowOrigin = origin[0];
        context.Response.Headers.AccessControlExposeHeaders = "Mcp-Session-Id";
    }
    if (HttpMethods.IsOptions(context.Request.Method))
    {
        context.Response.Headers.AccessControlAllowMethods = "GET, POST, DELETE, OPTIONS";
        context.Response.Headers.AccessControlAllowHeaders =
            "Content-Type, Mcp-Session-Id, Mcp-Protocol-Version";
        context.Response.StatusCode = StatusCodes.Status204NoContent;
        return;
    }
    await next(context);
});
app.MapMcp("/mcp");

await using (app)
{
    await app.RunAsync();
}
```

```ruby
require "mcp"

class OpenProjectBoard < MCP::Tool
  tool_name "open_project_board"
  title "Open project board"
  input_schema(properties: {})
  meta(ui: { resourceUri: "ui://project-board/v1.html" })

  def self.call(server_context:)
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: "Review the tasks in the project board."
        }
      ]
    )
  end
end

# Build the browser component first, then run this server from the project root.
script = File.read("web/dist/component.js").gsub(%r{</script}i) { '<\/script' }
html = '<!doctype html><html lang="en"><meta charset="utf-8">' \
       '<title>Project board</title><body><div id="root"></div>' \
       "<script type=\"module\">#{script}</script></body></html>"
resource = MCP::Resource.define(uri: "ui://project-board/v1.html", name: "widget", mime_type: "text/html;profile=mcp-app") do
  [
    MCP::Resource::TextContents.new(
      uri: "ui://project-board/v1.html", mime_type: "text/html;profile=mcp-app", text: html,
      meta: {
        ui: {
          prefersBorder: true,

          domain: "https://example.com",
          csp: {
            connectDomains: ["https://api.example.com"],
            resourceDomains: ["https://static.example.com"]
          }
        }
      }
    )
  ]
end

server = MCP::Server.new(
  name: "docs-example", version: "1.0.0", tools: [OpenProjectBoard], resources: [resource]
)
MCP::Server::Transports::StdioTransport.new(server).open
```





For Go, use `github.com/modelcontextprotocol/go-sdk/mcp` at version `v1.8.0`. Build `web/dist/component.js` in the preceding steps, then run the Go server from the project root.







For Ruby, install the `mcp` gem at version `1.10.0` and run the server from the project root after building `web/dist/component.js` in the preceding steps.




Associate the resource URI with only the tools that should render the
component. For broader MCP Apps compatibility, use `_meta.ui.resourceUri`.
ChatGPT also honors `_meta["openai/outputTemplate"]` as a compatibility alias.

Treat the resource URI as a cache key. When you make a breaking change to the
HTML, JavaScript, or CSS, publish a new URI and update every tool that
references it.

### Content security policy (CSP)

Declare the exact domains the component connects to or loads resources from:

- `connectDomains` for API requests.
- `resourceDomains` for scripts, styles, images, and other assets.
- `frameDomains` only when the component must embed specific iframe origins.

Nested frames are blocked by default. Keep each allowlist as narrow as possible.
The plugin review process checks the declared policy against the UI behavior.

Plugins with UI can embed pages from their MCP server’s own registrable domain, including full existing editors and admin interfaces. For example, a server at `https://api.example.com/mcp` can embed `https://app.example.com`: both use the registrable domain `example.com`. Separate tenants on a shared hosting service count as different domains; using the same hosting provider does not establish ownership.

Declare each required iframe origin in the resource CSP using `_meta.ui.csp.frameDomains` (or the legacy `_meta["openai/widgetCSP"].frame_domains`). For third-party domains, iframe embeds should be limited to cases where the embedded experience is essential.

You must still provide a justification [when submitting a plugin that uses iframes](https://developers.openai.com/plugins/deploy/submission#complete-review-information). Explain what each embedded page does, why the plugin embeds it, and who controls its domain. Iframe use can require additional review or escalation and may lead to slower approval or rejection if the content cannot be assessed. Sharing the MCP server’s domain does not guarantee approval.

All other plugin requirements still apply to embedded pages, including the [checkout](https://developers.openai.com/plugins/plugin-guidelines#5.3) and [privacy](https://developers.openai.com/plugins/plugin-guidelines#4) requirements.

Component UI templates are the recommended path for production.

During development you can rebuild the component bundle whenever your React code changes and hot-reload the server.

## Offer checkout in your UI

If you want to offer users the ability to check out through your plugin's UI
flows, use the component to present products, prices, terms, and payment choices
before confirmation. Keep the underlying catalog and order tools useful without
UI, then choose an external checkout flow or, when available, an embedded
payment option.

### Use external checkout by default

External checkout is the recommended and generally available approach. Link
from the component to a merchant-hosted checkout flow on your own domain,
where you handle:

- Pricing and payment collection.
- Taxes, discounts, and fees.
- Shipping and fulfillment.
- Refunds, support, and compliance.

Current approval is limited to plugins for physical-goods purchases. Do not
offer other commerce categories unless OpenAI has explicitly enabled them for
your plugin.

### Use saved payment methods

For eligible physical-goods purchases, optional UI can let customers select a
payment method they previously saved with your service. This flow can display
eligible saved methods but cannot collect new payment credentials. Your MCP
server processes the purchase and returns the authoritative order result.

### Use the ChatGPT payment sheet



Embedded checkout with the ChatGPT payment sheet is in private beta for select
  marketplaces and is not available to all developers or users.



For enabled integrations, `window.openai.requestCheckout` opens the ChatGPT
payment sheet:

```tsx
const order = await window.openai.requestCheckout(checkoutSession);
```


The checkout flow has four parts:

1. An MCP tool returns a checkout session in `structuredContent`.
2. The component displays the line items, totals, terms, and fulfillment
   choices.
3. The component calls `requestCheckout(checkoutSession)` after the user
   chooses to pay.
4. ChatGPT sends the selected payment token to the MCP server's
   `complete_checkout` tool, which charges the payment method and returns the
   completed order.

The checkout session must include:

- A unique session ID.
- Line items and quantities.
- Totals in integer minor currency units.
- Payment-provider and merchant metadata.
- Required legal, privacy, refund, and support links.

Treat the server as the source of truth for prices and order status. Verify the
payment token, make the operation idempotent, persist the order, and return an
authoritative receipt. Never trust totals calculated only in the component.

Use `payment_mode: "test"` to exercise the end-to-end flow without moving real
funds. Handle cancellation, declined payments, and payment-provider errors in
the component.

For complete checkout-session fields, payment-provider behavior, the
`complete_checkout` result shape, and delegated-payment requirements, see the
[checkout API reference](https://developers.openai.com/plugins/build/monetization).