# ext-apps messaging — widget ↔ host ↔ server

The `@modelcontextprotocol/ext-apps` package provides the `App` class (browser side) and `registerAppTool`/`registerAppResource` helpers (server side). Messaging is bidirectional and persistent.

---

## Widget → Host

### `app.sendMessage({ role, content })`

Inject a visible message into the conversation. This is how user actions become conversation turns.

```js
app.sendMessage({
  role: "user",
  content: [{ type: "text", text: "User selected order #1234" }],
});
```

The message appears in chat and Claude responds to it. Use `role: "user"` — the widget speaks on the user's behalf.

### `app.updateModelContext({ content })`

Update Claude's context **silently** — no visible message. Use for state that informs but doesn't warrant a chat bubble.

```js
app.updateModelContext({
  content: [{ type: "text", text: "Currently viewing: orders from last 30 days" }],
});
```

### `app.callServerTool({ name, arguments })`

Call a tool on your MCP server directly, bypassing Claude. Returns the tool result.

```js
const result = await app.callServerTool({
  name: "fetch_order_details",
  arguments: { orderId: "1234" },
});
```

Use for data fetches that don't need Claude's reasoning — pagination, detail lookups, refreshes.

---

## Host → Widget

### `app.ontoolresult = ({ content }) => {...}`

Fires when the tool handler's return value is piped to the widget. This is the primary data-in path.

```js
app.ontoolresult = ({ content }) => {
  const data = JSON.parse(content[0].text);
  renderUI(data);
};
```

**Set this BEFORE `await app.connect()`** — the result may arrive immediately after connection.

### `app.ontoolinput = ({ arguments }) => {...}`

Fires with the arguments Claude passed to the tool. Useful if the widget needs to know what was asked for (e.g., highlight the search term).

---

## Server → Widget (progress)

For long-running operations, emit progress notifications. The client sends a `progressToken` in the request's `_meta`; the server emits against it.

```typescript
// In the tool handler
async ({ query }, extra) => {
  const token = extra._meta?.progressToken;
  for (let i = 0; i < steps.length; i++) {
    if (token !== undefined) {
      await extra.sendNotification({
        method: "notifications/progress",
        params: { progressToken: token, progress: i, total: steps.length, message: steps[i].name },
      });
    }
    await steps[i].run();
  }
  return { content: [{ type: "text", text: "Complete" }] };
}
```

No `{ notify }` destructure — `extra` is `RequestHandlerExtra`; progress goes through `sendNotification`.

---

## Lifecycle

1. Claude calls a tool with `_meta.ui.resourceUri` declared
2. Host fetches the resource (your HTML) and renders it in an iframe
3. Widget script runs, sets handlers, calls `await app.connect()`
4. Host pipes the tool's return value → `ontoolresult` fires
5. Widget renders, user interacts
6. Widget calls `sendMessage` / `updateModelContext` / `callServerTool` as needed
7. Widget persists until conversation context moves on — subsequent calls to the same tool reuse the iframe and fire `ontoolresult` again

There's no explicit "submit and close" — the widget is a long-lived surface.

---

## CSP gotchas

The iframe runs under a restrictive Content-Security-Policy:

| Symptom | Cause | Fix |
|---|---|---|
| Widget renders but JS doesn't run | Inline event handlers blocked | Use `addEventListener` — never `onclick="..."` in HTML |
| `eval` / `new Function` errors | Script-src restriction | Don't use them; use JSON.parse for data |
| External scripts fail | CDN not allowlisted | `esm.sh` is safe; avoid others |
| `fetch()` to your API fails | Cross-origin blocked | Route through `app.callServerTool()` instead |
| External CSS doesn't load | `style-src` restriction | Inline styles in a `<style>` tag |
| Fonts don't load | `font-src` restriction | Use system fonts (`font: 14px system-ui`) |

When in doubt, open the iframe's devtools console — CSP violations log there.
