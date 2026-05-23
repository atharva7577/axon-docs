# Add AXON to your IDE

AXON is a hosted, OpenAI-compatible endpoint. If your IDE supports custom model providers, you can add AXON to its model dropdown in under a minute — no plugin, no install, no friction.

```
Base URL:  https://api.axon.nexalyte.tech/v1
API key:   <your AXON key>      # request one at console.nexalyte.tech
Model:     axon-auto            # any string works — AXON re-routes internally
```

That's it. Every request runs through AXON's intent-aware router and per-workspace memory. You keep your IDE, AXON keeps the spend down.

**Agent mode works too.** AXON forwards OpenAI-spec `tools`, `tool_choice`, and `tool_calls` (streaming + non-streaming) end-to-end, translating to/from Anthropic's `tool_use` and Gemini's `functionCall` formats under the hood. So Continue/Cursor/Cline/Roo Code Agent loops — the ones where the model reads files, runs commands, edits diffs — route through AXON just like chat does. Cost savings compound across the loop as the routing-experience store learns which model actually closes each tool turn.

---

## IDE coverage at a glance

| IDE                  | Status  | Chat dropdown | Agent mode (tool calling) |
|----------------------|---------|---------------|---------------------------|
| **Cursor**           | ✅ Today | "AXON" entry | ✅ Tool-call pass-through verified |
| **Continue**         | ✅ Today | "AXON" entry | ✅ Tool-call pass-through verified |
| **Cline**            | ✅ Today | "AXON" entry | ✅ Multi-step tool loops route through AXON |
| **Roo Code**         | ✅ Today | "AXON" entry | ✅ Same as Cline |
| **Aider**            | ✅ Today | CLI flag | ✅ Architect-mode tool calls pass through |
| **Windsurf**         | ✅ Today | "AXON" entry | ⚠️  Cascade-specific tool surface — not all features cross the wire |
| **Claude Code**      | ⚠️  Soon | No | Pending Anthropic-compat shim |
| **Google Antigravity** | ❌ No | Gemini-locked | Pending Gemini-compat shim |
| **GitHub Copilot**   | ❌ No   | Fixed model list | Fixed model list, no custom providers |

---

## Cursor

1. **Cursor Settings** → **Models** → **+ Add custom model**.
2. Pick **OpenAI** as the provider type.
3. Fill in:
   - **Model name:** `axon-auto`
   - **Base URL:** `https://api.axon.nexalyte.tech/v1`
   - **API key:** your AXON key (any string works if AXON is in shared-key mode)
4. Toggle the new "axon-auto" entry **on** in the model list.

Open Cursor's chat. The model picker now shows **axon-auto** alongside Sonnet, GPT-4o, etc. Pick it and chat normally — Cursor's `@Codebase`, inline edits, and tab autocomplete all route through AXON.

### Enabling Agent mode (Composer)

Cursor's Composer / Agent doesn't send tool definitions to a custom OpenAI provider unless that provider's entry has the **"Supports tool calls"** (or "function calling") box ticked. Without it Composer falls back to chat-only behavior — Agent mode looks active in the UI but file reads / terminal calls never reach AXON.

- **Cursor Settings → Models → your axon-auto entry → Advanced → ✅ Supports tool calls.**
- Restart Cursor.

Some Cursor builds auto-detect tool support by model-ID prefix (`gpt-4o`, `claude-3.5`, etc.) and silently disable it for unknown names like `axon-auto`. If you can't find the toggle, name the model `gpt-4o` in your Cursor config (still pointing at AXON's base URL) — AXON ignores the model field, the router still picks correctly, and Cursor flips tools on by name-match.

> **BYOK variant:** to use your own OpenAI/Anthropic keys instead of AXON's pooled key, set **Override OpenAI Base URL** to `https://api.axon.nexalyte.tech/v1` under **Cursor Settings → Models → OpenAI API Key**, and paste your real key. AXON proxies and routes through your key.

---

## Continue

**`~/.continue/config.json`:**

```json
{
  "models": [
    {
      "title": "AXON",
      "provider": "openai",
      "model": "axon-auto",
      "apiBase": "https://api.axon.nexalyte.tech/v1",
      "apiKey": "<your AXON key>"
    }
  ],
  "tabAutocompleteModel": {
    "title": "AXON autocomplete",
    "provider": "openai",
    "model": "axon-auto",
    "apiBase": "https://api.axon.nexalyte.tech/v1",
    "apiKey": "<your AXON key>"
  }
}
```

**`~/.continue/config.yaml`** (newer Continue versions):

```yaml
models:
  - name: AXON
    provider: openai
    model: axon-auto
    apiBase: https://api.axon.nexalyte.tech/v1
    apiKey: <your AXON key>
```

