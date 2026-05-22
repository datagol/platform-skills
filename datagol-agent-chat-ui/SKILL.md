---
name: datagol-agent-chat-ui
description: How to generate a Vite + React chat UI that streams from a DataGOL custom agent (created via datagol_create_agent). Use this when the user wants a website / app that talks to the agent.
---

# Agent Chat UI Generation

Use this skill when the user asks for a "chatbot website", "chat app", "frontend for my agent", "interface to talk to my agent", or similar. It pairs naturally with `datagol-create-agent`: first create the agent on DataGOL, then generate a UI that invokes it.

> **Prerequisite: run `datagol-interview` first.** The user-management answer
> (mandatory question §2) decides whether this chat app shows a sign-in
> screen or just consumes a paste-token. The interview output also
> determines how many conversation starters to scaffold and what UX to
> render. After the interview, present a `datagol-detailed-plan` before writing
> any chat-app files.

## Auth — two tokens, two purposes

Generated apps use **two different auth credentials** depending on which endpoint they call:

| Endpoint type | Header | Credential |
|---|---|---|
| Workbook CRUD (`/noCo/api/v2/*`) | `x-auth-token` | `DATAGOL_SERVICE_TOKEN` |
| Agent chat (`/ai/api/v2/*`, streaming) | `Authorization: Bearer` | `DATAGOL_USER_TOKEN` |

> ⚠️ **Agents cannot be created under the service account.** `POST /customAgents/api/v1` with `x-auth-token: SERVICE_TOKEN` returns `400 "Either userId, email, or anonymousUserId is required"`. Agents must be created with `Authorization: Bearer ${DATAGOL_TOKEN}` (the platform user's bearer), which makes the **platform user** the owner. As a result, all agent conversation and streaming calls must also use `Authorization: Bearer DATAGOL_USER_TOKEN` — the service token cannot access user-owned agents.

**Two tokens, set up at scaffold time:**

```ts
// src/config.ts
export const DATAGOL_SERVICE_TOKEN = '<service token>';            // long-lived, for workbook ops
export const DATAGOL_USER_TOKEN    = import.meta.env.VITE_DATAGOL_USER_TOKEN;  // for agent ops
export const AGENT_ID = '<agent id from creation>';
```

`DATAGOL_USER_TOKEN` is **never hardcoded in source**. It comes from the `VITE_DATAGOL_USER_TOKEN` env var, which Vite bakes in at build time:
- **Dev / sandbox**: write to `.env.local` (gitignored). Value = `process.env.DATAGOL_TOKEN` available at scaffold time.
- **Vercel / production**: set in Vercel dashboard → Settings → Environment Variables. To rotate an expired token: update the var and redeploy — no code changes.

Always add `src/vite-env.d.ts` so TypeScript knows about the env var:
```ts
// src/vite-env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv { readonly VITE_DATAGOL_USER_TOKEN: string; }
interface ImportMeta { readonly env: ImportMetaEnv; }
```

Always add `.env.local` and `.env*` to `.gitignore`.

```ts
// src/lib/agent.ts — agent calls use Bearer, not x-auth-token
import { DATAGOL_USER_TOKEN } from '../config';

function headers() {
  return { 'Authorization': `Bearer ${DATAGOL_USER_TOKEN}`, 'Content-Type': 'application/json' };
}
```

Get the platform user's numeric `userId` via `GET /idp/api/v1/user` with the bearer at scaffold time and hardcode it in `createConversation()`. Don’t fetch it at runtime.

**`dgFetch` (workbook ops) is unchanged** — it still uses `x-auth-token: DATAGOL_SERVICE_TOKEN`. Only `src/lib/agent.ts` switches to the user bearer.

## Endpoints (two-step flow)

Invoking the agent is **two HTTP calls per chat thread**: first create a
conversation server-side, then stream messages into it.

### 0. Fetch the userId (once, cached)

The conversation endpoint requires a numeric `userId` field. Fetch it from:

```
GET ${DATAGOL_BASE_URL}/idp/api/v1/user
x-auth-token: <DATAGOL_SERVICE_TOKEN>
```

Response shape: `{ id: number, ... }`. Cache the value in a module-level variable so the call only happens once per session.

```ts
let _userId: number | null = null;
async function getUserId(): Promise<number> {
  if (_userId !== null) return _userId;
  const res = await fetch(`${DATAGOL_BASE_URL}/idp/api/v1/user`, { headers: headers() });
  if (!res.ok) throw new Error(`Failed to fetch user: HTTP ${res.status}`);
  const data = (await res.json()) as { id?: number };
  if (!data.id) throw new Error('id missing from /idp/api/v1/user response');
  _userId = data.id;
  return _userId;
}
```

### 1. Create a conversation (once per thread)

```
POST ${DATAGOL_BASE_URL}/ai/api/v2/conversations
x-auth-token: <DATAGOL_SERVICE_TOKEN>
Content-Type: application/json

{
  "name":      "<short label, e.g. truncated first message>",
  "active":    true,
  "scope":     "PRIVATE",
  "userId":    <userId from step 0>,
  "agentType": "CustomAgent",
  "uiMetadata": {
    "type": "CustomAgent",
    "parameters": {
      "customAgentId": "<AGENT_ID>",
      "parameters":    { "customAgentId": "<AGENT_ID>" }
    },
    "configuration": { "llmModel": "claude-sonnet-4-6" }
  }
}
```

Response includes `id` — the **conversationId** to reuse in step 2 for every
message in this thread.

### 2. Stream a message (per turn)

```
# ⚠️ Streaming lives on a SEPARATE host. Use DATAGOL_AI_URL from
# src/config.ts (set up by datagol-app-auth — note: prod and test
# hosts don't share a naming pattern):
#   prod: https://ai.datagol.ai/ai/api/v2/messages/streaming
#   test: https://testing-core-ai.datagol.ai/ai/api/v2/messages/streaming
POST ${DATAGOL_AI_URL}/ai/api/v2/messages/streaming
x-auth-token: <DATAGOL_SERVICE_TOKEN>
Content-Type: application/json

{
  "agentType":        "CustomAgent",
  "type":             "CustomAgent",
  "conversationId":   "<id from step 1>",
  "message":          "<user input>",
  "parameters":       { "customAgentId": "<AGENT_ID>" },
  "uiMetadata":       { "parameters": {} },
  "selectedLlmModel": "claude-sonnet-4-6"
}
```

Response is **Server-Sent Events** (`text/event-stream`). The wire format:

- **Records are separated by blank lines** (`\n\n`).
- **Comment lines** start with `:` (e.g. `:heartbeat`, `:keep-alive`). Skip them.
- **Payload lines** are JSON objects. They may appear with OR without a leading `data:` prefix — the parser must tolerate both. Strip `data:` if present, then `JSON.parse`.

Each payload object has the shape:

```json
{ "content": <varies>, "response_type": "<type>", "metadata": null }
```

`response_type` values you must handle:

| Type | `content` shape | What to do |
|---|---|---|
| `message_id` | number | First event of the turn. Server-assigned id. Stash if you need a handle; otherwise ignore. |
| `text` | string | A chunk of assistant text (often a few words / one bullet at a time, with markdown). **Append** to the in-progress assistant message — don't replace. |
| `tool_call` | `{ action, input, tool_call_id }` | Agent is invoking a tool (e.g. `sales-activity_get_table_schema`). Render an inline "running tool" indicator. Match the `tool_call_id` to its result later. |
| `subagent_task_end` | `{ tool_call_id, tool_response: [{ type: "text", text: "<JSON string>" }] }` | Tool finished. **`tool_response[0].text` is a JSON-encoded string** — `JSON.parse` it again to get the actual payload (e.g. a workbook schema). Mark the matching `tool_call` as done. |
| `html` | string | A complete HTML/SVG fragment — typically a chart, table, or other visualization the agent built. Render it in a **sandboxed `<iframe srcdoc>`** so its `<style>` doesn't leak and any embedded scripts can't access the host page. See "HTML rendering" below. **Don't** dump it via `dangerouslySetInnerHTML`. |
| any other | — | Ignore safely (forward-compat). Don't crash. |

The stream has **no explicit `done` event** — completion is signalled by the HTTP response ending (`reader.read()` returns `done: true`).

### HTML rendering (charts / visualizations)

The agent emits `response_type: "html"` when it has produced a complete HTML or SVG fragment — typically a chart drawn from workbook data, with its own `<style>` block. Render each such fragment in an isolated iframe:

- **`<iframe srcdoc={html} sandbox="allow-scripts">`** — `sandbox` (without `allow-same-origin`) blocks the iframe from reading the parent's `localStorage`, cookies, or DOM. Inline `<style>` and `<svg>` work fine. Inline `<script>` runs but can't escape.
- Auto-size to content height: have the iframe `postMessage` its scroll height to the parent on `load` + `ResizeObserver`. Parent updates the iframe's `height` style. The scaffold below has the snippet.
- Never use `dangerouslySetInnerHTML`. The HTML originates from a trusted backend, but iframe sandboxing is cheap insurance and keeps the page's CSS clean (the chart's `<style>` would otherwise leak to your message bubbles).

