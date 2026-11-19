# Reference

> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

**For new MCP apps, follow the [MCP Apps UI](https://modelcontextprotocol.io/docs/extensions/apps) and [OpenAI MCP extensions](https://github.com/openai/mcp-extensions/blob/main/docs/spec.md) specifications.** ChatGPT supports and endorses both specifications as the official way to add UI to your MCP server.

This page documents the legacy `window.openai` API, which predates these specifications. Use these APIs only for capabilities these specifications don't cover.

## `window.openai` component bridge

ChatGPT provides `window.openai` for compatibility aliases and optional
ChatGPT extensions. New UI should use the MCP Apps bridge whenever the shared
specification provides an equivalent, then use `window.openai` only for
ChatGPT-specific capabilities.

See [build a ChatGPT UI](https://developers.openai.com/plugins/build/chatgpt-ui) for implementation walkthroughs.

If your tool requires confirmation, treat missing initial `toolInput` as
expected. ChatGPT does not load approval-gated arguments into widget values
before approval; instead, the host delivers them through
`ui/notifications/tool-input` once the user approves the call.

### Capabilities

| Capability          | What it does                                                                                                                                                                     | Typical use                                                                                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| State & data        | `window.openai.toolInput`                                                                                                                                                        | Arguments supplied when the tool was invoked. For approval-gated tools, this may remain `null` until the host sends `ui/notifications/tool-input` after approval.                                   |
| State & data        | `window.openai.toolOutput`                                                                                                                                                       | Your `structuredContent`. Keep fields concise; the model reads them verbatim.                                                                                                                       |
| State & data        | `window.openai.toolResponseMetadata`                                                                                                                                             | Canonical widget-only tool result metadata. In ChatGPT this includes `status`, `call_tool_result`, and `mcp_tool_result`, preserving the full MCP result envelope, including hidden `_meta`.        |
| State & data        | `window.openai.widgetState`                                                                                                                                                      | Snapshot of UI state persisted between renders.                                                                                                                                                     |
| State & data        | `window.openai.setWidgetState(state)`                                                                                                                                            | Stores a new snapshot synchronously; call it after every meaningful UI interaction.                                                                                                                 |
| Widget runtime APIs | `window.openai.callTool(name, args)`                                                                                                                                             | Invoke another MCP tool from the widget (mirrors model-initiated calls).                                                                                                                            |
| Widget runtime APIs | `window.openai.sendFollowUpMessage({ prompt, scrollToBottom })`                                                                                                                  | Ask ChatGPT to post a message authored by the component. `scrollToBottom` is optional, defaults to `true`, and can be set to `false` to prevent automatic scrolling.                                |
| Widget runtime APIs | `window.openai.uploadFile(file, { library?: boolean })`                                                                                                                          | Upload a user-selected file and receive a `fileId`. Pass `{ library: true }` to also save the upload in the user's ChatGPT file library when that library is available.                             |
| Widget runtime APIs | `window.openai.selectFiles()`                                                                                                                                                    | Open ChatGPT's file library picker and return plugin-authorized files as `{ fileId, fileName, mimeType }[]`. Feature-detect this helper because the file library may not be available to all users. |
| Widget runtime APIs | `window.openai.getFileDownloadUrl({ fileId })`                                                                                                                                   | Retrieve a temporary download URL for a file uploaded by the widget, selected from the file library, passed via file params, or returned by tool file references.                                   |
| Widget runtime APIs | `window.openai.requestDisplayMode(...)`                                                                                                                                          | Request PiP/fullscreen modes.                                                                                                                                                                       |
| Widget runtime APIs | `window.openai.requestModal({ params, template })`                                                                                                                               | Spawn a modal owned by ChatGPT. Omit `template` to use the current template, or pass a registered template URI to switch modal content.                                                             |
| Widget runtime APIs | `window.openai.requestClose()`                                                                                                                                                   | Ask ChatGPT to close the current widget.                                                                                                                                                            |
| Widget runtime APIs | `window.openai.notifyIntrinsicHeight(...)`                                                                                                                                       | Report dynamic widget heights to avoid scroll clipping.                                                                                                                                             |
| Widget runtime APIs | `window.openai.openExternal({ href, redirectUrl })`                                                                                                                              | Open a vetted external link in the user's browser. For approved redirect targets, ChatGPT appends `?redirectUrl=...` by default; set `redirectUrl: false` to skip it.                               |
| Widget runtime APIs | `window.openai.setOpenInAppUrl({ href })`                                                                                                                                        | Optionally override the external target shown in fullscreen. If unset, ChatGPT keeps the default behavior and opens the component's current iframe path.                                            |
| Context             | `window.openai.theme`, `window.openai.displayMode`, `window.openai.maxHeight`, `window.openai.safeArea`, `window.openai.view`, `window.openai.userAgent`, `window.openai.locale` | Environment signals you can read or subscribe to through `useOpenAiGlobal` to adapt visuals and copy.                                                                                               |

### `useOpenAiGlobal` helper

Many ChatGPT UI projects wrap `window.openai` access in small helper functions
so views remain testable. This example helper listens for host
`openai:set_globals` events and lets React components subscribe to a single
global value:

```typescript
import { useSyncExternalStore } from "react";

type WebplusGlobals = {
  widgetState: { selectedId?: string | null; [key: string]: unknown } | null;
  toolInput: Record<string, unknown>;
  toolOutput: Record<string, unknown> | null;
  theme: "light" | "dark";
};
const SET_GLOBALS_EVENT_TYPE = "openai:set_globals";

export function useOpenAiGlobal<K extends keyof WebplusGlobals>(
  key: K
): WebplusGlobals[K] {
  return useSyncExternalStore(
    (onChange) => {
      const handleSetGlobal = (event: Event) => {
        if (!(event instanceof CustomEvent)) return;
        const value = event.detail?.globals?.[key];
        if (value === undefined) {
          return;
        }

        onChange();
      };

      window.addEventListener(SET_GLOBALS_EVENT_TYPE, handleSetGlobal, {
        passive: true,
      });

      return () => {
        window.removeEventListener(SET_GLOBALS_EVENT_TYPE, handleSetGlobal);
      };
    },
    () => {
      const globals = (window as Window & { openai?: WebplusGlobals }).openai;
      if (!globals) throw new Error("This hook requires a ChatGPT app host.");
      return globals[key];
    }
  );
}
```


### Close the UI

Call `window.openai.requestClose()` to ask ChatGPT to close the current UI.

### Request another presentation mode

Use `window.openai.requestDisplayMode` to request inline, picture-in-picture,
or fullscreen presentation:

```tsx
await window.openai?.requestDisplayMode({ mode: "fullscreen" });
// On mobile, picture-in-picture may be presented as fullscreen.
```


### Open a modal

Use `window.openai.requestModal` to open a host-controlled modal. Provide the
URI of another UI template registered by the same MCP server, or omit
`template` to open the current template:

```tsx
await window.openai.requestModal({
  template: "ui://widget/details.html",
});
```


## File APIs

ChatGPT supports file upload/download helpers as optional `window.openai`
extensions.

| API                                                     | Purpose                                             | Notes                                                                                                                                   |
| ------------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `window.openai.uploadFile(file, { library?: boolean })` | Upload a user-selected file and receive a `fileId`. | Pass `{ library: true }` to also save the upload in the user's ChatGPT file library when that library is available to the current user. |
| `window.openai.selectFiles()`                           | Open the file library picker for existing files.    | Returns `[{ fileId, fileName, mimeType }]`. Feature-detect this helper because the file library may not be available to all users.      |
| `window.openai.getFileDownloadUrl({ fileId })`          | Request a temporary download URL for a file.        | Works for files uploaded by the widget, selected from the file library, passed via file params, or returned by tool file references.    |

The ChatGPT file library is optional and may not be available to every user.
Files returned from `window.openai.selectFiles()` are already authorized for
the current plugin when the helper is available. Use the returned `fileId` with
`window.openai.getFileDownloadUrl({ fileId })` or in a tool input that uses
file params.

Upload a user-selected file:

```tsx
const { fileId } = await window.openai.uploadFile(file, {
  library: true,
});
```


Select files that the user already uploaded to ChatGPT:

```tsx
if (window.openai?.selectFiles) {
  const files = await window.openai.selectFiles();
  // [{ fileId, fileName, mimeType }]
}
```


Feature-detect `window.openai.selectFiles` and fall back to
`window.openai.uploadFile` when the file library is unavailable.

Request a temporary download URL:

```tsx
const { downloadUrl } = await window.openai.getFileDownloadUrl({ fileId });
```


### Define file inputs

To let ChatGPT pass files to a tool, list each top-level file input in
`_meta["openai/fileParams"]`. Each listed field must resolve to a file object or
an array of file objects.

Every file object schema must declare all four supported properties:

| Property       | Type     | Declare in `properties` | Include in `required` |
| -------------- | -------- | :---------------------: | :-------------------: |
| `download_url` | `string` |           Yes           |          Yes          |
| `file_id`      | `string` |           Yes           |          Yes          |
| `mime_type`    | `string` |           Yes           |          No           |
| `file_name`    | `string` |           Yes           |          No           |

`mime_type` and `file_name` are optional values, but you must declare their
properties in the schema. The **Scan Tools** step and plugin submission reject a
file schema that omits any of the four properties, does not require
`download_url` and `file_id`, marks either optional property as required, or
requires a property other than `download_url` or `file_id`. You can declare
extra optional properties.

This complete tool descriptor accepts one required file input:

```json
{
  "name": "analyze_file",
  "title": "Analyze file",
  "description": "Analyzes a user-provided file without modifying it.",
  "inputSchema": {
    "type": "object",
    "$defs": {
      "OpenAIFile": {
        "type": "object",
        "properties": {
          "download_url": { "type": "string" },
          "file_id": { "type": "string" },
          "mime_type": { "type": "string" },
          "file_name": { "type": "string" }
        },
        "required": ["download_url", "file_id"],
        "additionalProperties": false
      }
    },
    "properties": {
      "file": { "$ref": "#/$defs/OpenAIFile" }
    },
    "required": ["file"]
  },
  "annotations": {
    "readOnlyHint": true,
    "openWorldHint": false,
    "destructiveHint": false
  },
  "_meta": {
    "openai/fileParams": ["file"]
  }
}
```

To accept more than one file, define the top-level field as an array and use the
same file object schema in `items`. The tool can require the top-level file
field independently of the properties required inside each file object.

At runtime, ChatGPT passes file values with snake case fields:

```json
{
  "download_url": "https://...",
  "file_id": "file_...",
  "mime_type": "image/png",
  "file_name": "input.png"
}
```

ChatGPT always includes `download_url` and `file_id`; it may omit `mime_type`
and `file_name`. Use `file_id` as the `fileId` value for
`window.openai.getFileDownloadUrl({ fileId })` when a widget needs a fresh
temporary download URL.

When persisting widget state, use the structured shape (`modelContent`, `privateContent`, `imageIds`) if you want the model to see image IDs during follow-up turns.

## Host-backed navigation

The sandbox runtime mirrors navigation history from the iframe into ChatGPT's
UI. Use standard routing APIs, such as React Router, and the host keeps its
navigation controls in sync with your UI.

Router setup with React Router's `BrowserRouter`:

```tsx
import {
  BrowserRouter,
  Routes,
  Route,
  Link,
  useParams,
} from "react-router-dom";
function PizzaListPlugin() {
  const { placeId } = useParams();
  return (
    <main>
      

Example Pizzeria


      <p>{placeId ?? "Choose a place"}</p>
    </main>
  );
}

export default function PizzaListRouter() {
  return (
    

<Routes>
        }>
          } />
        </Route>
      </Routes>


  );
}
```


Programmatic navigation:

```typescript
import { createElement } from "react";
import { BrowserRouter, useNavigate } from "react-router-dom";
function NavigationExample() {
  const navigate = useNavigate();

  function openDetails(placeId: string) {
    navigate(`place/${placeId}`, { replace: false });
  }

  function closeDetails() {
    navigate("..", { replace: true });
  }

  return createElement(
    "div",
    null,
    createElement(
      "button",
      { onClick: () => openDetails("pizzeria") },
      "Open details"
    ),
    createElement("button", { onClick: closeDetails }, "Close details")
  );
}
```


## Tool descriptor parameters

By default, a tool description should include the fields listed [here](https://modelcontextprotocol.io/specification/2025-06-18/server/tools#tool).

Declare `outputSchema` for any tool that returns `structuredContent`. The
schema should describe the exact object your tool returns so clients can
validate results and the model can reason about follow-up tool calls.

### `_meta` fields on tool descriptor

Use these `_meta` fields on the tool descriptor. Prefer the MCP Apps standard
key `_meta.ui.resourceUri` for linking a tool to a UI template. ChatGPT supports
OpenAI-specific metadata for compatibility and optional extensions.

| Key                                       |    Placement    | Type         | Limits                                          | Purpose                                                                                                                                                                                                                                                                                     |
| ----------------------------------------- | :-------------: | ------------ | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `_meta["securitySchemes"]`                | Tool descriptor | array        | None                                            | Back-compat mirror for clients that only read `_meta`.                                                                                                                                                                                                                                      |
| `_meta.ui.resourceUri`                    | Tool descriptor | string (URI) | None                                            | Standard resource URI for the UI template.                                                                                                                                                                                                                                                  |
| `_meta.ui.visibility`                     | Tool descriptor | string[]     | default `["model", "app"]`                      | Controls whether a tool is available to the model, the UI, or both. The `app` value is the MCP Apps protocol identifier for UI.                                                                                                                                                             |
| `_meta["openai/outputTemplate"]`          | Tool descriptor | string (URI) | None                                            | OpenAI-specific optional/compatibility alias for `_meta.ui.resourceUri` in ChatGPT.                                                                                                                                                                                                         |
| `_meta["openai/profile"]`                 | Tool descriptor | boolean      | Optional; only `true` designates a profile tool | Identifies the authenticated, read-only tool that returns the current profile. Implement it to help users recognize and manage multiple connected accounts. Users can connect multiple accounts without it. See [Support multiple accounts](https://developers.openai.com/plugins/build/auth#support-multiple-accounts). |
| `_meta["openai/widgetAccessible"]`        | Tool descriptor | boolean      | default `false`                                 | OpenAI-specific compatibility field used by existing UI integrations; prefer `_meta.ui.visibility` + `tools/call`.                                                                                                                                                                          |
| `_meta["openai/visibility"]`              | Tool descriptor | string       | `public` (default) or `private`                 | OpenAI-specific compatibility field used by existing UI integrations; prefer `_meta.ui.visibility`.                                                                                                                                                                                         |
| `_meta["openai/toolInvocation/invoking"]` | Tool descriptor | string       | ≤ 64 chars                                      | Short status text while the tool runs.                                                                                                                                                                                                                                                      |
| `_meta["openai/toolInvocation/invoked"]`  | Tool descriptor | string       | ≤ 64 chars                                      | Short status text after the tool completes.                                                                                                                                                                                                                                                 |
| `_meta["openai/fileParams"]`              | Tool descriptor | string[]     | None                                            | List of top-level input fields that represent files. Each field receives `{ download_url, file_id, mime_type?, file_name? }`.                                                                                                                                                               |




The Go resource examples use `github.com/modelcontextprotocol/go-sdk/mcp` at version `v1.8.0`. Save your bundled MCP Apps browser component as `widget.js` in the project directory, then run the Go server from that directory.







The Ruby resource examples require the `mcp` gem at version `1.10.0` and your bundled MCP Apps browser component saved as `widget.js` in the project directory. Run the Ruby server from that directory.




Example:

Add tool descriptor metadata

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
const server = new McpServer({ name: "docs-example", version: "1.0.0" });

// Local example data; replace this with your application data.
async function performSearch(query: string) {
  const documents = [
    {
      id: "intro",
      title: "Introduction to Acme",
      url: "https://example.com/intro",
    },
  ];
  return documents.filter((document) =>
    document.title.toLowerCase().includes(query.toLowerCase())
  );
}

import { registerAppTool } from "@modelcontextprotocol/ext-apps/server";
import { z } from "zod";

registerAppTool(
  server,
  "search",
  {
    title: "Public Search",
    description: "Search public documents.",
    inputSchema: { q: z.string() },
    outputSchema: {
      results: z.array(
        z.object({
          id: z.string(),
          title: z.string(),
          url: z.string(),
        })
      ),
    },
    _meta: {
      securitySchemes: [
        { type: "noauth" },
        { type: "oauth2", scopes: ["search.read"] },
      ],
      ui: { resourceUri: "ui://widget/story.html" },
      // Optional compatibility alias (ChatGPT only):
      // "openai/outputTemplate": "ui://widget/story.html",
      "openai/toolInvocation/invoking": "Searching…",
      "openai/toolInvocation/invoked": "Results ready",
    },
  },
  async ({ q }) => {
    const results = await performSearch(q);

    return {
      structuredContent: { results },
      content: [{ type: "text", text: `Found ${results.length} results.` }],
    };
  }
);
```

```go
package main

import (
	"context"
	"fmt"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
	"os"
	"regexp"
	"strings"
)

type searchInput struct {
	Q string `json:"q"`
}
type document struct {
	ID    string `json:"id"`
	Title string `json:"title"`
	URL   string `json:"url"`
}
type searchOutput struct {
	Results []document `json:"results"`
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "search", Title: "Public Search", Description: "Search public documents.", Meta: mcp.Meta{
		"securitySchemes": []map[string]any{{"type": "noauth"}, {"type": "oauth2", "scopes": []string{"search.read"}}},
		"ui":              map[string]any{"resourceUri": "ui://widget/story.html"}, "openai/toolInvocation/invoking": "Searching…", "openai/toolInvocation/invoked": "Results ready",
	}}, func(_ context.Context, _ *mcp.CallToolRequest, input searchInput) (*mcp.CallToolResult, searchOutput, error) {
		// Local example data; replace this with your application data.
		documents := []document{{ID: "intro", Title: "Introduction to Acme", URL: "https://example.com/intro"}}
		found := []document{}
		for _, item := range documents {
			if strings.Contains(strings.ToLower(item.Title), strings.ToLower(input.Q)) {
				found = append(found, item)
			}
		}
		return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: fmt.Sprintf("Found %d results.", len(found))}}}, searchOutput{Results: found}, nil
	})
	// Build the browser component first, then run from the project root.
	script, err := os.ReadFile("widget.js")
	if err != nil {
		log.Fatal(err)
	}
	escaped := regexp.MustCompile(`(?i)</script`).ReplaceAllLiteralString(string(script), `<\/script`)
	html := `<!doctype html><html lang="en"><meta charset="utf-8"><title>Tool result</title><body><div id="root"></div><script type="module">` + escaped + `</script></body></html>`
	server.AddResource(&mcp.Resource{URI: "ui://widget/story.html", Name: "widget", MIMEType: "text/html;profile=mcp-app"}, func(context.Context, *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: "ui://widget/story.html", MIMEType: "text/html;profile=mcp-app", Text: html}}}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```ruby
require "mcp"

class PublicSearch < MCP::Tool
  tool_name "search"
  title "Public Search"
  description "Search public documents."
  input_schema(properties: { q: { type: "string" } }, required: ["q"])
  output_schema(
    type: "object", properties: {
      results: {
        type: "array",

        items: {
          type: "object",

          properties: {
            id: { type: "string" },
            title: { type: "string" },
            url: { type: "string" }
          },
          required: ["id", "title", "url"]
        }
      }
    }, required: ["results"]
  )
  meta(
    securitySchemes: [
      { type: "noauth" }, {
        type: "oauth2",
        scopes: ["search.read"]
      }
    ],
    ui: { resourceUri: "ui://widget/story.html" },
    "openai/toolInvocation/invoking": "Searching…",
    "openai/toolInvocation/invoked": "Results ready"
  )

  def self.call(q:, server_context:)
    # Local example data; replace this with your application data.
    documents = [
      {
        id: "intro",
        title: "Introduction to Acme",
        url: "https://example.com/intro"
      }
    ]
    results = documents.select { |document| document.fetch(:title).downcase.include?(q.downcase) }
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: "Found #{results.length} results."
        }
      ],
      structured_content: { results: results }
    )
  end
end

# Build the browser component first, then run this server from the project root.
script = File.read("widget.js").gsub(%r{</script}i) { '<\/script' }
html = '<!doctype html><html lang="en"><meta charset="utf-8">' \
       '<title>Tool result</title><body><div id="root"></div>' \
       "<script type=\"module\">#{script}</script></body></html>"
resource = MCP::Resource.define(uri: "ui://widget/story.html", name: "widget", mime_type: "text/html;profile=mcp-app") do
  [
    MCP::Resource::TextContents.new(
      uri: "ui://widget/story.html", mime_type: "text/html;profile=mcp-app", text: html
    )
  ]
end

server = MCP::Server.new(
  name: "docs-example", version: "1.0.0", tools: [PublicSearch],
  resources: [resource]
)
MCP::Server::Transports::StdioTransport.new(server).open
```