Reload Continue. "AXON" appears in the model picker.

### Enabling Agent mode (REQUIRED for tool use)

Continue's Agent mode only forwards its file-read / edit / run tools to a model that explicitly **declares tool-use capability**. Without the declaration the model sees a plain chat prompt with no tools attached — you get "I can't access the codebase" non-answers even with Agent toggled on.

Add `capabilities` and `roles` to your AXON model entry:

**`config.yaml`:**

```yaml
models:
  - name: AXON
    provider: openai
    model: axon-auto
    apiBase: https://api.axon.nexalyte.tech/v1
    apiKey: <your axon_live_... key>
    roles:
      - chat
      - edit
      - apply
    capabilities:
      - tool_use
    defaultCompletionOptions:
      contextLength: 128000
```

**`config.json`:** add `"capabilities": ["tool_use"]` and `"roles": ["chat", "edit", "apply"]` to the model entry.

Reload Continue, switch the chat input from "Chat" to "Agent" (top-left of the input box), then ask "go through the codebase and tell me…" — you should see Continue render tool-call cards ("Read file: …", "Run command: …") before the natural-language reply, and AXON's logs should show `[ToolMode] selected=… tools=<N>`.

If you stay in Chat mode without `@-mentions`, AXON sees only the raw prompt and can't introspect your code — that's a Continue contract, not an AXON limit.

---

## Cline (VS Code)

1. Open the Cline settings panel (gear icon in the Cline sidebar).
2. **API Provider** → **OpenAI Compatible**.
3. Fill in:
   - **Base URL:** `https://api.axon.nexalyte.tech/v1`
   - **API Key:** your AXON key
   - **Model ID:** `axon-auto`
4. Save.

Cline's model dropdown now lists `axon-auto`. All of Cline's multi-step tool calls route through AXON — the routing-experience store learns from the first few calls in a task and routes subsequent ones cheaper, so savings compound on longer tasks.

### Enabling Agent mode

Nothing to enable. Cline is tool-loop-first by design — every request to your provider includes its tool definitions, regardless of which model you picked. As long as the model entry points at AXON (which now does OpenAI-shape tool calling end-to-end), Cline's loop drives correctly.

---

## Roo Code (VS Code)

Roo Code (the Cline fork) uses the same custom-provider flow:

1. **Roo Code Settings** → **Providers** → **Add Provider**.
2. Choose **OpenAI Compatible**.
3. **Base URL:** `https://api.axon.nexalyte.tech/v1`
4. **API Key:** your AXON key
5. **Model:** `axon-auto`

Save. Roo Code's chat dropdown shows the new provider.

### Enabling Agent mode

Same as Cline — nothing to enable. Roo Code inherits Cline's tool-loop architecture and always sends tools on every request.

---

## Aider (CLI)

```bash
export OPENAI_API_KEY="<your AXON key>"
export OPENAI_API_BASE="https://api.axon.nexalyte.tech/v1"

aider --model openai/axon-auto
```

Or one-liner:

```bash
aider --openai-api-base https://api.axon.nexalyte.tech/v1 \
      --openai-api-key <your AXON key> \
      --model openai/axon-auto
```

Aider's a CLI tool, so there's no dropdown — the `--model` flag is the equivalent.

### Enabling Agent mode

Plain `aider` runs in prompt-only mode (no tools). For agentic file-touching behavior use **`--architect`** mode:

```bash
aider --architect \
      --openai-api-base https://api.axon.nexalyte.tech/v1 \
      --openai-api-key <your AXON key> \
      --model openai/axon-auto
```

In architect mode aider sends tool calls (edit, run, search) which AXON forwards to the routed model. Without `--architect` you'll get text suggestions instead of applied edits.

---

## Windsurf

Windsurf's Cascade supports a **Custom Provider** (BYOK) mode that accepts OpenAI-compatible endpoints.

1. **Windsurf Settings** → **Cascade** → **Models / Providers** → **Add Custom Provider**.
2. Pick **OpenAI Compatible**.
3. Fill in:
   - **Base URL:** `https://api.axon.nexalyte.tech/v1`
   - **API Key:** your AXON key
   - **Model:** `axon-auto`
4. Save and select the new provider in Cascade's chat.

> **Caveat:** Windsurf's custom-provider surface has been less stable than Cursor's. If the model doesn't appear after save, restart Windsurf. If a specific Cascade feature (Flows, Memories) refuses non-Codeium providers, that's an upstream Windsurf restriction — not an AXON limitation.

### Enabling Agent mode

Like Cursor, Windsurf needs the custom-provider entry to declare tool support before Cascade will send tools through. Under **Windsurf Settings → Cascade → Custom Provider → axon-auto → Advanced**, tick **"Supports function calling"** (label varies by Windsurf version). Restart Windsurf. If the toggle is missing in your build, rename the model to `gpt-4o` in the config so Windsurf's name-prefix detection enables tools automatically.