### Embedded entity links in text

Assistant text chunks can contain markdown links where the URL slot holds a **workbook UUID**, not a real URL:

```
[Sales Activity](0789c21e-86f5-4f7e-829a-7da0f5e18199)
```

These are entity references, not hyperlinks. If you render markdown, intercept them: render the label as plain text (or as a chip), and **don't** wrap the UUID in an `<a href>` — clicking would 404.

### Sample wire trace

```
:heartbeat

{"content":15892,"response_type":"message_id","metadata":null}

:keep-alive

{"content":{"action":"sales-activity_get_table_schema","input":{},"tool_call_id":"call_mL6Y..."},"response_type":"tool_call","metadata":null}

:keep-alive

{"content":{"tool_call_id":"call_mL6Y...","tool_response":[{"type":"text","text":"{\"id\":\"0789c21e-...\",\"tableName\":\"...\",\"columns\":[...]}"}]},"response_type":"subagent_task_end","metadata":null}

:keep-alive

{"content":"- **Sales Activity** is a workbook table titled ","response_type":"text","metadata":null}

{"content":"**\"Sales Activity\"**","response_type":"text","metadata":null}
…
```

Note in the trace above:
- Payload lines have no `data:` prefix — the parser must not require one.
- `tool_call` precedes `subagent_task_end` for the same `tool_call_id`.
- Text chunks can be very small (a phrase, a few words) — concatenate, don't render each as its own paragraph.