### Annotations

To label a tool as "read-only," use the following
[`ToolAnnotations`
fields](https://modelcontextprotocol.io/specification/2025-11-25/schema#toolannotations)
on the tool descriptor:

| Key               | Type    | Required | Notes                                                                                                                                                                                                                                       |
| ----------------- | ------- | :------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `readOnlyHint`    | boolean | Required | Signal that the tool only retrieves or computes information and doesn't create, update, delete, or send data outside the conversation.                                                                                                      |
| `destructiveHint` | boolean | Required | Declare that the tool may delete or overwrite user data so the host knows to elicit explicit approval first.                                                                                                                                |
| `openWorldHint`   | boolean | Required | Declare that the tool accesses the public internet or open-ended external entities, including through read-only actions such as web search. A bounded private account or workspace isn't open-world solely because it is externally hosted. |
| `idempotentHint`  | boolean | Optional | Declare that calling the tool with the same arguments has no extra effect on its environment.                                                                                                                                               |

These hints only influence how ChatGPT or Codex frames the tool call to the
user; servers must still enforce their own authorization logic.

Example:

Annotate a read-only tool

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
const server = new McpServer({ name: "docs-example", version: "1.0.0" });

// Local example data; replace this with your application data.
async function fetchSavedRecipes() {
  return [{ id: "pasta", title: "Tomato pasta" }];
}

import { z } from "zod";

server.registerTool(
  "list_saved_recipes",
  {
    title: "List saved recipes",
    description: "Returns the user’s saved recipes without modifying them.",
    inputSchema: {},
    outputSchema: {
      recipes: z.array(
        z.object({
          id: z.string(),
          title: z.string(),
        })
      ),
    },
    annotations: { readOnlyHint: true },
  },
  async () => ({
    content: [],
    structuredContent: { recipes: await fetchSavedRecipes() },
  })
);
```

```go
package main

import (
	"context"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
)

type recipe struct {
	ID    string `json:"id"`
	Title string `json:"title"`
}
type recipeOutput struct {
	Recipes []recipe `json:"recipes"`
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "list_saved_recipes", Description: "Returns the user's saved recipes without modifying them.", Annotations: &mcp.ToolAnnotations{ReadOnlyHint: true}}, func(context.Context, *mcp.CallToolRequest, struct{}) (*mcp.CallToolResult, recipeOutput, error) {
		// Local example data; replace this with your application data.
		return nil, recipeOutput{Recipes: []recipe{{ID: "pasta", Title: "Tomato pasta"}}}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```ruby
require "mcp"

class ListSavedRecipes < MCP::Tool
  tool_name "list_saved_recipes"
  description "Returns the user's saved recipes without modifying them."
  input_schema(properties: {})
  output_schema(
    type: "object",
    properties: {
      recipes: {
        type: "array",
        items: {
          type: "object",
          properties: {
            id: { type: "string" },
            title: { type: "string" }
          },
          required: ["id", "title"]
        }
      }
    },
    required: ["recipes"]
  )
  annotations(read_only_hint: true, destructive_hint: nil, idempotent_hint: nil, open_world_hint: nil)

  def self.call(server_context:)
    # Local example data; replace this with your application data.
    MCP::Tool::Response.new(
      [], structured_content: {
        recipes: [
          {
            id: "pasta",
            title: "Tomato pasta"
          }
        ]
      }
    )
  end
end

server = MCP::Server.new(name: "docs-example", version: "1.0.0", tools: [ListSavedRecipes])
MCP::Server::Transports::StdioTransport.new(server).open
```





## Component resource `_meta` fields

Set these keys on the resource template that serves your component (`registerResource`). They help ChatGPT describe and frame the rendered iframe without leaking metadata to other clients.

| Key                                        |     Placement     | Type            | Purpose                                                                                                                                                                                           |
| ------------------------------------------ | :---------------: | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `_meta.ui.prefersBorder`                   | Resource contents | boolean         | Hint that the component should render inside a bordered card when supported.                                                                                                                      |
| `_meta.ui.csp`                             | Resource contents | object          | Preferred metadata surface for standard widget CSP fields: `connectDomains`, `resourceDomains`, and optional `frameDomains`.                                                                      |
| `_meta.ui.domain`                          | Resource contents | string (origin) | Dedicated origin for hosted components (required when submitting a plugin with UI; must be unique per plugin). Defaults to `https://web-sandbox.oaiusercontent.com`.                              |
| `_meta["openai/ui"].availableDisplayModes` | Resource contents | string[]        | Supported display modes: `inline`, `fullscreen`, and `pip`. Lets ChatGPT choose a display mode before loading the component.                                                                      |
| `_meta["openai/widgetDescription"]`        | Resource contents | string          | Human-readable summary surfaced to the model when the component loads, reducing redundant assistant narration.                                                                                    |
| `_meta["openai/widgetPrefersBorder"]`      | Resource contents | boolean         | OpenAI-specific compatibility alias for `_meta.ui.prefersBorder` in ChatGPT.                                                                                                                      |
| `_meta["openai/widgetCSP"]`                | Resource contents | object          | Legacy ChatGPT compatibility key for widget CSP metadata. Standard CSP fields are superseded by `_meta.ui.csp`, but `redirect_domains` is still required for trusted `openExternal` destinations. |
| `_meta["openai/widgetDomain"]`             | Resource contents | string (origin) | OpenAI-specific compatibility alias for `_meta.ui.domain` in ChatGPT.                                                                                                                             |

ChatGPT supports the legacy `_meta["openai/widgetCSP"]` compatibility key with the following snake_case field names:

- `connect_domains`: `string[]`
- `resource_domains`: `string[]`
- `frame_domains?`: `string[]`
- `redirect_domains?`: `string[]`. ChatGPT extension for `window.openai.openExternal` redirect targets.

The standard `_meta.ui.csp` object is generally preferred for new UI and supports:

- `connectDomains`: `string[]`. Domains the widget may contact via fetch/XHR.
- `resourceDomains`: `string[]`. Domains for static assets (images, fonts, scripts, styles).
- `frameDomains?`: `string[]`. Optional list of origins allowed for iframe embeds. By default, widgets can't render subframes. Plugins can embed their own domain, including existing editors and admin interfaces, under the [iframe policy](https://developers.openai.com/plugins/plugin-guidelines#iframes-and-embedded-pages). A justification is required at submission, and iframe use can require additional review or lead to slower approval.

However, `_meta.ui.csp` does not support `redirect_domains` for `window.openai.openExternal(...)` links. To allowlist redirect targets, you must still set `_meta["openai/widgetCSP"].redirect_domains`.

### Declare display modes before the UI loads

Set `_meta["openai/ui"].availableDisplayModes` on the resource contents returned
by your server. For a fullscreen-only component, this lets ChatGPT open it in
fullscreen immediately, without showing an inline loading state first.

Return this object in the resource's `contents` array, with `html` set to your
component's HTML:

```ts
const resource = {
  uri: "ui://widget/viewer.html",
  mimeType: "text/html;profile=mcp-app",
  text: html,
  _meta: {
    "openai/ui": {
      availableDisplayModes: ["fullscreen"],
    },
  },
};
```

List every mode your component supports, such as `["inline", "fullscreen"]`.
Keep declaring `availableDisplayModes` during MCP Apps initialization too;
the server declaration supplements that flow, which also supports dynamic
display modes.

## Tool results

Tool results can contain the following [fields](https://modelcontextprotocol.io/specification/2025-06-18/server/tools#tool-result). Notably:

| Key                 | Type                  | Required | Notes                                                                                           |
| ------------------- | --------------------- | -------- | ----------------------------------------------------------------------------------------------- |
| `structuredContent` | object                | Optional | Surfaced to the model and the component. Must match the declared `outputSchema`, when provided. |
| `content`           | string or `Content[]` | Optional | Surfaced to the model and the component.                                                        |
| `_meta`             | object                | Optional | Delivered only to the component. Hidden from the model.                                         |

Only `structuredContent` and `content` appear in the conversation transcript. The host forwards `_meta` to the component so you can hydrate UI without exposing the data to the model.

Host-provided tool result metadata:

| Key                               |            Placement            | Type   | Purpose                                                                                                                 |
| --------------------------------- | :-----------------------------: | ------ | ----------------------------------------------------------------------------------------------------------------------- |
| `_meta["openai/widgetSessionId"]` | Tool result `_meta` (from host) | string | Stable ID for the currently mounted widget instance; use it to correlate logs and tool calls until the widget unmounts. |

Example:

Return structured tool results

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
const server = new McpServer({ name: "docs-example", version: "1.0.0" });

// Local example data; replace this with your application data.
function generateZooAnimals(count: number) {
  return Array.from({ length: count }, (_, index) => ({
    id: `animal-${index + 1}`,
    name: `Penguin ${index + 1}`,
    species: "Penguin",
  }));
}

import { registerAppTool } from "@modelcontextprotocol/ext-apps/server";
import { z } from "zod";

registerAppTool(
  server,
  "get_zoo_animals",
  {
    title: "get_zoo_animals",
    inputSchema: { count: z.number().int().min(1).max(20).optional() },
    outputSchema: {
      animals: z.array(
        z.object({
          id: z.string(),
          name: z.string(),
          species: z.string(),
        })
      ),
    },
    _meta: { ui: { resourceUri: "ui://widget/widget.html" } },
  },
  async ({ count = 10 }) => {
    const animals = generateZooAnimals(count);

    return {
      structuredContent: { animals },
      content: [{ type: "text", text: `Here are ${animals.length} animals.` }],
      _meta: {
        allAnimalsById: Object.fromEntries(
          animals.map((animal) => [animal.id, animal])
        ),
      },
    };
  }
);
```

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"regexp"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

type zooInput struct {
	Count *int `json:"count,omitempty"`
}
type animal struct {
	ID      string `json:"id"`
	Name    string `json:"name"`
	Species string `json:"species"`
}
type zooOutput struct {
	Animals []animal `json:"animals"`
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "get_zoo_animals", Meta: mcp.Meta{"ui": map[string]any{"resourceUri": "ui://widget/widget.html"}}, InputSchema: map[string]any{"type": "object", "properties": map[string]any{"count": map[string]any{"type": "integer", "minimum": 1, "maximum": 20}}}}, func(_ context.Context, _ *mcp.CallToolRequest, input zooInput) (*mcp.CallToolResult, zooOutput, error) {
		count := 10
		if input.Count != nil {
			count = *input.Count
		}
		// Local example data; replace this with your application data.
		animals := []animal{}
		byID := map[string]animal{}
		for i := 1; i <= count; i++ {
			value := animal{ID: fmt.Sprintf("animal-%d", i), Name: fmt.Sprintf("Penguin %d", i), Species: "Penguin"}
			animals = append(animals, value)
			byID[value.ID] = value
		}
		return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: fmt.Sprintf("Here are %d animals.", count)}}, Meta: mcp.Meta{"allAnimalsById": byID}}, zooOutput{Animals: animals}, nil
	})
	// Build the browser component first, then run from the project root.
	script, err := os.ReadFile("widget.js")
	if err != nil {
		log.Fatal(err)
	}
	escaped := regexp.MustCompile(`(?i)</script`).ReplaceAllLiteralString(string(script), `<\/script`)
	html := `<!doctype html><html lang="en"><meta charset="utf-8"><title>Tool result</title><body><div id="root"></div><script type="module">` + escaped + `</script></body></html>`
	server.AddResource(&mcp.Resource{URI: "ui://widget/widget.html", Name: "widget", MIMEType: "text/html;profile=mcp-app"}, func(context.Context, *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{URI: "ui://widget/widget.html", MIMEType: "text/html;profile=mcp-app", Text: html}}}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```ruby
require "mcp"

class ZooAnimals < MCP::Tool
  tool_name "get_zoo_animals"
  input_schema(
    properties: {
      count: {
        type: "integer",
        minimum: 1,
        maximum: 20
      }
    }
  )
  output_schema(
    type: "object", properties: {
      animals: {
        type: "array",

        items: {
          type: "object",

          properties: {
            id: { type: "string" },
            name: { type: "string" },
            species: { type: "string" }
          },
          required: ["id", "name", "species"]
        }
      }
    }, required: ["animals"]
  )
  meta(ui: { resourceUri: "ui://widget/widget.html" })

  def self.call(server_context:, count: 10)
    # Local example data; replace this with your application data.
    animals = Array.new(count) do |index|
      {
        id: "animal-#{index + 1}",
        name: "Penguin #{index + 1}",
        species: "Penguin"
      }
    end
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: "Here are #{animals.length} animals."
        }
      ],
      structured_content: { animals: animals },
      meta: { allAnimalsById: animals.to_h { |animal| [animal.fetch(:id), animal] } }
    )
  end
