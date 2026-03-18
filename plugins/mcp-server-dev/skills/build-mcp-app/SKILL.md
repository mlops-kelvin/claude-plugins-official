---
name: build-mcp-app
description: This skill should be used when the user wants to build an "MCP app", add "interactive UI" or "widgets" to an MCP server, "render components in chat", build "MCP UI resources", make a tool that shows a "form", "picker", "dashboard" or "confirmation dialog" inline in the conversation, or mentions "apps SDK" in the context of MCP. Use AFTER the build-mcp-server skill has settled the deployment model, or when the user already knows they want UI widgets.
version: 0.1.0
---

# Build an MCP App (Interactive UI Widgets)

An MCP app is a standard MCP server that **also serves UI resources** — interactive components rendered inline in the chat surface. Build once, runs in Claude *and* ChatGPT and any other host that implements the apps surface.

The UI layer is **additive**. Under the hood it's still tools, resources, and the same wire protocol. If you haven't built a plain MCP server before, the `build-mcp-server` skill covers the base layer. This skill adds widgets on top.

---

## When a widget beats plain text

Don't add UI for its own sake — most tools are fine returning text or JSON. Add a widget when one of these is true:

| Signal | Widget type |
|---|---|
| Tool needs structured input Claude can't reliably infer | Form |
| User must pick from a list Claude can't rank (files, contacts, records) | Picker / table |
| Destructive or billable action needs explicit confirmation | Confirm dialog |
| Output is spatial or visual (charts, maps, diffs, previews) | Display widget |
| Long-running job the user wants to watch | Progress / live status |

If none apply, skip the widget. Text is faster to build and faster for the user.

---

## Widgets vs Elicitation — route correctly

Before building a widget, check if **elicitation** covers it. Elicitation is spec-native, zero UI code, works in any compliant host.

| Need | Elicitation | Widget |
|---|---|---|
| Confirm yes/no | ✅ | overkill |
| Pick from short enum | ✅ | overkill |
| Fill a flat form (name, email, date) | ✅ | overkill |
| Pick from a large/searchable list | ❌ (no scroll/search) | ✅ |
| Visual preview before choosing | ❌ | ✅ |
| Chart / map / diff view | ❌ | ✅ |
| Live-updating progress | ❌ | ✅ |

If elicitation covers it, use it. See `../build-mcp-server/references/elicitation.md`.

---

## Architecture: two deployment shapes

### Remote MCP app (most common)

Hosted streamable-HTTP server. Widget templates are served as **resources**; tool results reference them. The host fetches the resource, renders it in an iframe sandbox, and brokers messages between the widget and Claude.

```
┌──────────┐  tools/call   ┌────────────┐
│  Claude  │─────────────> │ MCP server │
│   host   │<── result ────│  (remote)  │
│          │  + widget ref │            │
│          │               │            │
│          │ resources/read│            │
│          │─────────────> │  widget    │
│ ┌──────┐ │<── template ──│  HTML/JS   │
│ │iframe│ │               └────────────┘
│ │widget│ │
│ └──────┘ │
└──────────┘
```

### MCPB-packaged MCP app (local + UI)

Same widget mechanism, but the server runs locally inside an MCPB bundle. Use this when the widget needs to drive a **local** application — e.g., a file picker that browses the actual local disk, a dialog that controls a desktop app.

For MCPB packaging mechanics, defer to the **`build-mcpb`** skill. Everything below applies to both shapes.

---

## How widgets attach to tools

A widget-enabled tool has **two separate registrations**:

1. **The tool** declares a UI resource via `_meta.ui.resourceUri`. Its handler returns plain text/JSON — NOT the HTML.
2. **The resource** is registered separately and serves the HTML.

When Claude calls the tool, the host sees `_meta.ui.resourceUri`, fetches that resource, renders it in an iframe, and pipes the tool's return value into the iframe via the `ontoolresult` event.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { registerAppTool, registerAppResource, RESOURCE_MIME_TYPE }
  from "@modelcontextprotocol/ext-apps/server";
import { z } from "zod";

const server = new McpServer({ name: "contacts", version: "1.0.0" });

// 1. The tool — returns DATA, declares which UI to show
registerAppTool(server, "pick_contact", {
  description: "Open an interactive contact picker",
  inputSchema: { filter: z.string().optional() },
  _meta: { ui: { resourceUri: "ui://widgets/contact-picker.html" } },
}, async ({ filter }) => {
  const contacts = await db.contacts.search(filter);
  // Plain JSON — the widget receives this via ontoolresult
  return { content: [{ type: "text", text: JSON.stringify(contacts) }] };
});

// 2. The resource — serves the HTML
registerAppResource(
  server,
  "Contact Picker",
  "ui://widgets/contact-picker.html",
  {},
  async () => ({
    contents: [{
      uri: "ui://widgets/contact-picker.html",
      mimeType: RESOURCE_MIME_TYPE,
      text: pickerHtml,  // your HTML string
    }],
  }),
);
```

The URI scheme `ui://` is convention. The mime type MUST be `RESOURCE_MIME_TYPE` (`"text/html;profile=mcp-app"`) — this is how the host knows to render it as an interactive iframe, not just display the source.

---

## Widget runtime — the `App` class

Inside the iframe, your script talks to the host via the `App` class from `@modelcontextprotocol/ext-apps`. This is a **persistent bidirectional connection** — the widget stays alive as long as the conversation is active, receiving new tool results and sending user actions.