Several Cascade features (Flows, Memories, Cascade Lite) are coded to refuse non-Codeium providers regardless of this toggle — that's an upstream Windsurf restriction, no fix on the AXON side.

---

## OpenAI Python SDK

```python
from openai import OpenAI

client = OpenAI(
    api_key="<your AXON key>",
    base_url="https://api.axon.nexalyte.tech/v1",
)

response = client.chat.completions.create(
    model="axon-auto",
    messages=[{"role": "user", "content": "Reverse a linked list in Python."}],
    stream=True,
)
```

Async, LangChain, LlamaIndex, Vercel AI SDK — anything that takes a `base_url` works the same way.

---

## OpenAI JS/TS SDK

```typescript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "<your AXON key>",
  baseURL: "https://api.axon.nexalyte.tech/v1",
});

const stream = await client.chat.completions.create({
  model: "axon-auto",
  messages: [{ role: "user", content: "Explain the JS event loop." }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

---

## MCP (Model Context Protocol)

AXON's relationship to MCP is bidirectional — read whichever direction applies to you:

- **Your IDE has MCP servers → tools flow *through* AXON.** Nothing to configure. Once your IDE (Cursor, Continue, Cline, Claude Code, etc.) collects tools from its MCP servers, it ships them to the model as standard OpenAI `tools` in `/v1/chat/completions`. AXON's tool-calling pass-through forwards them verbatim to OpenAI, Anthropic, or Google, and forwards the resulting `tool_calls` back. There is no MCP-specific configuration on the AXON side for this path — agent mode just works (provided the IDE's agent mode is enabled, see per-IDE sections above).
  - **Gemini caveat:** Gemini enforces tool-name regex `^[a-zA-Z_][a-zA-Z0-9_-]{0,63}$` and rejects deeply-nested or `$ref`/`oneOf`-heavy JSON schemas. MCP tool names with dots or slashes, or schemas generated by some MCP servers, can 400 against Gemini specifically. AXON does not sanitize — if Gemini complains, route around it by pinning a different provider (e.g. `provider_override: "openai"`) or rename the offending tool upstream.

- **You want to consume AXON's own primitives as MCP tools.** AXON exposes its routing/memory primitives as native MCP tools. Three tools: `route_for_intent`, `get_edit_pattern`, `query_workspace_memory`. Use the configs below.

**HTTP transport (Cursor, Continue, Cline, any HTTP-aware MCP client) — recommended:**

```json
{
  "mcpServers": {
    "axon": {
      "url":     "https://api.axon.nexalyte.tech/mcp",
      "headers": { "Authorization": "Bearer <your AXON key>" }
    }
  }
}
```

**Claude Desktop / Claude Code (stdio-only clients):**

These clients don't speak HTTP MCP yet, so bridge through `mcp-remote` (a maintained third-party stdio→HTTP shim):

```json
{
  "mcpServers": {
    "axon": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote",
        "https://api.axon.nexalyte.tech/mcp",
        "--header", "Authorization:Bearer <your AXON key>"
      ]
    }
  }
}
```

No AXON package needs to be installed — `mcp-remote` is published on npm and handles the transport bridge.

---

## Any other OpenAI-compatible tool

If your tool accepts a base URL override, AXON works. The pattern is always:

| Setting               | Value                                  |
|-----------------------|----------------------------------------|
| Base URL / API base   | `https://api.axon.nexalyte.tech/v1`        |
| API key               | your AXON key (or your real provider key for BYOK) |
| Model name            | `axon-auto` (or any string)            |

---

## IDEs that don't support AXON yet

Some IDEs lock the model picker to their own provider. We're working on shims for the ones worth supporting.

### Claude Code (Anthropic CLI)

Claude Code's model dropdown is hardcoded to Anthropic models. The dropdown can't be extended, but Claude Code does respect `ANTHROPIC_BASE_URL`. An Anthropic-compatible `/v1/messages` proxy on AXON is **planned** — once shipped:

```bash
export ANTHROPIC_BASE_URL=https://api.axon.nexalyte.tech
export ANTHROPIC_API_KEY=<your AXON key>
claude
```

The dropdown will still show "Sonnet 4.6" etc., but every request flows through AXON. AXON will respect the user's choice as a quality ceiling and route down when KNN says it's safe.

### Google Antigravity

Antigravity is Gemini-locked at the model layer with no custom-provider setting. A Gemini-compatible (`generateContent` schema) shim on AXON would unlock it, but isn't built yet. Track Google's BYOK roadmap; if they add OpenAI-compat support natively, AXON will plug in immediately.