end

# Build the browser component first, then run this server from the project root.
script = File.read("widget.js").gsub(%r{</script}i) { '<\/script' }
html = '<!doctype html><html lang="en"><meta charset="utf-8">' \
       '<title>Tool result</title><body><div id="root"></div>' \
       "<script type=\"module\">#{script}</script></body></html>"
resource = MCP::Resource.define(uri: "ui://widget/widget.html", name: "widget", mime_type: "text/html;profile=mcp-app") do
  [
    MCP::Resource::TextContents.new(
      uri: "ui://widget/widget.html", mime_type: "text/html;profile=mcp-app", text: html
    )
  ]
end

server = MCP::Server.new(
  name: "docs-example", version: "1.0.0", tools: [ZooAnimals],
  resources: [resource]
)
MCP::Server::Transports::StdioTransport.new(server).open
```


### Error tool result

To return an error on the tool result, use the following `_meta` key:

| Key                             | Purpose      | Type               | Notes                                                    |
| ------------------------------- | ------------ | ------------------ | -------------------------------------------------------- |
| `_meta["mcp/www_authenticate"]` | Error result | string or string[] | RFC 7235 `WWW-Authenticate` challenges to trigger OAuth. |

## `_meta` fields the client provides

| Key                            | When provided           | Type            | Purpose                                                                                      |
| ------------------------------ | ----------------------- | --------------- | -------------------------------------------------------------------------------------------- |
| `_meta["openai/locale"]`       | Initialize + tool calls | string (BCP 47) | Requested locale (older clients may send `_meta["webplus/i18n"]`).                           |
| `_meta["openai/userAgent"]`    | Tool calls              | string          | Optional, best-effort user agent hint for analytics or formatting.                           |
| `_meta["openai/userLocation"]` | Tool calls              | object          | Coarse location hint (`city`, `region`, `country`, `timezone`, `longitude`, `latitude`).     |
| `_meta["openai/subject"]`      | Tool calls              | string          | Anonymized user id sent to MCP servers for the purposes of rate limiting and identification  |
| `_meta["openai/session"]`      | Tool calls              | string          | Anonymized conversation id for correlating tool calls within the same ChatGPT session.       |
| `_meta["openai/organization"]` | Tool calls              | string          | Anonymized organization id associated with the current ChatGPT organization, when available. |

Operation-phase `_meta["openai/userAgent"]` and `_meta["openai/userLocation"]` are hints only; servers should never rely on them for authorization decisions and must tolerate their absence. Treat `_meta["openai/userAgent"]` as optional, best-effort metadata rather than a stable way to detect which host surface is calling your server.

Example:

Read client metadata

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
const server = new McpServer({ name: "docs-example", version: "1.0.0" });

// Local example data; replace this with your application data.
async function findNearbyCafes(city: unknown) {
  return city === "San Francisco"
    ? [{ name: "Example Cafe", address: "123 Example Street, San Francisco" }]
    : [];
}

function formatIntro(locale: unknown, city: unknown) {
  const location = typeof city === "string" ? city : "your area";
  return locale === "es" ? `Cafés en ${location}` : `Cafes in ${location}`;
}

import { z } from "zod";

server.registerTool(
  "recommend_cafe",
  {
    title: "Recommend a cafe",
    inputSchema: {},
    outputSchema: {
      cafes: z.array(
        z.object({
          name: z.string(),
          address: z.string(),
        })
      ),
    },
  },
  async (_args, { _meta }) => {
    const locale = _meta?.["openai/locale"] ?? "en";
    const userLocation = _meta?.["openai/userLocation"];
    const location =
      typeof userLocation === "object" &&
      userLocation !== null &&
      "city" in userLocation
        ? userLocation.city
        : undefined;
    const cafes = await findNearbyCafes(location);

    return {
      content: [{ type: "text", text: formatIntro(locale, location) }],
      structuredContent: { cafes },
    };
  }
);
```