### Rule: never mint conversationId client-side

A client-generated UUID will produce a thread the server has no record of.
Always start with the POST to `/conversations`.

> **In an existing user codebase?** If the sandbox was opened via "Open repo from GitHub" (project config has `source: 'github'`, or there's a `.git/` directory and a non-template `package.json`), read the **`datagol-integrate`** skill first. It tells you how to graft this scaffold onto the user's existing app — framework detection, mount point, styling convention, what NOT to touch. The shapes documented here still apply; the wiring changes.

## When to Use

- After (or alongside) `datagol_create_agent` — you have an `agentId` to wire in.
- The user wants a chat surface for end users, not a CRUD editor.

If the user wants a CRUD app instead, use `datagol-ui-generation`.

## Tech Stack

- React 18 + Vite + TypeScript (the sandbox is already configured for this).
- Inline styles only (no Tailwind config).
- Native `fetch` with `ReadableStream` for SSE — **do not** add `eventsource-parser` or any other dependency.
- `useState` + `useEffect` only; no react-router; no state library.

## File Structure

```
src/
├── App.tsx              — chat page (single-page app)
├── lib/
│   ├── agent.ts         — sendMessage(), SSE parser
│   └── config.ts        — agentId, base URL, token
└── components/
    ├── MessageList.tsx
    ├── MessageBubble.tsx
    └── Composer.tsx     — textarea + send button
```

## Config Pattern

Hardcode the real `agentId` returned by `datagol_create_agent`.

### Token: use the service token — no postMessage bridge needed

**Always use `DATAGOL_SERVICE_TOKEN` for agent API calls.** The service token is already in `src/config.ts` (from `.env.local`). No `window.postMessage`, no `localStorage` bridge, no user JWT required.

The generated app uses `DATAGOL_SERVICE_TOKEN` directly from `src/config.ts` — the same token used for all workbook API calls. No postMessage bridge, no localStorage token, no user JWT.

```typescript
// src/lib/agent.ts  (config section)
import { DATAGOL_BASE_URL, DATAGOL_AI_URL, DATAGOL_SERVICE_TOKEN } from './config';

export const AGENT_ID = '024c4a46-2d93-4330-8bda-5bb560ffea72'; // replace with real id

// Conversation creation hits the main backend.
export const CONV_API   = `${DATAGOL_BASE_URL}/ai/api/v2/conversations`;
// Streaming hits a *separate* host (prod: ai.datagol.ai, test:
// testing-core-ai.datagol.ai). DATAGOL_AI_URL is defined in
// src/config.ts (see datagol-app-auth). Don't try to derive it from
// DATAGOL_BASE_URL — the test/prod hosts don't share a naming pattern.
export const STREAM_API = `${DATAGOL_AI_URL}/ai/api/v2/messages/streaming`;

// All agent API calls: x-auth-token with service token.
// Works because the agent was created with x-auth-token (service account is owner).
export function headers() {
  return { 'x-auth-token': DATAGOL_SERVICE_TOKEN, 'Content-Type': 'application/json' };
}
```

## Streaming Client

```typescript
// src/lib/agent.ts
import { STREAM_API, CONV_API, AGENT_ID, DATAGOL_SERVICE_TOKEN } from './config'

export type StreamEvent =
  | { kind: 'delta'; text: string }
  | { kind: 'html'; html: string }
  | { kind: 'tool_start'; toolCallId: string; action: string; input: unknown }
  | { kind: 'tool_end'; toolCallId: string; result: unknown }  // result is the *parsed* payload
  | { kind: 'done' }
  | { kind: 'error'; message: string }

// All agent API calls use x-auth-token with the service token.
function headers() {
  return { 'x-auth-token': DATAGOL_SERVICE_TOKEN, 'Content-Type': 'application/json' };
}

/**
 * Fetch and cache the numeric userId for the service account.
 * Required by the conversation creation endpoint.
 */
let _userId: number | null = null;
async function getUserId(): Promise<number> {
  if (_userId !== null) return _userId;
  const res = await fetch(`${DATAGOL_BASE_URL}/idp/api/v1/user`, { headers: headers() });
  if (!res.ok) throw new Error(`Failed to fetch user: HTTP ${res.status}`);
  const data = (await res.json()) as { id?: number };
  if (!data.id) throw new Error('id missing from /idp/api/v1/user response');
  _userId = data.id;
  return _userId;
}

/**
 * Create a fresh conversation thread on the server. Returns the conversationId
 * to reuse across all messages in this thread.
 * The agent must have been created with the service token (x-auth-token) so
 * the service account is the owner — see datagol-create-agent skill.
 */
export async function createConversation(name: string): Promise<string> {
  const userId = await getUserId();
  const res = await fetch(CONV_API, {
    method: 'POST',
    headers: headers(),
    body: JSON.stringify({
      name: name.slice(0, 60) || 'New chat',
      active: true,
      scope: 'PRIVATE',
      userId,
      agentType: 'CustomAgent',
      uiMetadata: {
        type: 'CustomAgent',
        parameters: {
          customAgentId: AGENT_ID,
          parameters: { customAgentId: AGENT_ID },
        },
        configuration: { llmModel: 'claude-sonnet-4-6' },
      },
    }),
  });
  if (!res.ok) {
    const text = await res.text().catch(() => `HTTP ${res.status}`);
    throw new Error(`Failed to create conversation: ${text}`);
  }
  const conv = await res.json() as { id: string };
  return conv.id;
}

/**
 * Send `message` to the custom agent and yield streaming events. The
 * `conversationId` MUST come from createConversation() — never client-generated.
 */
export async function* sendMessage(
  message: string,
  conversationId: string,
  signal?: AbortSignal,
): AsyncGenerator<StreamEvent> {
  const res = await fetch(STREAM_API, {
    method: 'POST',
    headers: headers(),
    body: JSON.stringify({
      agentType: 'CustomAgent',
      type: 'CustomAgent',
      conversationId,
      message,
      parameters: { customAgentId: AGENT_ID },
      uiMetadata: { parameters: {} },
      selectedLlmModel: 'claude-sonnet-4-6',
    }),
    signal,
  })

  if (!res.ok) {
    const text = await res.text().catch(() => `HTTP ${res.status}`)
    yield { kind: 'error', message: text || `HTTP ${res.status}` }
    return
  }
  if (!res.body) {
    yield { kind: 'error', message: 'No response body' }
    return
  }

  const reader = res.body.getReader()
  const decoder = new TextDecoder()
  let buffer = ''

  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buffer += decoder.decode(value, { stream: true })

    // Records are separated by blank lines.
    const records = buffer.split('\n\n')
    buffer = records.pop() ?? ''

    for (const record of records) {
      // A record can have multiple lines (e.g. a comment + a payload line).
      // Pull out the first non-comment payload line. The DataGOL stream emits
      // payload lines as bare JSON without a `data:` prefix, but we strip the
      // prefix if present so the parser also handles standard SSE servers.
      let payload: string | null = null
      for (const rawLine of record.split('\n')) {
        const line = rawLine.trim()
        if (!line) continue
        if (line.startsWith(':')) continue        // comment / heartbeat
        payload = line.startsWith('data:') ? line.slice(5).trim() : line
        break
      }
      if (!payload) continue

      // Wire format (verified):
      //   { "content": "...",     "response_type": "text",                "metadata": null }
      //   { "content": 15892,     "response_type": "message_id",          "metadata": null }
      //   { "content": { action, input, tool_call_id },
      //                           "response_type": "tool_call",           "metadata": null }
      //   { "content": { tool_call_id, tool_response: [{ type, text }] },
      //                           "response_type": "subagent_task_end",   "metadata": null }
      let obj: { content?: unknown; response_type?: string }
      try { obj = JSON.parse(payload) }
      catch { continue }

      switch (obj.response_type) {
        case 'text':
          if (typeof obj.content === 'string' && obj.content.length > 0) {
            yield { kind: 'delta', text: obj.content }
          }
          break
        case 'html':
          if (typeof obj.content === 'string' && obj.content.length > 0) {
            yield { kind: 'html', html: obj.content }
          }
          break
        case 'tool_call': {
          const c = obj.content as { action?: string; input?: unknown; tool_call_id?: string } | undefined
          if (c?.tool_call_id && c?.action) {
            yield { kind: 'tool_start', toolCallId: c.tool_call_id, action: c.action, input: c.input }
          }
          break
        }
        case 'subagent_task_end': {
          const c = obj.content as {
            tool_call_id?: string
            tool_response?: Array<{ type?: string; text?: string }>
          } | undefined
          const id = c?.tool_call_id
          const text = c?.tool_response?.[0]?.text
          if (id && typeof text === 'string') {
            // tool_response[0].text is itself JSON-encoded; parse it again.
            // If parsing fails, surface the raw string so the UI can still display it.
            let result: unknown
            try { result = JSON.parse(text) }
            catch { result = text }
            yield { kind: 'tool_end', toolCallId: id, result }
          }
          break
        }
        case 'message_id':
          // Server-assigned message id. Currently unused.
          break
        default:
          // Unknown response_type — ignore safely (forward-compat).
          break
      }
    }
  }
  // No explicit `done` event — completion is signalled by HTTP stream end.
  yield { kind: 'done' }
}
```

The parser only ever yields `delta` events for chunks of text; the `done`
signal comes from the stream ending, not an in-band event.

## App.tsx Pattern

```tsx
import { useState, useRef, useEffect } from 'react'
import { sendMessage, createConversation } from './lib/agent'

type ToolActivity = { toolCallId: string; action: string; done: boolean }
type Part =
  | { kind: 'text'; text: string }   // accumulated `delta` chunks
  | { kind: 'html'; html: string }   // one complete `html` event (a chart, etc.)
type Msg = {
  id: string
  role: 'user' | 'assistant'
  content: string                    // user messages only
  parts?: Part[]                     // assistant messages: text/html in order
  streaming?: boolean
  tools?: ToolActivity[]             // tool calls observed during this turn
}

/* ── Minimal Markdown renderer ─────────────────────────────────────────────────
 * Handles the elements the `datagol-create-agent` skill instructs the agent to emit:
 *   ## / ### headers, bullet lists, **bold**, `code`, ```fenced```, tables,
 *   and [label](https://…) links. Non-http link targets (e.g. workbook UUIDs
 *   from the agent's stream) render as plain text — never as <a href>.
 * No external dependency. ~70 lines.
 */
function Md({ text }: { text: string }) {
  const lines = text.split('\n')
  const out: React.ReactNode[] = []
  let i = 0
  while (i < lines.length) {
    const ln = lines[i]!

    // Fenced code block
    if (ln.startsWith('```')) {
      const code: string[] = []
      i++
      while (i < lines.length && !lines[i]!.startsWith('```')) { code.push(lines[i]!); i++ }
      out.push(
        // Wrap long code lines instead of side-scrolling — chat rule 11.
        <pre key={i} style={{
          background: '#f3f3f0', padding: '8px 10px', borderRadius: 6,
          fontSize: 12, margin: '6px 0', maxWidth: '100%',
          whiteSpace: 'pre-wrap', wordBreak: 'break-word',
        }}>
          <code>{code.join('\n')}</code>
        </pre>,
      )
      i++; continue
    }

    // Pipe-delimited table
    if (ln.includes('|') && ln.trim().startsWith('|')) {
      const rows: string[][] = []
      while (i < lines.length && lines[i]!.includes('|') && lines[i]!.trim().startsWith('|')) {
        if (!/^\|[-:\s|]+\|$/.test(lines[i]!.trim())) {
          rows.push(lines[i]!.split('|').slice(1, -1).map(c => c.trim()))
        }
        i++
      }
      if (rows.length > 0) out.push(
        // table-layout: fixed forces columns to share the bubble's width
        // evenly; word-break on cells lets long values wrap. No scrollbar.
        <table key={i} style={{
          borderCollapse: 'collapse', tableLayout: 'fixed',
          width: '100%', maxWidth: '100%',
          fontSize: 13, margin: '6px 0',
        }}>
          <thead><tr>{rows[0]!.map((h, ci) => (
            <th key={ci} style={{ textAlign: 'left', padding: '6px 8px', borderBottom: '1px solid #e4e4e0', fontWeight: 600, wordBreak: 'break-word' }}>{h}</th>
          ))}</tr></thead>
          <tbody>{rows.slice(1).map((row, ri) => (
            <tr key={ri}>{row.map((cell, ci) => (
              <td key={ci} style={{ padding: '6px 8px', borderBottom: '1px solid #f3f3f0', wordBreak: 'break-word' }}>{fmt(cell)}</td>
            ))}</tr>
          ))}</tbody>
        </table>,
      )
      continue
    }

    if (ln.startsWith('### '))      out.push(<p key={i} style={{ fontSize: 14, fontWeight: 600, margin: '8px 0 2px' }}>{fmt(ln.slice(4))}</p>)
    else if (ln.startsWith('## '))  out.push(<p key={i} style={{ fontSize: 15, fontWeight: 700, margin: '10px 0 2px' }}>{fmt(ln.slice(3))}</p>)
    else if (ln.startsWith('- '))   out.push(<div key={i} style={{ display: 'flex', gap: 6, marginLeft: 6 }}><span>•</span><span>{fmt(ln.slice(2))}</span></div>)
    else if (ln === '')             out.push(<div key={i} style={{ height: 4 }} />)
    else                            out.push(<p key={i} style={{ margin: '2px 0', lineHeight: 1.55 }}>{fmt(ln)}</p>)
    i++
  }
  return <div style={{ fontSize: 14 }}>{out}</div>
}

