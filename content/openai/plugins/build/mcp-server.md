# Build an MCP server

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

Add an MCP server when a plugin use case needs live data, authentication,
controlled actions, or code that runs on infrastructure you operate. The
server defines the tools available to ChatGPT and Codex. It does not need to
return custom UI.

Start from the supported goals in your
[use-case inventory](https://developers.openai.com/plugins/plan/use-case). Each tool should help complete a
recognizable user goal and should expose only the data and actions required for
that goal.

Build the tools first. After the server works without custom UI, you can [add
  UI to the MCP server](https://developers.openai.com/plugins/build/chatgpt-ui) for workflows that need
  visual interaction.

## Choose an MCP software development kit

The official software development kits provide schema helpers, server scaffolding, and streamable
HTTP transport:

- [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk),
  published as `@modelcontextprotocol/sdk`.
- [Python SDK](https://github.com/modelcontextprotocol/python-sdk), published
  as `mcp`.

Install the SDK that matches your server stack:

```bash
# TypeScript
npm install @modelcontextprotocol/sdk zod

# Python
pip install mcp
```




For the Ruby examples, use Ruby 3.3 and install `mcp` version `1.10.0`, `rackup` version `2.3`, and `webrick`. Save one complete server example as `main.rb`, then run `ruby main.rb`. It listens for Streamable HTTP requests at `http://localhost:3000/mcp`.







For the Go examples, use Go 1.26 or later and install `github.com/modelcontextprotocol/go-sdk` version `v1.8.0` in your Go module. Save one complete server example as `main.go`, then run `go run .`. It listens for Streamable HTTP requests at `http://localhost:3000/mcp`.




## Create the server

Create an MCP server with a stable name and version:

Create an MCP server

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({
  name: "acme-projects",
  version: "1.0.0",
});
```

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()
	server := mcp.NewServer(&mcp.Implementation{Name: "acme-projects", Version: "1.0.0"}, nil)
	handler := mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, &mcp.StreamableHTTPOptions{Stateless: true, JSONResponse: true})
	mux := http.NewServeMux()
	mux.Handle("/mcp", handler)
	port := "3000"
	listener, err := net.Listen("tcp", "127.0.0.1:"+port)
	if err != nil {
		log.Fatal(err)
	}
	httpServer := &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	fmt.Println("Listening on http://" + listener.Addr().String())
	shutdownDone := make(chan struct{})
	go func() {
		defer close(shutdownDone)
		<-ctx.Done()
		shutdownCtx, stop := context.WithTimeout(context.Background(), 5*time.Second)
		defer stop()
		if err := httpServer.Shutdown(shutdownCtx); err != nil {
			log.Print(err)
		}
	}()
	if err := httpServer.Serve(listener); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
	<-shutdownDone
}
```

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

// From src/content/platform/dotnet-examples:
// dotnet run -- apps-sdk-mcp-server-create-the-server
// MCP endpoint: http://localhost:3000/mcp

WebApplicationBuilder builder = WebApplication.CreateBuilder();
builder.WebHost.UseUrls("http://127.0.0.1:3000");
builder.Logging.ClearProviders();

builder.Services
    .AddMcpServer(options =>
    {
        options.ServerInfo = new Implementation
        {
            Name = "acme-projects",
            Version = "1.0.0",
        };
    })
    .WithHttpTransport(options => options.Stateless = true)
    .WithListToolsHandler((request, cancellationToken) =>
        ValueTask.FromResult(new ListToolsResult { Tools = [] }));

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
require "rackup"
require "rackup/handler/webrick"

server = MCP::Server.new(
  name: "acme-projects",
  version: "1.0.0"
)
transport = MCP::Server::Transports::StreamableHTTPTransport.new(server, stateless: true, enable_json_response: true)
app = Rack::URLMap.new("/mcp" => transport)
Rackup::Handler::WEBrick.run(app, Host: "127.0.0.1", Port: Integer(ENV.fetch("PORT", "3000"))) do |http_server|
  trap("INT") { http_server.shutdown }
  trap("TERM") { http_server.shutdown }
end
```


MCP servers can also return an
[`instructions` field](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle#initialization)
during initialization. ChatGPT and Codex use these instructions alongside tool
metadata.

Use server instructions for guidance that applies across tools, such as
required tool sequences or shared rate limits. Keep the most important details
in the first 512 characters. Do not repeat every tool description or try to
change the model's personality.

Add server instructions

```typescript
const server = new McpServer(
  { name: "acme-projects", version: "1.0.0" },
  {
    instructions:
      "Before updating a project, call get_project to confirm its ID and current status.",
  }
);
```

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()
	server := mcp.NewServer(&mcp.Implementation{Name: "acme-projects", Version: "1.0.0"}, &mcp.ServerOptions{Instructions: "Before updating a project, call get_project to confirm its ID and current status."})
	handler := mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, &mcp.StreamableHTTPOptions{Stateless: true, JSONResponse: true})
	mux := http.NewServeMux()
	mux.Handle("/mcp", handler)
	port := "3000"
	listener, err := net.Listen("tcp", "127.0.0.1:"+port)
	if err != nil {
		log.Fatal(err)
	}
	httpServer := &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	fmt.Println("Listening on http://" + listener.Addr().String())
	shutdownDone := make(chan struct{})
	go func() {
		defer close(shutdownDone)
		<-ctx.Done()
		shutdownCtx, stop := context.WithTimeout(context.Background(), 5*time.Second)
		defer stop()
		if err := httpServer.Shutdown(shutdownCtx); err != nil {
			log.Print(err)
		}
	}()
	if err := httpServer.Serve(listener); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
	<-shutdownDone
}
```

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

// From src/content/platform/dotnet-examples:
// dotnet run -- apps-sdk-mcp-server-create-the-server-2
// MCP endpoint: http://localhost:3000/mcp

WebApplicationBuilder builder = WebApplication.CreateBuilder();
builder.WebHost.UseUrls("http://127.0.0.1:3000");
builder.Logging.ClearProviders();

builder.Services
    .AddMcpServer(options =>
    {
        options.ServerInfo = new Implementation
        {
            Name = "acme-projects",
            Version = "1.0.0",
        };
        options.ServerInstructions = "Before updating a project, call get_project to confirm its ID and current status.";
    })
    .WithHttpTransport(options => options.Stateless = true)
    .WithListToolsHandler((request, cancellationToken) =>
        ValueTask.FromResult(new ListToolsResult { Tools = [] }));

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
require "rackup"
require "rackup/handler/webrick"

server = MCP::Server.new(
  name: "acme-projects",
  version: "1.0.0",
  instructions: "Before updating a project, call get_project to confirm its ID and current status."
)
transport = MCP::Server::Transports::StreamableHTTPTransport.new(server, stateless: true, enable_json_response: true)
app = Rack::URLMap.new("/mcp" => transport)
Rackup::Handler::WEBrick.run(app, Host: "127.0.0.1", Port: Integer(ENV.fetch("PORT", "3000"))) do |http_server|
  trap("INT") { http_server.shutdown }
  trap("TERM") { http_server.shutdown }
end
```


## Define tools from user goals

Create one tool for each distinct action the plugin must support. Prefer
focused operations such as `list_projects`, `get_project`, and
`update_project` over one tool with many unrelated modes.

Each tool needs:

- An action-oriented name and human-readable title.
- A description that explains when to use it.
- An explicit input schema.
- An output schema when the tool returns structured data.
- Accurate safety annotations.
- A handler that authorizes the request and performs the operation.

The model uses this metadata to decide whether and how to call the tool. Treat
names, descriptions, schemas, and annotations as part of the plugin's
user-facing behavior.

Define a project lookup tool

```typescript
// Local example data; replace this with your application data.
const projects = [
  { id: "project-1", name: "Website", status: "active" },
  { id: "project-2", name: "Launch", status: "archived" },
];
async function listProjects({ status }: { status?: string }) {
  return projects.filter((project) => !status || project.status === status);
}

import { z } from "zod";

server.registerTool(
  "list_projects",
  {
    title: "List projects",
    description:
      "Use this when the user wants to find or review projects in their Acme workspace.",
    inputSchema: {
      status: z.enum(["active", "archived"]).optional(),
    },
    outputSchema: {
      projects: z.array(
        z.object({
          id: z.string(),
          name: z.string(),
          status: z.string(),
        })
      ),
    },
    annotations: {
      readOnlyHint: true,
      openWorldHint: false,
      destructiveHint: false,
    },
  },
  async ({ status }) => {
    const projects = await listProjects({ status });

    return {
      structuredContent: { projects },
      content: [
        {
          type: "text",
          text: `Found ${projects.length} projects.`,
        },
      ],
    };
  }
);
```

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

type listProjectsInput struct {
	Status string `json:"status,omitempty"`
}
type project struct {
	ID     string `json:"id"`
	Name   string `json:"name"`
	Status string `json:"status"`
}
type listProjectsOutput struct {
	Projects []project `json:"projects"`
}

func main() {
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	closed := false
	mcp.AddTool(server, &mcp.Tool{
		Name:        "list_projects",
		Description: "Use this when the user wants to find or review projects in their Acme workspace.",
		InputSchema: map[string]any{"type": "object", "properties": map[string]any{
			"status": map[string]any{"type": "string", "enum": []string{"active", "archived"}},
		}},
		Annotations: &mcp.ToolAnnotations{ReadOnlyHint: true, OpenWorldHint: &closed, DestructiveHint: &closed},
	}, func(_ context.Context, _ *mcp.CallToolRequest, input listProjectsInput) (*mcp.CallToolResult, listProjectsOutput, error) {
		// Local example data; replace this with your application data.
		projects := []project{{ID: "project-1", Name: "Website", Status: "active"}, {ID: "project-2", Name: "Launch", Status: "archived"}}
		selected := []project{}
		for _, p := range projects {
			if input.Status == "" || input.Status == p.Status {
				selected = append(selected, p)
			}
		}
		result := &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: fmt.Sprintf("Found %d projects.", len(selected))}}}
		return result, listProjectsOutput{Projects: selected}, nil
	})
	handler := mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, &mcp.StreamableHTTPOptions{Stateless: true, JSONResponse: true})
	mux := http.NewServeMux()
	mux.Handle("/mcp", handler)
	port := "3000"
	listener, err := net.Listen("tcp", "127.0.0.1:"+port)
	if err != nil {
		log.Fatal(err)
	}
	httpServer := &http.Server{Handler: mux, ReadHeaderTimeout: 5 * time.Second}
	fmt.Println("Listening on http://" + listener.Addr().String())
	shutdownDone := make(chan struct{})
	go func() {
		defer close(shutdownDone)
		<-ctx.Done()
		shutdownCtx, stop := context.WithTimeout(context.Background(), 5*time.Second)
		defer stop()
		if err := httpServer.Shutdown(shutdownCtx); err != nil {
			log.Print(err)
		}
	}()
	if err := httpServer.Serve(listener); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
	<-shutdownDone
}
```

```csharp
using System.Text.Json;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Protocol;
using ModelContextProtocol.Server;

// From src/content/platform/dotnet-examples:
// dotnet run -- apps-sdk-mcp-server-define-tools-from-user-goals
// MCP endpoint: http://localhost:3000/mcp

WebApplicationBuilder builder = WebApplication.CreateBuilder();
builder.WebHost.UseUrls("http://127.0.0.1:3000");
builder.Logging.ClearProviders();

builder.Services
    .AddMcpServer(options =>
    {
        options.ServerInfo = new Implementation
        {
            Name = "docs-example",
            Version = "1.0.0",
        };
    })
    .WithHttpTransport(options => options.Stateless = true)
    .WithListToolsHandler((request, cancellationToken) =>
        ValueTask.FromResult(new ListToolsResult
        {
            Tools =
            [
                new Tool
                {
                    Name = "list_projects",
                    Description = "Use this when the user wants to find or review projects in their Acme workspace.",
                    InputSchema = JsonSerializer.SerializeToElement(new
                    {
                        type = "object",
                        properties = new
                        {
                            status = new
                            {
                                type = "string",
                                @enum = new List<string> { "active", "archived" },
                            },
                        },
                    }),
                    Annotations = JsonSerializer.Deserialize<ToolAnnotations>(
                        """
                        {"readOnlyHint":true,"openWorldHint":false,"destructiveHint":false}
                        """)!,
                },
            ],
        }))
    .WithCallToolHandler((request, cancellationToken) =>
    {
        if (request.Params is not { Name: "list_projects" } parameters)
        {
            throw new ArgumentException("Unknown tool.", nameof(request));
        }

        string? status = null;
        if (parameters.Arguments is { } arguments
            && arguments.TryGetValue("status", out JsonElement value))
        {
            if (value.ValueKind != JsonValueKind.String
                || value.GetString() is not ("active" or "archived"))
            {
                return ValueTask.FromResult(new CallToolResult
                {
                    IsError = true,
                    Content = [new TextContentBlock
                    {
                        Text = "status must be active or archived when provided",
                    }],
                });
            }
            status = value.GetString();
        }

        // Local example data; replace this with your application data.
        var projects = new[]
        {
            new { id = "project-1", name = "Website", status = "active" },
            new { id = "project-2", name = "Launch", status = "archived" },
        };
        var selected = projects
            .Where(project => status is null || project.status == status)
            .ToArray();
        return ValueTask.FromResult(new CallToolResult
        {
            Content = [new TextContentBlock
            {
                Text = $"Found {selected.Length} projects.",
            }],
            StructuredContent = JsonSerializer.SerializeToElement(new
            {
                projects = selected,
            }),
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
require "rackup"
require "rackup/handler/webrick"

class ListProjects < MCP::Tool
  tool_name "list_projects"
  description "Use this when the user wants to find or review projects in their Acme workspace."
  input_schema(
    properties: {
      status: {
        type: "string",
        enum: ["active", "archived"]
      }
    }
  )
  output_schema(
    type: "object",
    properties: {
      projects: {
        type: "array",
        items: {
          type: "object",
          properties: {
            id: { type: "string" },
            name: { type: "string" },
            status: { type: "string" }
          },
          required: ["id", "name", "status"]
        }
      }
    },
    required: ["projects"]
  )
  annotations(read_only_hint: true, open_world_hint: false, destructive_hint: false, idempotent_hint: nil)

  def self.call(server_context:, status: nil)
    # Local example data; replace this with your application data.
    projects = [
      {
        id: "project-1",
        name: "Website",
        status: "active"
      },
      {
        id: "project-2",
        name: "Launch",
        status: "archived"
      }
    ].select { |project| status.nil? || project[:status] == status }
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: "Found #{projects.length} projects."
        }
      ],
      structured_content: { projects: projects }
    )
  end
end

server = MCP::Server.new(name: "docs-example", version: "1.0.0", tools: [ListProjects])
transport = MCP::Server::Transports::StreamableHTTPTransport.new(server, stateless: true, enable_json_response: true)
app = Rack::URLMap.new("/mcp" => transport)
Rackup::Handler::WEBrick.run(app, Host: "127.0.0.1", Port: Integer(ENV.fetch("PORT", "3000"))) do |http_server|
  trap("INT") { http_server.shutdown }
  trap("TERM") { http_server.shutdown }
end
```


## Return useful results without UI

A tool result can include:

- `structuredContent`: concise data the model can inspect and use in later
  calls.
- `content`: text or other MCP content that helps the model answer the user.
- `_meta`: client-specific data hidden from the model.

Return enough information for the model to complete the workflow without a
component. Use stable identifiers in structured results so later tools can
refer to the same records.

Do not put secrets, access tokens, or unnecessary personal data in tool
results. Treat `_meta` as hidden from the model, not as a substitute for
authorization or secure storage.

## Authenticate and authorize requests

Add authentication when a tool reads private data or takes action for a user.
Enforce authorization in the MCP server for every request; never rely on the
model to decide whether a user has access.

See [Authenticate users](https://developers.openai.com/plugins/build/auth) for OAuth discovery, security
schemes, and authorization challenges.

To improve the experience of using multiple accounts, expose an authenticated,
read-only profile tool and mark it with `_meta["openai/profile"]: true`.
OpenAI uses the profile information to identify connected accounts consistently
and help users tell them apart. Resolve the profile from the request's validated
credentials, and keep every tool call scoped to those credentials. Users can
connect multiple accounts without a profile tool. See
[Support multiple accounts](https://developers.openai.com/plugins/build/auth#support-multiple-accounts) for
the schema and implementation example.

## Tool annotations and elicitation

Set annotations according to actual behavior:

- `readOnlyHint`: `true` only when the tool cannot change state.
- `destructiveHint`: `true` when a tool can cause irreversible or difficult to
  reverse outcomes.
- `openWorldHint`: `true` when a tool accesses the public internet or open-ended
  external entities, including through read-only actions such as web search.
  A tool limited to a bounded private account or workspace can set this to
  `false`, even when that service is externally hosted.

Annotations help ChatGPT and Codex choose appropriate confirmation and safety
behavior. They do not replace authorization, validation, or confirmation in
your server.

Use MCP elicitation when the server needs structured information that was not
provided in the original tool call. Keep elicitation focused on information
the user can reasonably supply. Do not use it to collect secrets or bypass
normal authentication.

## Company knowledge compatibility

Company knowledge can use read-only tools from your MCP server. To make a
plugin eligible as a company knowledge source, implement the standard
`search` and `fetch` tool input schemas and mark other read-only tools with
`readOnlyHint: true`.

Return absolute, user-openable URLs for sources that the model should cite. Keep
internal document identifiers in the result's `id` field. For the required
schemas and result shapes, see
[Building MCP servers for ChatGPT and API integrations](https://platform.openai.com/docs/mcp).

## Run and test locally

Expose a streamable HTTP endpoint, typically at `/mcp`, then inspect it with
[MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector):

```bash
npx @modelcontextprotocol/inspector
```

In the Inspector UI, select **Streamable HTTP** and enter
`http://localhost:3000/mcp`.

Use the inspector to:

1. Confirm that initialization succeeds.
2. Review server instructions and the advertised tool list.
3. Call every tool with representative and invalid inputs.
4. Verify schemas, results, errors, and annotations.
5. Confirm that authorization is enforced for private data and write actions.

Then [connect the custom MCP server to ChatGPT](https://developers.openai.com/plugins/deploy/connect-chatgpt) and run the direct,
indirect, edge-case, and out-of-scope requests from your use-case inventory.

## Deploy the endpoint

For public plugin submission, deploy the MCP server at a stable, publicly
reachable HTTPS endpoint. [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
can connect a private MCP server in ChatGPT, but it does not satisfy
public submission requirements.

The production endpoint must:

- Support the MCP streamable HTTP transport.
- Respond at a stable URL, typically ending in `/mcp`.
- Meet the latency and availability needs of the plugin's workflows.
- Reach required services and data stores.
- Preserve authentication and authorization boundaries.
- Produce logs and metrics for failed initialization and tool calls.

If the MCP server must remain private, deploy a public HTTPS proxy that forwards
MCP requests to the private server. Use
[OpenAI-managed mTLS](https://developers.openai.com/plugins/build/auth#mutual-tls-mtls) to authenticate
ChatGPT as the MCP client, and use [OAuth 2.1](https://developers.openai.com/plugins/build/auth) when your
plugin requires user authentication. If your network requires an IP allowlist,
use the published [ChatGPT connectors IP ranges](https://developers.openai.com/api/docs/guides/ip-addresses)
and update the allowlist automatically. An IP allowlist does not replace
authentication or authorization.

The public endpoint must remain reachable for plugin review and
[domain verification](https://developers.openai.com/plugins/deploy/submission#domain-verification). Do not
use Secure MCP Tunnel alone, a temporary tunnel, or a local endpoint for public
submission.

### Choose infrastructure

You can deploy the MCP server to serverless, container, edge, or traditional
application infrastructure. Choose a platform based on:

- Runtime and dependency support.
- Streaming response behavior.
- Cold-start and request latency.
- Network access to required services.
- Data residency and compliance requirements.
- Secret management.
- Logging, tracing, and alerting.
- Rollback and versioning support.

If the server also hosts optional UI assets, deploy those assets at stable
origins allowed by the component's
[content security policy](https://developers.openai.com/plugins/build/chatgpt-ui#content-security-policy-csp).

### Configure the production endpoint

Before deployment:

1. Set production credentials through the host's secret-management system.
2. Configure the authorization server and allowed redirect behavior.
3. Apply timeouts and rate limits to expensive or externally visible tools.
4. Remove debug responses and unnecessary personal data.
5. Confirm that logs do not contain access tokens or sensitive tool results.

After deployment, call the production endpoint with
[MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector). Verify
initialization, server instructions, tools, schemas, annotations,
authentication, results, and errors.

### Plan for updates

Keep published tool names and schemas backward compatible. Add fields or tools
without breaking existing contracts. If metadata changes, refresh the
custom MCP server connection and rerun the evaluation set before submission.

For optional UI, version resource identifiers when HTML, JavaScript, or CSS
changes in a way that could break a cached component.

<a id="supercharge-your-mcp-with-extensions"></a>

## Add MCP extensions

OpenAI MCP Extensions connect your plugin to ChatGPT's sidebar, composer, and
file viewers. See [Plugin Extensions](https://developers.openai.com/plugins/build/extensions) for examples
and setup.

## Security reminders

- Treat every tool input as untrusted.
- Validate parameters and enforce authorization on the server.
- Require confirmation for consequential write actions.
- Keep secrets and sensitive data out of tool metadata and results.
- Log enough context to investigate failures without logging credentials or
  unnecessary personal data.
- Rate-limit expensive or externally visible actions.