```go
package main

import (
	"context"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"log"
)

type cafe struct {
	Name    string `json:"name"`
	Address string `json:"address"`
}
type cafeOutput struct {
	Cafes []cafe `json:"cafes"`
}

func main() {
	server := mcp.NewServer(&mcp.Implementation{Name: "docs-example", Version: "1.0.0"}, nil)
	mcp.AddTool(server, &mcp.Tool{Name: "recommend_cafe"}, func(_ context.Context, request *mcp.CallToolRequest, _ struct{}) (*mcp.CallToolResult, cafeOutput, error) {
		meta := request.Params.Meta
		locale, _ := meta["openai/locale"].(string)
		city := "your area"
		if location, ok := meta["openai/userLocation"].(map[string]any); ok {
			if value, ok := location["city"].(string); ok {
				city = value
			}
		}
		// Local example data; replace this with your application data.
		cafes := []cafe{}
		if city == "San Francisco" {
			cafes = append(cafes, cafe{Name: "Example Cafe", Address: "123 Example Street, San Francisco"})
		}
		text := "Cafes in " + city
		if locale == "es" {
			text = "Cafés en " + city
		}
		return &mcp.CallToolResult{Content: []mcp.Content{&mcp.TextContent{Text: text}}}, cafeOutput{Cafes: cafes}, nil
	})
	if err := server.Run(context.Background(), &mcp.StdioTransport{}); err != nil {
		log.Fatal(err)
	}
}
```