/* Sandboxed HTML renderer. Wraps the agent's html in a srcdoc iframe so its
 * <style>/<script> can't reach the parent page. Auto-resizes to content
 * height via a postMessage from the iframe back to the parent. */
function HtmlContent({ html }: { html: string }) {
  const ref = useRef<HTMLIFrameElement | null>(null)
  const [height, setHeight] = useState(80)

  useEffect(() => {
    function onMsg(e: MessageEvent) {
      if (e.source !== ref.current?.contentWindow) return
      const d = e.data as { type?: string; height?: number } | null
      if (d?.type === 'datagol_html_height' && typeof d.height === 'number') {
        setHeight(Math.max(40, Math.ceil(d.height) + 8))
      }
    }
    window.addEventListener('message', onMsg)
    return () => window.removeEventListener('message', onMsg)
  }, [])

  // Inject CSS to keep charts fluid: SVGs scale down to the bubble's width,
  // images cap at 100%, and overflow is hidden so the iframe never grows
  // its own scrollbars (chat rule 11). The resize-reporter script reports
  // body height so the parent can size the iframe to fit content.
  const srcdoc = `<!doctype html>
<html><head><meta charset="utf-8">
<style>
  html, body { margin: 0; padding: 0; overflow: hidden; max-width: 100%;
               font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
  body { width: 100%; box-sizing: border-box; }
  body * { max-width: 100%; box-sizing: border-box; }
  svg, img, canvas, video { max-width: 100%; height: auto; display: block; }
</style>
</head><body>${html}
<script>
  function post(){var h=Math.max(document.documentElement.scrollHeight,document.body.scrollHeight);window.parent.postMessage({type:'datagol_html_height',height:h},'*');}
  window.addEventListener('load',post);
  if (window.ResizeObserver) new ResizeObserver(post).observe(document.body);
</script>
</body></html>`

  return (
    <iframe
      ref={ref}
      srcDoc={srcdoc}
      sandbox="allow-scripts"
      style={{ width: '100%', height: height + 'px', border: 'none', background: 'transparent' }}
      title="Agent visualization"
    />
  )
}