### GitHub Copilot

Fixed model list, no custom providers. No path today.

---

## Verifying it's working

After your first request:

1. Open the dashboard: <https://console.nexalyte.tech> → **Analytics**.
2. The **Cost Savings Over Time** chart should show a new bar within ~30s.
3. The **Recent Traces** table shows the model AXON actually picked vs. the model you asked for.

If you see your request in the traces table, AXON is in the path. If you don't, double-check the base URL — the single most common setup mistake is `/v1` missing from the end.

---

## Getting an AXON key

AXON is in invite-only beta. Keys are operator-issued, not self-serve. To request one, reach out via [contact channel] — you'll receive a copy-paste onboarding message containing:

- Your `axon_live_…` API key (shown **once** — save it; we can't recover it)
- Your tenant ID
- A 5-second `curl` to verify the key works
- Per-IDE config snippets matching the ones above

Once you have the key, paste it at <https://console.nexalyte.tech> to see your tenant dashboard: live cost, model distribution, accept-rate, recent traces.

### Pooled vs. BYOK — pick the right mode for your flow

Two modes, two natural fits:

**Pooled (recommended for IDE dropdown flows)** — AXON spends from its own provider credits and bills you a flat subscription. You configure one key (your AXON key) in the IDE and you're done. This is the only mode that works cleanly with IDEs whose custom-provider UI lacks a custom-headers slot (most of them — Cursor in particular).

**BYOK (recommended for SDK / MCP flows)** — you forward your own provider key (`x-openai-key`, `x-anthropic-key`, or `x-google-key`) on every request, so AXON spends *your* provider credit and just charges for the routing+memory layer. Trivial from code (one extra header from your `.env`); awkward from an IDE GUI. AXON still does the full routing and never sees the provider key in storage.

The flag is `tenants.byok_required` (per-tenant; defaults to `true` for new tenants today). Operator flips it via `POST /v1/admin/tenants/:id/byok-required` — request pooled-mode access during onboarding and the operator will set it for you.

---

## MCP — Model Context Protocol

AXON hosts an MCP endpoint at `POST /mcp` (same host, same Bearer auth as `/v1/chat/completions`). Any MCP-compatible client — Claude Desktop, Cursor, Cline, Continue — can point at it and pick up two layers of tools in one place:

1. **AXON-native tools** (3): `route_for_intent`, `get_edit_pattern`, `query_workspace_memory`. These let an agent introspect AXON's routing memory before making an LLM call.
2. **Curated catalog of community MCP servers** that the operator has enabled for your tenant. AXON spawns them as long-lived stdio subprocesses on the AXON host and namespaces their tools (`fetch:get`, `time:now`, `sequential-thinking:think`, ...). Per-tenant allowlist controls what each tenant can see and call.

### Phase 1 catalog (server-side execution)

| id                    | what it does                                                  | needs |
|-----------------------|---------------------------------------------------------------|-------|
| `fetch`               | Fetch a URL → HTML-to-markdown                                | — |
| `sequential-thinking` | Structured chain-of-thought scratch-pad                       | — |
| `time`                | Timezone-aware time, ISO conversions                          | — |
| `brave-search`        | Web search via Brave API (operator-shared key)                | `BRAVE_API_KEY` on host |

The operator enables these per tenant from the **Admin → Tools** tab. Toggle state takes effect on the tenant's next `/mcp` request — no restart needed.

### Pointing a client at AXON's MCP surface

Claude Desktop (`~/.claude/settings.json`):

```json
{
  "mcpServers": {
    "axon": {
      "type":  "http",
      "url":   "https://api.axon.nexalyte.tech/mcp",
      "headers": { "Authorization": "Bearer <your AXON key>" }
    }
  }
}
```

Cursor / Cline / Continue / Claude Code follow the same pattern — point them at the same URL with the same header.

### Client-local tools (filesystem, git on your workspace)

AXON deliberately does **not** host filesystem/git MCP servers on the AXON VPS — those servers expect access to *your* workspace, not the operator's. Wire them into your local MCP client config *alongside* AXON's surface. The agent sees one merged tools list and uses each tool from wherever it lives.

```json
{
  "mcpServers": {
    "axon":       { "type": "http", "url": "https://api.axon.nexalyte.tech/mcp", "headers": { "Authorization": "Bearer …" } },
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/code"] },
    "git":        { "command": "uvx", "args": ["mcp-server-git", "--repository", "/Users/me/code/myrepo"] }
  }
}
```

### Operator setup

```sh
cd backend && bash scripts/mcp-install.sh
```

That warms `npx`/`uvx` caches for the catalog packages. Then in the admin UI Tools tab, pick a tenant and tick the catalog entries to expose.