```ruby
require "mcp"

class RecommendCafe < MCP::Tool
  tool_name "recommend_cafe"
  input_schema(properties: {})
  output_schema(
    type: "object",
    properties: {
      cafes: {
        type: "array",
        items: {
          type: "object",
          properties: {
            name: { type: "string" },
            address: { type: "string" }
          },
          required: ["name", "address"]
        }
      }
    },
    required: ["cafes"]
  )

  def self.call(server_context:)
    metadata = server_context[:_meta] || {}
    locale = metadata[:"openai/locale"] || "en"
    location = metadata[:"openai/userLocation"]
    city = location[:city] if location.is_a?(Hash)
    city = "your area" unless city.is_a?(String)
    # Local example data; replace this with your application data.
    cafes = (city == "San Francisco") ? [
      {
        name: "Example Cafe",
        address: "123 Example Street, San Francisco"
      }
    ] : []
    text = (locale == "es") ? "Cafés en #{city}" : "Cafes in #{city}"
    MCP::Tool::Response.new(
      [
        {
          type: "text",
          text: text
        }
      ], structured_content: { cafes: cafes }
    )
  end
end

server = MCP::Server.new(name: "docs-example", version: "1.0.0", tools: [RecommendCafe], server_context: {})
MCP::Server::Transports::StdioTransport.new(server).open
```