/* Streaming indicator. Keep one visible whenever the assistant is mid-stream
 * — the user should never wonder whether something is still happening. */
function Thinking() {
  return (
    <div style={{
      display: 'inline-flex', alignItems: 'center', gap: 8,
      fontSize: 12, color: '#6b6b68', fontStyle: 'italic',
      padding: '4px 0',
    }}>
      <span style={{ display: 'inline-flex', gap: 4 }}>
        <span style={{ width: 6, height: 6, borderRadius: '50%', background: '#9a9a98', animation: 'datagolDot 1.2s infinite', animationDelay: '0s' }} />
        <span style={{ width: 6, height: 6, borderRadius: '50%', background: '#9a9a98', animation: 'datagolDot 1.2s infinite', animationDelay: '0.2s' }} />
        <span style={{ width: 6, height: 6, borderRadius: '50%', background: '#9a9a98', animation: 'datagolDot 1.2s infinite', animationDelay: '0.4s' }} />
      </span>
      <span>thinking</span>
    </div>
  )
}

/* Inject the keyframes once. Lives at module scope so it doesn't reattach on
 * every render. Vite/CSS-in-JS would also work; this keeps the scaffold
 * dependency-free. */
if (typeof document !== 'undefined' && !document.getElementById('datagol-keyframes')) {
  const s = document.createElement('style')
  s.id = 'datagol-keyframes'
  s.textContent = '@keyframes datagolDot { 0%, 80%, 100% { opacity: 0.3; transform: translateY(0) } 40% { opacity: 1; transform: translateY(-2px) } }'
  document.head.appendChild(s)
}

