# Widget Templates

Minimal HTML scaffolds for the common widget shapes. Copy, fill in, ship.

All templates use the `App` class from `@modelcontextprotocol/ext-apps` via ESM CDN. They're intentionally framework-free — widgets are small enough that React/Vue hydration cost usually isn't worth it.

---

## Serving widget HTML

Widgets are static HTML — data arrives at runtime via `ontoolresult`, not baked in. Store each widget as a string constant or read from disk:

```typescript
import { readFileSync } from "node:fs";
import { registerAppResource, RESOURCE_MIME_TYPE } from "@modelcontextprotocol/ext-apps/server";

const pickerHtml = readFileSync("./widgets/picker.html", "utf8");

registerAppResource(server, "Picker", "ui://widgets/picker.html", {},
  async () => ({
    contents: [{ uri: "ui://widgets/picker.html", mimeType: RESOURCE_MIME_TYPE, text: pickerHtml }],
  }),
);
```

---

## Picker (single-select list)

```html
<!doctype html>
<meta charset="utf-8" />
<style>
  body { font: 14px system-ui; margin: 0; }
  ul { list-style: none; padding: 0; margin: 0; max-height: 280px; overflow-y: auto; }
  li { padding: 10px 14px; cursor: pointer; border-bottom: 1px solid #eee; }
  li:hover { background: #f5f5f5; }
  .sub { color: #666; font-size: 12px; }
</style>
<ul id="list"></ul>
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "Picker", version: "1.0.0" }, {});
  const ul = document.getElementById("list");

  app.ontoolresult = ({ content }) => {
    const { items } = JSON.parse(content[0].text);
    ul.innerHTML = "";
    for (const it of items) {
      const li = document.createElement("li");
      li.innerHTML = `<div>${it.label}</div><div class="sub">${it.sub ?? ""}</div>`;
      li.addEventListener("click", () => {
        app.sendMessage({
          role: "user",
          content: [{ type: "text", text: `Selected: ${it.id}` }],
        });
      });
      ul.append(li);
    }
  };

  await app.connect();
</script>
```

**Tool returns:** `{ content: [{ type: "text", text: JSON.stringify({ items: [{ id, label, sub? }] }) }] }`

---

## Confirm dialog

```html
<!doctype html>
<meta charset="utf-8" />
<style>
  body { font: 14px system-ui; margin: 16px; }
  .actions { display: flex; gap: 8px; margin-top: 16px; }
  button { padding: 8px 16px; cursor: pointer; }
  .danger { background: #d33; color: white; border: none; }
</style>
<p id="msg"></p>
<div class="actions">
  <button id="cancel">Cancel</button>
  <button id="confirm" class="danger">Confirm</button>
</div>
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "Confirm", version: "1.0.0" }, {});

  app.ontoolresult = ({ content }) => {
    const { message, confirmLabel } = JSON.parse(content[0].text);
    document.getElementById("msg").textContent = message;
    if (confirmLabel) document.getElementById("confirm").textContent = confirmLabel;
  };

  await app.connect();

  document.getElementById("confirm").addEventListener("click", () => {
    app.sendMessage({ role: "user", content: [{ type: "text", text: "Confirmed." }] });
  });
  document.getElementById("cancel").addEventListener("click", () => {
    app.sendMessage({ role: "user", content: [{ type: "text", text: "Cancelled." }] });
  });
</script>
```

**Tool returns:** `{ content: [{ type: "text", text: JSON.stringify({ message, confirmLabel? }) }] }`

**Note:** For simple confirmation, prefer **elicitation** over a widget — see `../build-mcp-server/references/elicitation.md`. Use this widget when you need custom styling or context beyond what a native form offers.

---

## Progress (long-running)

```html
<!doctype html>
<meta charset="utf-8" />
<style>
  body { font: 14px system-ui; margin: 16px; }
  .bar { height: 8px; background: #eee; border-radius: 4px; overflow: hidden; }
  .fill { height: 100%; background: #2a7; transition: width 200ms; }
</style>
<p id="label">Starting…</p>
<div class="bar"><div id="fill" class="fill" style="width:0%"></div></div>
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "Progress", version: "1.0.0" }, {});
  const label = document.getElementById("label");
  const fill = document.getElementById("fill");

  // The tool result fires when the job completes — intermediate updates
  // arrive via the same handler if the server streams them
  app.ontoolresult = ({ content }) => {
    const state = JSON.parse(content[0].text);
    if (state.progress !== undefined) {
      label.textContent = state.message ?? `${state.progress}/${state.total}`;
      fill.style.width = `${(state.progress / state.total) * 100}%`;
    }
    if (state.done) {
      label.textContent = "Complete";
      fill.style.width = "100%";
    }
  };

  await app.connect();
</script>
```

Server side, emit progress via `extra.sendNotification({ method: "notifications/progress", ... })` — see `apps-sdk-messages.md`.

---

## Display-only (chart / preview)

Display widgets don't call `sendMessage` — they render and sit there. The tool should return a text summary **alongside** the widget so Claude can keep reasoning while the user sees the visual:

```typescript
registerAppTool(server, "show_chart", {
  description: "Render a revenue chart",
  inputSchema: { range: z.enum(["week", "month", "year"]) },
  _meta: { ui: { resourceUri: "ui://widgets/chart.html" } },
}, async ({ range }) => {
  const data = await fetchRevenue(range);
  return {
    content: [{
      type: "text",
      text: `Revenue is up ${data.change}% over the ${range}. Chart rendered.\n\n` +
            JSON.stringify(data.points),
    }],
  };
});
```

```html
<!doctype html>
<meta charset="utf-8" />
<style>body { font: 14px system-ui; margin: 12px; }</style>
<canvas id="chart" width="400" height="200"></canvas>
<script type="module">
  import { App } from "https://esm.sh/@modelcontextprotocol/ext-apps@1.2.2";

  const app = new App({ name: "Chart", version: "1.0.0" }, {});

  app.ontoolresult = ({ content }) => {
    // Parse the JSON points from the text content (after the summary line)
    const text = content[0].text;
    const jsonStart = text.indexOf("\n\n") + 2;
    const points = JSON.parse(text.slice(jsonStart));
    drawChart(document.getElementById("chart"), points);
  };

  await app.connect();

  function drawChart(canvas, points) { /* ... */ }
</script>
```