```html
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "ContactPicker", version: "1.0.0" }, {});

  // Set handlers BEFORE connecting
  app.ontoolresult = ({ content }) => {
    const contacts = JSON.parse(content[0].text);
    render(contacts);
  };

  await app.connect();

  // Later, when the user clicks something:
  function onPick(contact) {
    app.sendMessage({
      role: "user",
      content: [{ type: "text", text: `Selected contact: ${contact.id}` }],
    });
  }
</script>
```

| Method | Direction | Use for |
|---|---|---|
| `app.ontoolresult = fn` | Host → widget | Receive the tool's return value |
| `app.ontoolinput = fn` | Host → widget | Receive the tool's input args (what Claude passed) |
| `app.sendMessage({...})` | Widget → host | Inject a message into the conversation |
| `app.updateModelContext({...})` | Widget → host | Update context silently (no visible message) |
| `app.callServerTool({name, arguments})` | Widget → server | Call another tool on your server |

`sendMessage` is the typical "user picked something, tell Claude" path. `updateModelContext` is for state that Claude should know about but shouldn't clutter the chat.

**What widgets cannot do:**
- Access the host page's DOM, cookies, or storage
- Make network calls to arbitrary origins (CSP-restricted — route through `callServerTool`)

Keep widgets **small and single-purpose**. A picker picks. A chart displays. Don't build a whole sub-app inside the iframe — split it into multiple tools with focused widgets.

---

## Scaffold: minimal picker widget

**Install:**

```bash
npm install @modelcontextprotocol/sdk @modelcontextprotocol/ext-apps zod
```

**Server (`src/server.ts`):**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { registerAppTool, registerAppResource, RESOURCE_MIME_TYPE }
  from "@modelcontextprotocol/ext-apps/server";
import { readFileSync } from "node:fs";
import { z } from "zod";

const server = new McpServer({ name: "contact-picker", version: "1.0.0" });

const pickerHtml = readFileSync("./widgets/picker.html", "utf8");

registerAppTool(server, "pick_contact", {
  description: "Open an interactive contact picker. User selects one contact.",
  inputSchema: { filter: z.string().optional().describe("Name/email prefix filter") },
  _meta: { ui: { resourceUri: "ui://widgets/picker.html" } },
}, async ({ filter }) => {
  const contacts = await db.contacts.search(filter ?? "");
  return { content: [{ type: "text", text: JSON.stringify(contacts) }] };
});

registerAppResource(server, "Contact Picker", "ui://widgets/picker.html", {},
  async () => ({
    contents: [{ uri: "ui://widgets/picker.html", mimeType: RESOURCE_MIME_TYPE, text: pickerHtml }],
  }),
);

await server.connect(new StdioServerTransport());
```

**Widget (`widgets/picker.html`):**

```html
<!doctype html>
<meta charset="utf-8" />
<style>
  body { font: 14px system-ui; margin: 0; }
  ul { list-style: none; padding: 0; margin: 0; max-height: 300px; overflow-y: auto; }
  li { padding: 10px 14px; cursor: pointer; border-bottom: 1px solid #eee; }
  li:hover { background: #f5f5f5; }
  .sub { color: #666; font-size: 12px; }
</style>
<ul id="list"></ul>
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "ContactPicker", version: "1.0.0" }, {});
  const ul = document.getElementById("list");

  app.ontoolresult = ({ content }) => {
    const contacts = JSON.parse(content[0].text);
    ul.innerHTML = "";
    for (const c of contacts) {
      const li = document.createElement("li");
      li.innerHTML = `<div>${c.name}</div><div class="sub">${c.email}</div>`;
      li.addEventListener("click", () => {
        app.sendMessage({
          role: "user",
          content: [{ type: "text", text: `Selected contact: ${c.id} (${c.name})` }],
        });
      });
      ul.append(li);
    }
  };

  await app.connect();
</script>
```

See `references/widget-templates.md` for more widget shapes.

---

## Design notes that save you a rewrite

**One widget per tool.** Resist the urge to build one mega-widget that does everything. One tool → one focused widget → one clear result shape. Claude reasons about these far better.

**Tool description must mention the widget.** Claude only sees the tool description when deciding what to call. "Opens an interactive picker" in the description is what makes Claude reach for it instead of guessing an ID.

**Widgets are optional at runtime.** Hosts that don't support the apps surface simply ignore `_meta.ui` and render the tool's text content normally. Since your tool handler already returns meaningful text/JSON (the widget's data), degradation is automatic — Claude sees the data directly instead of via the widget.

**Don't block on widget results for read-only tools.** A widget that just *displays* data (chart, preview) shouldn't require a user action to complete. Return the display widget *and* a text summary in the same result so Claude can continue reasoning without waiting.

---

## Testing

- **Local:** point Claude desktop's MCP config at your server, trigger the tool, check the widget renders and `sendMessage` flows back into the chat.
- **Host fallback:** disable the apps surface (or use a host without it) and confirm the tool degrades gracefully.
- **CSP:** open browser devtools on the iframe — CSP violations are the #1 reason widgets silently fail.

---

## Reference files

- `references/widget-templates.md` — reusable HTML scaffolds for picker / confirm / progress / display
- `references/apps-sdk-messages.md` — the `App` class API: widget ↔ host ↔ server messaging