function fmt(text: string): React.ReactNode {
  const parts = text.split(/(\*\*[^*]+\*\*|`[^`]+`|\[[^\]]+\]\([^)]+\))/g)
  return parts.map((p, i) => {
    if (p.startsWith('**') && p.endsWith('**')) return <strong key={i} style={{ fontWeight: 600 }}>{p.slice(2, -2)}</strong>
    if (p.startsWith('`') && p.endsWith('`'))   return <code key={i} style={{ padding: '1px 4px', borderRadius: 3, background: '#f3f3f0', fontFamily: 'ui-monospace, monospace', fontSize: 12 }}>{p.slice(1, -1)}</code>
    const link = p.match(/^\[([^\]]+)\]\(([^)]+)\)$/)
    if (link) {
      const [, label, href] = link
      // Only http(s) targets become real links; UUID-style refs from the
      // agent stream render as plain text (per the wire-format docs).
      if (/^https?:\/\//i.test(href ?? '')) {
        return <a key={i} href={href} target="_blank" rel="noopener noreferrer" style={{ color: '#2563eb', textDecoration: 'underline' }}>{label}</a>
      }
      return <span key={i}>{label}</span>
    }
    return p
  })
}

export default function App() {
  const [messages, setMessages] = useState<Msg[]>([])
  const [input, setInput] = useState('')
  const [busy, setBusy] = useState(false)
  // No token state needed — DATAGOL_SERVICE_TOKEN is used directly in headers().
  // conversationId is null until the first send. createConversation() is called
  // lazily on the first user message; the resulting id is reused for the rest
  // of the thread until the user clicks "New chat".
  const conversationId = useRef<string | null>(null)
  const abortRef = useRef<AbortController | null>(null)

  async function send() {
    const text = input.trim()
    if (!text || busy) return
    // No token check needed — DATAGOL_SERVICE_TOKEN is always available from config.
    setInput('')
    const userMsgId = crypto.randomUUID()
    const asstId = crypto.randomUUID()
    setMessages(m => [...m,
      { id: userMsgId, role: 'user', content: text },
      { id: asstId, role: 'assistant', content: '', parts: [], streaming: true },
    ])
    setBusy(true)
    abortRef.current = new AbortController()
    try {
      // Lazy-create the conversation on the first message of a thread.
      if (!conversationId.current) {
        conversationId.current = await createConversation(text)
      }
      for await (const ev of sendMessage(text, conversationId.current, abortRef.current.signal)) {
        if (ev.kind === 'delta') {
          // Append to the last text part, or start a new one if the prior part
          // was something else (html / nothing). Keeps text and html interleaved
          // in the order they arrived.
          setMessages(m => m.map(x => {
            if (x.id !== asstId) return x
            const parts = [...(x.parts ?? [])]
            const last = parts[parts.length - 1]
            if (last?.kind === 'text') parts[parts.length - 1] = { kind: 'text', text: last.text + ev.text }
            else parts.push({ kind: 'text', text: ev.text })
            return { ...x, parts }
          }))
        } else if (ev.kind === 'html') {
          setMessages(m => m.map(x => x.id !== asstId ? x : {
            ...x,
            parts: [...(x.parts ?? []), { kind: 'html', html: ev.html }],
          }))
        } else if (ev.kind === 'tool_start') {
          setMessages(m => m.map(x => x.id !== asstId ? x : {
            ...x,
            tools: [...(x.tools ?? []), { toolCallId: ev.toolCallId, action: ev.action, done: false }],
          }))
        } else if (ev.kind === 'tool_end') {
          setMessages(m => m.map(x => x.id !== asstId ? x : {
            ...x,
            tools: (x.tools ?? []).map(t => t.toolCallId === ev.toolCallId ? { ...t, done: true } : t),
          }))
        } else if (ev.kind === 'error') {
          setMessages(m => m.map(x => x.id !== asstId ? x : {
            ...x,
            parts: [...(x.parts ?? []), { kind: 'text', text: 'Error: ' + ev.message }],
            streaming: false,
          }))
        } else if (ev.kind === 'done') {
          setMessages(m => m.map(x => x.id === asstId ? { ...x, streaming: false } : x))
        }
      }
    } catch (e) {
      const msg = e instanceof Error ? e.message : String(e)
      setMessages(m => m.map(x => x.id !== asstId ? x : {
        ...x,
        parts: [...(x.parts ?? []), { kind: 'text', text: 'Error: ' + msg }],
        streaming: false,
      }))
    } finally {
      setBusy(false)
      abortRef.current = null
    }
  }

  function newConversation() {
    abortRef.current?.abort()
    conversationId.current = null  // next send will mint a fresh server thread
    setMessages([])
  }

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh',
                  fontFamily: 'system-ui, -apple-system, "Segoe UI", sans-serif',
                  background: '#f8f8f6', color: '#1a1a18' }}>
      {/* Header */}
      <header style={{ display: 'flex', gap: 8, alignItems: 'center',
                       padding: '12px 20px', borderBottom: '1px solid #e4e4e0', background: '#fff' }}>
        <strong style={{ flex: 1 }}>Agent Chat</strong>

        <button onClick={newConversation}
                style={{ padding: '6px 12px', border: '1px solid #e4e4e0', borderRadius: 6,
                         background: '#fff', cursor: 'pointer', fontSize: 13 }}>New chat</button>
      </header>

      {/* Messages */}
      <main style={{ flex: 1, overflowY: 'auto', padding: 24,
                     display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 760,
                     margin: '0 auto', width: '100%' }}>
        {messages.length === 0 && (
          <p style={{ color: '#9a9a98', textAlign: 'center', marginTop: 80 }}>
            Ask the agent anything…
          </p>
        )}
        {messages.map(m => {
          const isUser = m.role === 'user'
          return (
            <div key={m.id} style={{
              alignSelf: isUser ? 'flex-end' : 'flex-start',
              background: isUser ? '#2563eb' : '#fff',
              color: isUser ? '#fff' : '#1a1a18',
              border: isUser ? 'none' : '1px solid #e4e4e0',
              borderRadius: 12, padding: '10px 14px', maxWidth: '80%',
              fontSize: 14, lineHeight: 1.5,
              display: 'flex', flexDirection: 'column', gap: 6,
              // overflow-wrap: anywhere lets long URLs/IDs/tokens wrap
              // mid-string instead of pushing the bubble past maxWidth.
              // min-width: 0 prevents flex-children from inheriting their
              // content's intrinsic width and silently defeating wrap.
              overflowWrap: 'anywhere',
              minWidth: 0,
            }}>
              {(m.tools ?? []).map(t => (
                <div key={t.toolCallId} style={{
                  fontSize: 12, color: '#6b6b68', fontFamily: 'ui-monospace, monospace',
                  display: 'inline-flex', alignItems: 'center', gap: 6,
                  padding: '2px 8px', borderRadius: 6, background: '#f3f3f0',
                }}>
                  <span>{t.done ? '✓' : '⋯'}</span>
                  <span>{t.action}</span>
                </div>
              ))}
              {/* User messages stay literal. Assistant content is a list of
                  parts — text rendered as Markdown, html rendered in a
                  sandboxed iframe — kept in arrival order so a chart can land
                  before or after the surrounding prose. */}
              {isUser ? (
                <span style={{ whiteSpace: 'pre-wrap' }}>{m.content}</span>
              ) : (
                <>
                  {(m.parts ?? []).map((p, i) => p.kind === 'text'
                    ? <Md key={i} text={p.text} />
                    : <HtmlContent key={i} html={p.html} />
                  )}
                  {/* `Thinking` is always visible while the stream is live —
                      the user should see motion even when the agent is between
                      tool calls or pausing mid-response. It's hidden only once
                      `agent_done` flips `streaming` to false. */}
                  {m.streaming && <Thinking />}
                </>
              )}
            </div>
          )
        })}
      </main>

      {/* Composer */}
      <footer style={{ padding: 16, borderTop: '1px solid #e4e4e0', background: '#fff' }}>
        <div style={{ display: 'flex', gap: 8, maxWidth: 760, margin: '0 auto' }}>
          <textarea value={input} onChange={e => setInput(e.target.value)}
            onKeyDown={e => { if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); send() } }}
            placeholder="Type a message — Enter to send"
            rows={1}
            style={{ flex: 1, padding: '10px 12px', border: '1px solid #e4e4e0',
                     borderRadius: 8, fontSize: 14, resize: 'none', fontFamily: 'inherit',
                     outline: 'none' }} />
          <button onClick={send} disabled={busy || !input.trim()}
            style={{ padding: '10px 18px', border: 'none', borderRadius: 8,
                     background: busy ? '#9a9a98' : '#2563eb', color: '#fff',
                     fontSize: 14, fontWeight: 600,
                     cursor: busy ? 'default' : 'pointer' }}>
            {busy ? '…' : 'Send'}
          </button>
        </div>
      </footer>
    </div>
  )
}
```

## Quality Rules

1. **Hardcode the real `agentId`** in `src/lib/config.ts` — get it from `datagol_create_agent`'s response. Don't ask the user to paste it.
2. **Use `DATAGOL_SERVICE_TOKEN` — no user token needed.** The service token from `src/config.ts` is used for all agent API calls via `x-auth-token`. No postMessage bridge, no localStorage token, no user JWT. This works because the agent is created with the service token (service account is the owner).
3. **Always create the conversation server-side first.** Call `createConversation()` lazily on the first message of a thread; reuse the returned `id` for the rest of the thread; null it out and let the next send mint a new one when the user clicks "New chat". **Never** generate a conversationId client-side.
4. **No `userId`, no `/me`, no JWT decode.** The DataGOL backend derives the identity from the `x-auth-token`. Do not add any user identity fields to request bodies.
5. **Append deltas to the last assistant message** — don't replace.
6. **Always show a streaming indicator** (e.g. `…` placeholder) until the stream ends.
7. **Provide an abort path** — at minimum a "New chat" button that aborts and resets.
8. **No external streaming libs.** The native fetch + ReadableStream snippet above is the canonical pattern.
9. **Make the chat window resizable.** Add an expand/contract toggle in the header (icon next to "New chat") that toggles a wider viewport — useful for reading long agent responses, tables, or rendered HTML charts. Two valid implementations:

   - **Standalone chat:** lift `maxWidth` on `<main>` and `<footer>` from a constrained value (e.g. `760px`) up to `1200px` (or the viewport width minus padding). Keep state in `App` as `const [wide, setWide] = useState(false)`; flip via the toggle.
   - **Embedded chat (floating-button + slide-in panel):** apply `position: fixed` to the chat wrapper. Default state pins it to `bottom: 24, right: 24, width: 620, height: 700`. Expanded state stretches to `top: 24, left: 24, right: 24, bottom: 24, width: auto, height: auto`. Render a sibling backdrop with `position: fixed; inset: 0; background: rgba(0,0,0,0.4); z-index: <wrapper z-index − 1>` so dimming flows behind the wrapper, not over the iframe contents.

   In **both** cases the chat root component (and any `HtmlContent` iframes inside) must stay **mounted across the toggle** — apply CSS only, don't conditionally render different trees. Streaming responses, the SSE reader, and any partially-arrived `parts[]` are lost on remount. Bind `Esc` to collapse only while expanded.

10. **Never put `\uXXXX` escape sequences as JSX text content.** JSX text between tags is rendered verbatim — `<span>•</span>` displays the literal seven-character string `•`, not a bullet. Either:
    - **Paste the actual character** in source: `<span>•</span>`, `<span>✓</span>`, `<span>×</span>`, `<span>›</span>`. The scaffold's source files are UTF-8; this is the preferred form.
    - Or wrap in a JS expression: `<span>{"•"}</span>` — the escape is interpreted because it's now a JS string literal, not JSX text.

    Same trap with `&#8226;` HTML entities — those work in static HTML but not in JSX text. Use the literal character. Common offenders: `•` `›` `→` `✓` `✗` `…` `—`.

11. **Nothing in a chat bubble overflows or scrolls.** Text, code, tables, and embedded charts must fit inside the bubble's `maxWidth: 80%` constraint — no horizontal scrollbars, no clipped content. Concretely:

    - **Text containers:** `overflowWrap: 'anywhere'` (or `wordBreak: 'break-word'`) so unbroken strings (URLs, IDs, tokens) wrap mid-string.
    - **Flex parent:** `minWidth: 0` on the bubble — flex children default to `minWidth: auto` (their content's intrinsic width), which silently defeats `overflow-wrap`.
    - **Code blocks (`<pre>`):** use `whiteSpace: 'pre-wrap'` + `wordBreak: 'break-word'` so wide code wraps instead of side-scrolling. Do **not** use `overflow-x: auto` on `<pre>` — that adds a scrollbar.
    - **Markdown tables:** `tableLayout: 'fixed'`, `width: '100%'`, and `wordBreak: 'break-word'` on cells. Fixed layout forces columns to share the bubble's width evenly; break-word lets long cell values wrap.
    - **HTML/chart embeds (`HtmlContent` iframe):** the iframe must `width: 100%` of the bubble. The srcdoc CSS must hide overflow (`html, body { overflow: hidden }`) and make SVG fluid (`svg { max-width: 100%; height: auto; display: block }`). The auto-resize script reports `scrollHeight`, but the iframe **must not show its own scrollbars** — if a chart can't fit at the bubble's width, the chart's `viewBox` should preserve aspect ratio and the SVG scales down. Never inject horizontal scroll inside the iframe.

    Test with: a 120-char URL, a 200-char code line, a 6-column table, and a chart whose source uses a fixed `width="880"` — none should produce a scrollbar; all should wrap or scale to fit.

## File Writing Order

1. `src/lib/config.ts`
2. `src/lib/agent.ts`
3. `src/App.tsx`

After writing, tell the user:
- The agent ID is wired in.
- The DataGOL token is auto-loaded from `?token=` (URL param injected by the
  Codex preview iframe) and reused from `localStorage` on subsequent loads —
  no manual paste required.
- Click **New chat** to reset the conversation thread.
