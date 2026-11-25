# MCP server and UI quickstart

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

## Introduction

Plugins use the [Model Context Protocol
(MCP)](https://developers.openai.com/plugins/concepts/mcp-server) to expose server-backed capabilities to
ChatGPT and Codex. This tutorial uses:

1. An MCP server that defines tools and exposes them to ChatGPT and Codex.
2. An optional web component, rendered in an iframe inside ChatGPT.

ChatGPT implements the open MCP Apps UI standard so you can build your UI once
and run it across MCP Apps-compatible hosts.

In this quickstart, we'll build a basic to-do workflow with UI contained in a
single HTML file that keeps the markup, CSS, and JavaScript together.

To see more advanced examples using React, see the [examples repository on GitHub](https://github.com/openai/openai-apps-sdk-examples).

## Build a web component

This step is optional. If you only need tools and no ChatGPT UI, skip to
  [Build an MCP server](#build-an-mcp-server) and do not register a UI resource.

Start by creating a file called `public/todo-widget.html` in a new directory.
ChatGPT will render this UI when the associated MCP tool returns it.
This file will contain the web component that will be rendered in the ChatGPT interface.

Add the following content:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Todo list</title>
    <style>
      :root {
        color: #0b0b0f;
        font-family:
          "Inter",
          system-ui,
          -apple-system,
          sans-serif;
      }

      html,
      body {
        width: 100%;
        min-height: 100%;
        box-sizing: border-box;
      }

      body {
        margin: 0;
        padding: 16px;
        background: #f6f8fb;
      }

      main {
        width: 100%;
        max-width: 360px;
        min-height: 260px;
        margin: 0 auto;
        background: #fff;
        border-radius: 16px;
        padding: 20px;
        box-shadow: 0 12px 24px rgba(15, 23, 42, 0.08);
      }

      h2 {
        margin: 0 0 16px;
        font-size: 1.25rem;
      }

      form {
        display: flex;
        gap: 8px;
        margin-bottom: 16px;
      }

      form input {
        flex: 1;
        padding: 10px 12px;
        border-radius: 10px;
        border: 1px solid #cad3e0;
        font-size: 0.95rem;
      }

      form button {
        border: none;
        border-radius: 10px;
        background: #111bf5;
        color: white;
        font-weight: 600;
        padding: 0 16px;
        cursor: pointer;
      }

      form button:disabled {
        opacity: 0.7;
        cursor: not-allowed;
      }

      input[type="checkbox"] {
        accent-color: #111bf5;
      }

      ul {
        list-style: none;
        padding: 0;
        margin: 0;
        display: flex;
        flex-direction: column;
        gap: 8px;
      }

      li {
        background: #f2f4fb;
        border-radius: 12px;
        padding: 10px 14px;
        display: flex;
        align-items: center;
        gap: 10px;
      }

      li span {
        flex: 1;
      }

      li[data-completed="true"] span {
        text-decoration: line-through;
        color: #6c768a;
      }

      li[data-busy="true"] {
        opacity: 0.7;
      }
    </style>
  </head>
  <body>
    <main>
      <h2>Todo list</h2>
      <form id="add-form" autocomplete="off">
        <input id="todo-input" name="title" placeholder="Add a task" />
        <button type="submit">Add</button>
      </form>
      <ul id="todo-list"></ul>
    </main>

    <script type="module">
      const listEl = document.querySelector("#todo-list");
      const formEl = document.querySelector("#add-form");
      const inputEl = document.querySelector("#todo-input");
      const addButtonEl = formEl.querySelector('button[type="submit"]');
      const addButtonText = addButtonEl.textContent;

      let tasks = [];
      let isAdding = false;
      const busyTodoIds = new Set();

      const render = () => {
        listEl.innerHTML = "";
        tasks.forEach((task) => {
          const li = document.createElement("li");
          li.dataset.id = task.id;
          li.dataset.completed = String(Boolean(task.completed));
          li.dataset.busy = String(busyTodoIds.has(task.id));

          const label = document.createElement("label");
          label.style.display = "flex";
          label.style.alignItems = "center";
          label.style.gap = "10px";

          const checkbox = document.createElement("input");
          checkbox.type = "checkbox";
          checkbox.checked = Boolean(task.completed);
          checkbox.disabled = busyTodoIds.has(task.id);

          const span = document.createElement("span");
          span.textContent = task.title;

          label.appendChild(checkbox);
          label.appendChild(span);
          li.appendChild(label);
          listEl.appendChild(li);
        });
      };

      const updateFromResponse = (response) => {
        if (response?.structuredContent?.tasks) {
          tasks = response.structuredContent.tasks;
          render();
        }
      };

      // MCP Apps standard bridge: JSON-RPC messages over postMessage.
      //
      // - Initialize the bridge with `ui/initialize`.
      // - Confirm readiness with `ui/notifications/initialized`.
      // - Call tools with `tools/call`.
      // - Listen for `ui/notifications/tool-result` to react to model-initiated tool calls.
      let rpcId = 0;
      const pendingRequests = new Map();

      const rpcNotify = (method, params) => {
        window.parent.postMessage({ jsonrpc: "2.0", method, params }, "*");
      };

      const rpcRequest = (method, params) =>
        new Promise((resolve, reject) => {
          const id = ++rpcId;
          pendingRequests.set(id, { resolve, reject });
          window.parent.postMessage(
            { jsonrpc: "2.0", id, method, params },
            "*"
          );
        });

      window.addEventListener(
        "message",
        (event) => {
          if (event.source !== window.parent) return;
          const message = event.data;
          if (!message || message.jsonrpc !== "2.0") return;

          // Responses
          if (typeof message.id === "number") {
            const pending = pendingRequests.get(message.id);
            if (!pending) return;
            pendingRequests.delete(message.id);

            if (message.error) {
              pending.reject(message.error);
              return;
            }

            pending.resolve(message.result);
            return;
          }

          // Notifications
          if (typeof message.method !== "string") return;
          if (message.method === "ui/notifications/tool-result") {
            updateFromResponse(message.params);
          }
        },
        { passive: true }
      );

      const initializeBridge = async () => {
        const appInfo = { name: "todo-widget", version: "0.1.0" };
        const appCapabilities = {};
        const protocolVersion = "2026-01-26";

        try {
          await rpcRequest("ui/initialize", {
            appInfo,
            appCapabilities,
            protocolVersion,
          });
          rpcNotify("ui/notifications/initialized", {});
        } catch (error) {
          console.error("Failed to initialize the MCP Apps bridge:", error);
          throw error;
        }
      };

      const bridgeReady = initializeBridge();

      const callTodoTool = async (name, payload) => {
        await bridgeReady;
        const response = await rpcRequest("tools/call", {
          name,
          arguments: payload,
        });
        updateFromResponse(response);
      };

      formEl.addEventListener("submit", async (event) => {
        event.preventDefault();
        const title = inputEl.value.trim();
        if (!title || isAdding) return;

        isAdding = true;
        addButtonEl.disabled = true;
        addButtonEl.textContent = "Adding…";

        try {
          await callTodoTool("add_todo", { title });
          inputEl.value = "";
        } catch (error) {
          console.error("Failed to add todo:", error);
        } finally {
          isAdding = false;
          addButtonEl.disabled = false;
          addButtonEl.textContent = addButtonText;
        }
      });

      listEl.addEventListener("change", async (event) => {
        const checkbox = event.target;
        if (!checkbox.matches('input[type="checkbox"]')) return;
        const id = checkbox.closest("li")?.dataset.id;
        if (!id) return;

        if (!checkbox.checked) {
          checkbox.checked = true;
          return;
        }

        if (busyTodoIds.has(id)) return;
        busyTodoIds.add(id);
        checkbox.disabled = true;
        const rowEl = checkbox.closest("li");
        if (rowEl) rowEl.dataset.busy = "true";

        try {
          await callTodoTool("complete_todo", { id });
        } catch (error) {
          console.error("Failed to complete todo:", error);
        } finally {
          busyTodoIds.delete(id);
          render();
        }
      });

      render();
    </script>
  </body>
</html>
```


### Use MCP Apps in your web component

For new UI, use the MCP Apps host bridge: JSON-RPC over `postMessage`
with `ui/*` notifications and methods such as `tools/call`.

After the shared MCP Apps flow works, add optional ChatGPT extensions through
`window.openai` only when you need capabilities the standard does not cover.
For details, see [Add UI to your MCP
server](https://developers.openai.com/plugins/build/chatgpt-ui#layer-on-chatgpt-extensions).

## Build an MCP server

Use an MCP SDK to create a server and expose a `/mcp` endpoint.

The server implementations use the same HTML page.




With Go 1.26 or later, initialize a module and add `github.com/modelcontextprotocol/go-sdk/mcp` at version `v1.8.0`. Save the example as `main.go`, keep `public/todo-widget.html` in the same project directory, and run `go run .`. Edit the `port` literal to choose another port.







For Ruby 3.3, install the `mcp` (1.10.0), `rackup` (2.3.1), and `webrick` (1.9.2) gems.




If you're using Python, refer to our [examples repository on GitHub](https://github.com/openai/openai-apps-sdk-examples) to see an example MCP server with the Python SDK.




The TypeScript setup uses the [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk).

Use Node.js 22.6 or later. Install the TypeScript SDK, MCP Apps helpers, and the `zod` package with:

```bash
npm install @modelcontextprotocol/sdk @modelcontextprotocol/ext-apps zod
```




### MCP server with UI resources

Register a resource for your component bundle and the tools the model can call (for example, `add_todo` and `complete_todo`) so ChatGPT can drive the UI.




For Java, add these tested Maven dependencies to your project: `io.modelcontextprotocol.sdk:mcp-core:1.1.5`, `io.modelcontextprotocol.sdk:mcp-json-jackson2:1.1.5`, `org.eclipse.jetty.ee10:jetty-ee10-servlet:12.0.41`, and `com.fasterxml.jackson.core:jackson-databind:2.18.11`. Use a JDK supported by [the Jetty compatibility table](https://jetty.org/docs/jetty/12/index.html). Put the imports before the server class and place the remaining statements in its `main` method, which must handle or declare checked exceptions. The Java server reads `public/todo-widget.html` from the working directory and serves the same widget and tools:







Save the TypeScript example as `server.ts`.







Save the Ruby example as `server.rb`.




Keep `public/todo-widget.html` in the same project directory:

Create an MCP server with a widget

```typescript
import { createServer } from "node:http";
import { readFileSync } from "node:fs";
import {
  registerAppResource,
  registerAppTool,
  RESOURCE_MIME_TYPE,
} from "@modelcontextprotocol/ext-apps/server";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { z } from "zod";

const todoHtml = readFileSync("public/todo-widget.html", "utf8");

const addTodoInputSchema = {
  title: z.string().min(1),
};

const completeTodoInputSchema = {
  id: z.string().min(1),
};

const todoOutputSchema = {
  tasks: z.array(
    z.object({
      id: z.string(),
      title: z.string(),
      completed: z.boolean(),
    })
  ),
};

let todos: { id: string; title: string; completed: boolean }[] = [];
let nextId = 1;

const replyWithTodos = (message: string) => ({
  content: message ? [{ type: "text" as const, text: message }] : [],
  structuredContent: { tasks: todos },
});

function createTodoServer() {
  const server = new McpServer({
    name: "todo-plugin-server",
    version: "0.1.0",
  });

  registerAppResource(
    server,
    "todo-widget",
    "ui://widget/todo.html",
    {},
    async () => ({
      contents: [
        {
          uri: "ui://widget/todo.html",
          mimeType: RESOURCE_MIME_TYPE,
          text: todoHtml,
        },
      ],
    })
  );

  registerAppTool(
    server,
    "add_todo",
    {
      title: "Add todo",
      description: "Creates a todo item with the given title.",
      inputSchema: addTodoInputSchema,
      outputSchema: todoOutputSchema,
      _meta: {
        ui: { resourceUri: "ui://widget/todo.html" },
      },
    },
    async (args) => {
      const title = args?.title?.trim?.() ?? "";
      if (!title) return replyWithTodos("Missing title.");
      const todo = { id: `todo-${nextId++}`, title, completed: false };
      todos = [...todos, todo];
      return replyWithTodos(`Added "${todo.title}".`);
    }
  );

  registerAppTool(
    server,
    "complete_todo",
    {
      title: "Complete todo",
      description: "Marks a todo as done by id.",
      inputSchema: completeTodoInputSchema,
      outputSchema: todoOutputSchema,
      _meta: {
        ui: { resourceUri: "ui://widget/todo.html" },
      },
    },
    async (args) => {
      const id = args?.id;
      if (!id) return replyWithTodos("Missing todo id.");
      const todo = todos.find((task) => task.id === id);
      if (!todo) {
        return replyWithTodos(`Todo ${id} was not found.`);
      }

      todos = todos.map((task) =>
        task.id === id ? { ...task, completed: true } : task
      );

      return replyWithTodos(`Completed "${todo.title}".`);
    }
  );

  return server;
}

const port = Number(process.env.PORT ?? 8787);
const MCP_PATH = "/mcp";

const httpServer = createServer(async (req, res) => {
  if (!req.url) {
    res.writeHead(400).end("Missing URL");
    return;
  }

  const url = new URL(req.url, `http://${req.headers.host ?? "localhost"}`);

  if (req.method === "OPTIONS" && url.pathname === MCP_PATH) {
    res.writeHead(204, {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "POST, GET, OPTIONS",
      "Access-Control-Allow-Headers": "content-type, mcp-session-id",
      "Access-Control-Expose-Headers": "Mcp-Session-Id",
    });
    res.end();
    return;
  }

  if (req.method === "GET" && url.pathname === "/") {
    res.writeHead(200, { "content-type": "text/plain" }).end("Todo MCP server");
    return;
  }

  const MCP_METHODS = new Set(["POST", "GET", "DELETE"]);
  if (url.pathname === MCP_PATH && req.method && MCP_METHODS.has(req.method)) {
    res.setHeader("Access-Control-Allow-Origin", "*");
    res.setHeader("Access-Control-Expose-Headers", "Mcp-Session-Id");

    const server = createTodoServer();
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined, // stateless mode
      enableJsonResponse: true,
    });

    res.on("close", () => {
      transport.close();
      server.close();
    });

    try {
      await server.connect(transport);
      await transport.handleRequest(req, res);
    } catch (error) {
      console.error("Error handling MCP request:", error);
      if (!res.headersSent) {
        res.writeHead(500).end("Internal server error");
      }
    }
    return;
  }

  res.writeHead(404).end("Not Found");
});

httpServer.listen(port, () => {
  console.log(
    `Todo MCP server listening on http://localhost:${port}${MCP_PATH}`
  );
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
	"strings"
	"sync"
	"syscall"
	"time"
)

type task struct {
	ID        string `json:"id"`
	Title     string `json:"title"`
	Completed bool   `json:"completed"`
}
type todoOutput struct {
	Tasks []task `json:"tasks"`
}
type addInput struct {
	Title string `json:"title"`
}
type completeInput struct {
	ID string `json:"id"`
}

func main() {
	html, err := os.ReadFile("public/todo-widget.html")
	if err != nil {
		log.Fatal(err)
	}
	server := mcp.NewServer(&mcp.Implementation{Name: "todo-plugin-server", Version: "0.1.0"}, nil)
	// This local demonstration keeps tasks in memory until the server exits.
	var mu sync.Mutex
	tasks := []task{}
	nextID := 1
	reply := func(message string) (*mcp.CallToolResult, todoOutput, error) {
		snapshot := append([]task{}, tasks...)
		return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: message}}}, todoOutput{Tasks: snapshot}, nil
	}
	schema := func(field string) map[string]any {
		return map[string]any{"type": "object", "properties": map[string]any{field: map[string]any{"type": "string", "minLength": 1}}, "required": []string{field}, "additionalProperties": false}
	}
	meta := mcp.Meta{"ui": map[string]any{"resourceUri": "ui://widget/todo.html"}}
	mcp.AddTool(server, &mcp.Tool{Name: "add_todo", Title: "Add todo", Description: "Creates a todo item with the given title.", InputSchema: schema("title"), Meta: meta}, func(_ context.Context, _ *mcp.CallToolRequest, input addInput) (*mcp.CallToolResult, todoOutput, error) {
		mu.Lock()
		defer mu.Unlock()
		title := strings.TrimSpace(input.Title)
		if title == "" {
			return reply("Missing title.")
		}
		tasks = append(tasks, task{ID: fmt.Sprintf("todo-%d", nextID), Title: title})
		nextID++
		return reply(fmt.Sprintf("Added %q.", title))
	})
	mcp.AddTool(server, &mcp.Tool{Name: "complete_todo", Title: "Complete todo", Description: "Marks a todo as done by id.", InputSchema: schema("id"), Meta: meta}, func(_ context.Context, _ *mcp.CallToolRequest, input completeInput) (*mcp.CallToolResult, todoOutput, error) {
		mu.Lock()
		defer mu.Unlock()
		for i := range tasks {
			if tasks[i].ID == input.ID {
				tasks[i].Completed = true
				return reply(fmt.Sprintf("Completed %q.", tasks[i].Title))
			}
		}
		return reply(fmt.Sprintf("Todo %s was not found.", input.ID))
	})
	server.AddResource(&mcp.Resource{URI: "ui://widget/todo.html", Name: "todo-widget", MIMEType: "text/html;profile=mcp-app"}, func(context.Context, *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: "ui://widget/todo.html", MIMEType: "text/html;profile=mcp-app", Text: string(html)}}}, nil
	})
	transport := mcp.NewStreamableHTTPHandler(func(*http.Request) *mcp.Server { return server }, &mcp.StreamableHTTPOptions{Stateless: true, JSONResponse: true})
	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path == "/" && r.Method == http.MethodGet {
			w.Header().Set("Content-Type", "text/plain")
			fmt.Fprint(w, "Todo MCP server")
			return
		}
		if r.URL.Path != "/mcp" {
			http.NotFound(w, r)
			return
		}
		w.Header().Set("Access-Control-Allow-Origin", "*")
		w.Header().Set("Access-Control-Expose-Headers", "Mcp-Session-Id")
		if r.Method == http.MethodOptions {
			w.Header().Set("Access-Control-Allow-Methods", "POST, GET, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "content-type, mcp-session-id, mcp-protocol-version")
			w.WriteHeader(http.StatusNoContent)
			return
		}
		transport.ServeHTTP(w, r)
	})
	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer cancel()
	port := "8787"
	listener, err := net.Listen("tcp", "127.0.0.1:"+port)
	if err != nil {
		log.Fatal(err)
	}
	httpServer := &http.Server{Handler: handler, ReadHeaderTimeout: 5 * time.Second}
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

```java
import com.fasterxml.jackson.databind.json.JsonMapper;
import io.modelcontextprotocol.json.jackson2.JacksonMcpJsonMapper;
import io.modelcontextprotocol.server.*;
import io.modelcontextprotocol.server.transport.DefaultServerTransportSecurityValidator;
import io.modelcontextprotocol.server.transport.HttpServletStreamableServerTransportProvider;
import io.modelcontextprotocol.spec.McpSchema;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import org.eclipse.jetty.ee10.servlet.*;
import org.eclipse.jetty.server.*;

var mapper = new JacksonMcpJsonMapper(JsonMapper.builder().build());
    String todoHtml = Files.readString(Path.of("public/todo-widget.html"));
    String todoOutputSchema =
"""
{"type":"object","properties":{"tasks":{"type":"array","items":{"type":"object","properties":{"id":{"type":"string"},"title":{"type":"string"},"completed":{"type":"boolean"}},"required":["id","title","completed"]}}},"required":["tasks"]}
""";
    var todos = new ArrayList<Map<String, Object>>();
    var nextId = new AtomicInteger(1);
    // Permit the local MCP Inspector UI; add other trusted browser origins explicitly.
    var allowedOrigins = List.of("http://localhost:6274", "http://127.0.0.1:6274");
    // Add your public tunnel hostname to this list before connecting ChatGPT.
    var allowedHosts = List.of("localhost:*", "127.0.0.1:*");
    var transport =
        HttpServletStreamableServerTransportProvider.builder()
            .jsonMapper(mapper)
            .mcpEndpoint("/mcp")
            .securityValidator(
                DefaultServerTransportSecurityValidator.builder()
                    .allowedHosts(allowedHosts)
                    .allowedOrigins(allowedOrigins)
                    .build())
            // Sessions close through MCP DELETE or server shutdown; this local demo has no idle
            // expiry.
            .sessionSweepInterval(null)
            .build();
    var resource =
        McpSchema.Resource.builder()
            .name("todo-widget")
            .uri("ui://widget/todo.html")
            .mimeType("text/html;profile=mcp-app")
            .build();
    var server =
        McpServer.sync(transport)
            .serverInfo("todo-plugin-server", "0.1.0")
            .capabilities(
                McpSchema.ServerCapabilities.builder().tools(false).resources(false, false).build())
            .resources(
                new McpServerFeatures.SyncResourceSpecification(
                    resource,
                    (exchange, request) ->
                        new McpSchema.ReadResourceResult(
                            List.of(
                                new McpSchema.TextResourceContents(
                                    resource.uri(), resource.mimeType(), todoHtml)))))
            .toolCall(
                McpSchema.Tool.builder()
                    .name("add_todo")
                    .title("Add todo")
                    .description("Creates a todo item with the given title.")
                    .inputSchema(
                        mapper,
"""
{"type":"object","properties":{"title":{"type":"string","minLength":1}},"required":["title"]}
""")
                    .outputSchema(mapper, todoOutputSchema)
                    .meta(Map.of("ui", Map.of("resourceUri", resource.uri())))
                    .build(),
                (exchange, request) -> {
                  synchronized (todos) {
                    if (!(request.arguments().get("title") instanceof String title)
                        || title.isEmpty())
                      return McpSchema.CallToolResult.builder()
                          .isError(true)
                          .addTextContent("A title string is required.")
                          .build();
                    if (!title.trim().isEmpty())
                      todos.add(
                          Map.of(
                              "id",
                              "todo-" + nextId.getAndIncrement(),
                              "title",
                              title.trim(),
                              "completed",
                              false));
                    return McpSchema.CallToolResult.builder()
                        .structuredContent(Map.of("tasks", List.copyOf(todos)))
                        .isError(false)
                        .build();
                  }
                })
            .toolCall(
                McpSchema.Tool.builder()
                    .name("complete_todo")
                    .title("Complete todo")
                    .description("Marks a todo as done by id.")
                    .inputSchema(
                        mapper,
"""
{"type":"object","properties":{"id":{"type":"string","minLength":1}},"required":["id"]}
""")
                    .outputSchema(mapper, todoOutputSchema)
                    .meta(Map.of("ui", Map.of("resourceUri", resource.uri())))
                    .build(),
                (exchange, request) -> {
                  synchronized (todos) {
                    if (!(request.arguments().get("id") instanceof String id) || id.isEmpty())
                      return McpSchema.CallToolResult.builder()
                          .isError(true)
                          .addTextContent("A todo ID string is required.")
                          .build();
                    for (int i = 0; i < todos.size(); i++) {
                      var todo = todos.get(i);
                      if (id.equals(todo.get("id")))
                        todos.set(
                            i, Map.of("id", id, "title", todo.get("title"), "completed", true));
                    }
                    return McpSchema.CallToolResult.builder()
                        .structuredContent(Map.of("tasks", List.copyOf(todos)))
                        .isError(false)
                        .build();
                  }
                })
            .build();
    var httpServer = new Server();
    var connector = new ServerConnector(httpServer);
    connector.setHost("127.0.0.1");
    connector.setPort(8787);
    httpServer.addConnector(connector);
    var context = new ServletContextHandler();
    context.setContextPath("/");
    context.addServlet(new ServletHolder(transport), "/mcp");
    context.addServlet(
        new ServletHolder(
            new jakarta.servlet.http.HttpServlet() {
              @Override
              protected void doGet(
                  jakarta.servlet.http.HttpServletRequest request,
                  jakarta.servlet.http.HttpServletResponse response)
                  throws java.io.IOException {
                if (!request.getRequestURI().equals("/")) {
                  response.sendError(404);
                  return;
                }
                response.setContentType("text/plain");
                response.getWriter().write("Todo MCP server");
              }
            }),
        "/");
    var cors =
        new FilterHolder(
            (jakarta.servlet.Filter)
                (request, response, chain) -> {
                  var req = (jakarta.servlet.http.HttpServletRequest) request;
                  var res = (jakarta.servlet.http.HttpServletResponse) response;
                  var origins = Collections.list(req.getHeaders("Origin"));
                  if (!origins.isEmpty()
                      && (origins.size() != 1 || !allowedOrigins.contains(origins.get(0)))) {
                    res.sendError(403, "Untrusted Origin");
                    return;
                  }
                  res.setHeader("Vary", "Origin");
                  if (!origins.isEmpty())
                    res.setHeader("Access-Control-Allow-Origin", origins.get(0));
                  res.setHeader("Access-Control-Expose-Headers", "Mcp-Session-Id");
                  if (req.getMethod().equals("OPTIONS")) {
                    res.setHeader("Access-Control-Allow-Methods", "POST, GET, DELETE, OPTIONS");
                    res.setHeader(
                        "Access-Control-Allow-Headers",
                        "content-type, mcp-session-id, mcp-protocol-version");
                    res.setStatus(204);
                    return;
                  }
                  chain.doFilter(request, response);
                });
    cors.setAsyncSupported(true);
    context.addFilter(cors, "/mcp", EnumSet.of(jakarta.servlet.DispatcherType.REQUEST));

    httpServer.setHandler(context);
    try {
      httpServer.start();
      httpServer.join();
    } finally {
      server.closeGracefully();
      httpServer.stop();
    }
```

```ruby
require "mcp"
require "rackup"
require "rackup/handler/webrick"

todo_html = File.read("public/todo-widget.html")
TODO_OUTPUT_SCHEMA = {
  type: "object",

  properties: {
    tasks: {
      type: "array",

      items: {
        type: "object",

        properties: {
          id: { type: "string" },
          title: { type: "string" },
          completed: { type: "boolean" }
        },
        required: ["id", "title", "completed"],

        additionalProperties: false
      }
    }
  },
  required: ["tasks"],

  additionalProperties: false
}.freeze

class TodoList
  def initialize
    @tasks = []
    @next_id = 1
    @mutex = Mutex.new
  end

  def add(title)
    @mutex.synchronize do
      title = title.strip
      next reply("Missing title.") if title.empty?

      @tasks << {
        id: "todo-#{@next_id}",
        title: title,
        completed: false
      }
      @next_id += 1
      reply("Added #{title.inspect}.")
    end
  end

  def complete(id)
    @mutex.synchronize do
      task = @tasks.find { |item| item[:id] == id }
      next reply("Todo #{id} was not found.") unless task

      task[:completed] = true
      reply("Completed #{task[:title].inspect}.")
    end
  end

  private

  def reply(message)
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: message
        }
      ],
      structured_content: { tasks: @tasks.map(&:dup) }
    )
  end
end

class AddTodo < MCP::Tool
  tool_name "add_todo"
  title "Add todo"
  description "Creates a todo item with the given title."
  input_schema(
    properties: {
      title: {
        type: "string",
        minLength: 1
      }
    }, required: ["title"], additionalProperties: false
  )
  output_schema(TODO_OUTPUT_SCHEMA)
  meta(ui: { resourceUri: "ui://widget/todo.html" })

  def self.call(title:, server_context:)
    server_context.fetch(:todos).add(title)
  end
end

class CompleteTodo < MCP::Tool
  tool_name "complete_todo"
  title "Complete todo"
  description "Marks a todo as done by id."
  input_schema(
    properties: {
      id: {
        type: "string",
        minLength: 1
      }
    }, required: ["id"], additionalProperties: false
  )
  output_schema(TODO_OUTPUT_SCHEMA)
  meta(ui: { resourceUri: "ui://widget/todo.html" })

  def self.call(id:, server_context:)
    server_context.fetch(:todos).complete(id)
  end
end

widget = MCP::Resource.define(uri: "ui://widget/todo.html", name: "todo-widget", mime_type: "text/html;profile=mcp-app") do
  [MCP::Resource::TextContents.new(uri: "ui://widget/todo.html", mime_type: "text/html;profile=mcp-app", text: todo_html)]
end
server = MCP::Server.new(
  name: "todo-plugin-server", version: "0.1.0", tools: [AddTodo, CompleteTodo],
  resources: [widget], server_context: { todos: TodoList.new }
)
transport = MCP::Server::Transports::StreamableHTTPTransport.new(server, stateless: true, enable_json_response: true)
mcp_app = Rack::URLMap.new("/mcp" => transport)
app = lambda do |environment|
  path, method = environment.values_at("PATH_INFO", "REQUEST_METHOD")
  headers = {
    "access-control-allow-origin" => "*",
    "access-control-expose-headers" => "Mcp-Session-Id"
  }
  if path == "/mcp" && method == "OPTIONS"
    [
      204, headers.merge(
        "access-control-allow-methods" => "POST, GET, OPTIONS",
        "access-control-allow-headers" => "content-type, mcp-session-id"
      ), []
    ]
  elsif path == "/" && method == "GET"
    [200, { "content-type" => "text/plain" }, ["Todo MCP server"]]
  elsif path == "/mcp" && ["POST", "GET", "DELETE"].include?(method)
    status, response_headers, body = mcp_app.call(environment)
    [status, response_headers.merge(headers), body]
  else
    [404, { "content-type" => "text/plain" }, ["Not Found"]]
  end
end

Rackup::Handler::WEBrick.run(app, Host: "127.0.0.1", Port: Integer(ENV.fetch("PORT", "8787"))) do |http_server|
  trap("INT") { http_server.shutdown }
  trap("TERM") { http_server.shutdown }
end
```


This snippet also responds to `GET /` for health checks, handles CORS preflight for `/mcp`, and returns `404 Not Found` for OAuth discovery routes you are not using yet. That keeps ChatGPT from surfacing 502 errors while you iterate without authentication.

## Run locally

If you're using a web framework like React, build your component into static assets so the HTML template can inline them.
Usually, you can run a build command such as `npm run build` to produce a `dist` directory with your compiled assets.

In this quickstart, since we're using vanilla HTML, no build step is required.




For Java, run the server class’s `main` method from your IDE or build tool with the Maven dependencies on the runtime classpath. Set the working directory to the project directory containing `public/todo-widget.html`. The Java server listens at `http://localhost:8787/mcp`; leave it running while testing with MCP Inspector.




Start the MCP server on `http://localhost:<port>/mcp` from the directory that contains your server file.




For TypeScript, make sure you have `"type": "module"` in your `package.json` file:

```json
{
  "type": "module",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.20.2",
    "@modelcontextprotocol/ext-apps": "^1.0.1",
    "zod": "^3.25.76"
  }
}
```

Run the TypeScript server with the following command:

```bash
node --experimental-strip-types server.ts
```







Run the Ruby version with `ruby server.rb`. The Ruby server accepts a `PORT` environment variable and reports startup through WEBrick.




By default, the servers serve `http://localhost:8787/mcp`.




The TypeScript server prints `Todo MCP server listening on http://localhost:8787/mcp` when ready.




### Test with MCP Inspector

You can use the [MCP Inspector](https://modelcontextprotocol.io/docs/tools/inspector) to test your server locally.

```bash
npx @modelcontextprotocol/inspector@latest
```

This opens the MCP Inspector interface. Select **Streamable HTTP**, enter
`http://localhost:8787/mcp`, and connect to test your server and inspect its
tool responses.

![MCP Inspector](https://developers.openai.com/images/apps-sdk/mcp_inspector.png)

### Expose your server to the public internet

For ChatGPT to access your server during development, you need to expose it to the public internet. You can use a tool such as [ngrok](https://ngrok.com/) to open a tunnel to your local server.

```bash
ngrok http <port>
```

This will give you a public URL like `https://<subdomain>.ngrok.app` that you can use to access your server from ChatGPT.




For Java, add your tunnel's exact hostname (for example, `your-tunnel.ngrok.app`, without `https://` or a path) to `allowedHosts`, then restart the server. Update the entry if your tunnel URL changes. Requests with other hostnames are rejected. Keep `allowedOrigins` restricted to trusted browser origins; ChatGPT's server connection does not require adding the tunnel URL to that list.




When you connect your MCP server in ChatGPT, provide the public URL with
the `/mcp` path (for example, `https://<subdomain>.ngrok.app/mcp`).

## Connect your MCP server in ChatGPT

Once your MCP server and web component work locally, connect the server in
ChatGPT:

1. Go to [ChatGPT Plugins](https://chatgpt.com/plugins), select the plus button, then **Add custom MCP server**.
2. Paste the HTTPS + `/mcp` URL from your tunnel or deployment (for example, `https://<subdomain>.ngrok.app/mcp`), name the connection, and configure authentication.
3. Review the risk warning, select **I understand and want to continue**, then **Create as a plugin**.
4. Install the resulting plugin, open a new chat, type `@` in the prompt box, and select the plugin. Prompt the model (for example, “Add a new task to read my book”). ChatGPT will stream tool payloads so you can confirm inputs and outputs.

## Next steps

From there, you can iterate on the UI/UX, prompts, tool metadata, and the overall experience.

Refresh the plugin connection after each change to the MCP server (tools,
  metadata, and related configuration). You can do this from the detail page at
  [ChatGPT plugins](https://chatgpt.com/plugins).

When you're preparing for public distribution, review
[Submit plugins](https://developers.openai.com/plugins/deploy/submission), the
[Plugin guidelines](https://developers.openai.com/plugins/plugin-guidelines), and
[Brainstorm plugin use cases](https://developers.openai.com/plugins/plan/use-case). If you're building a UI, you
can also review the [UI guidelines](https://developers.openai.com/plugins/concepts/ui-guidelines).

Once you understand the basics, you can
[build richer UI](https://developers.openai.com/plugins/build/chatgpt-ui), [authenticate
users](https://developers.openai.com/plugins/build/auth) when needed, and
[manage state](https://developers.openai.com/plugins/build/chatgpt-ui#manage-state).