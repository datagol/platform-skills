---
name: datagol-slack-connector
description: Slack-specific connector implementation for DataGOL-generated apps — channels and messages. Triggered by "build a slack connector", "connect slack", "sync my slack", "build a slack integration", "index slack messages". Layers on top of `datagol-connector` (parent), supplying the Slack Messages + Slack Channels workbook schemas, OAuth scopes (channels:read, channels:history, groups:read, groups:history, users:read, team:read), backfill logic (conversations.list + conversations.history with oldest=now-30d), polling logic (per-channel oldest=last_ts), and the Slack Web API client used in the generated app. DMs are out of scope in v1.
---

# DataGOL Slack Connector — Channels + Messages

This skill is a child of **`datagol-connector`**. It supplies *only* the Slack-specific bits: scopes, service path, row schemas, backfill queries, polling fan-out, response-envelope handling, rate-limit retry. The shared mechanics (OAuth handoff, `Integration Connectors` workbook, token broker, polling-loop scaffold, hard rules around persistence and dedup) all live in the parent.

> **Read `datagol-connector` and `datagol-app-auth` first.** This skill assumes you've already done the pre-flight checklist (workspace, service account, `Integration Connectors` workbook, Connections page shell) from those skills. Don't duplicate their setup steps here.

## When to use

- The user says "build a slack connector", "connect slack", "sync my slack", "index slack messages", "pull our slack into a workbook", "build a slack integration".
- The user wants to surface Slack data inside DataGOL — search across channel history, dashboards over message volume, agents that answer questions about discussions, etc. (Those downstream features are scaffolded by other skills like `datagol-create-dashboard` and `datagol-create-agent`; this skill provides the *data*.)
- The user is OK with browser-only polling (page must be open). For always-on indexing, surface that constraint and offer to flag it as a follow-up — there is no backend scheduler today.

## OAuth scopes

Default to **bot-token, read-only**. Configure on the Slack app's **OAuth & Permissions** page (Slack-side; not the generated app's concern):

| Scope                | Why                                                             |
|----------------------|-----------------------------------------------------------------|
| `channels:read`      | List public channels the bot is in (`conversations.list`)       |
| `channels:history`   | Read messages in public channels (`conversations.history`)      |
| `groups:read`        | List private channels the bot is invited to                     |
| `groups:history`     | Read messages in those private channels                         |
| `users:read`         | Resolve `user_id` → display name in a future skill              |
| `team:read`          | Capture `team_id` for multi-workspace disambiguation            |

**Never request `im:*` or `mpim:*` scopes in v1.** DMs are deliberately out of scope so the bot's footprint is predictable and the user knows exactly what they consented to. Adding DMs later is a separate skill iteration.

If the user asks to send messages, post replies, or modify channel state, ask before requesting write scopes (`chat:write`, `chat:write.public`, etc.).

## Service path & OAuth start URL

| Service                 | `<servicePath>`  |
|-------------------------|------------------|
| Slack (channels + msgs) | `slack`          |

Full OAuth start URL (assembled by `src/api/connectors.ts → startOAuthUrl('slack')`):

```
GET ${DATAGOL_BASE_URL}/idp/api/v1/oauth2/slack/authorize
    ?x-auth-token=<encodeURIComponent(DATAGOL_SERVICE_TOKEN)>
    &sourceType=connector
    &clientRedirectUri=<encodeURIComponent(window.location.href)>
```

Verified shape against the backend (test env). Three things that bite if you get them wrong:

- The query param is **`x-auth-token`** — not `jwtToken`, not `Authorization`. Same name as the header used elsewhere.
- The token is the **service token**, never a user JWT. (See parent skill — using a user JWT here orphans the connector and the broker returns `REAUTHENTICATION_REQUIRED`.)
- `clientRedirectUri` is the generated app's URL, including the trailing slash if any. URL-encode it (`encodeURIComponent`).

## Workbook schemas

Two data workbooks. Both reference `connector_id` so rows link back to `Integration Connectors` — useful when multiple Slack workspaces are connected.

### `Slack Messages` — one row per message (top-level *and* thread replies)

| internal name      | type      | notes                                                          |
|--------------------|-----------|----------------------------------------------------------------|
| `connector_id`     | LONG_TEXT | FK to `Integration Connectors.connector_id`                    |
| `team_id`          | LONG_TEXT | Slack workspace id (`T0123…`)                                  |
| `channel_id`       | LONG_TEXT | Slack channel id (`C0123…`)                                    |
| `channel_name`     | LONG_TEXT | denormalized for cheap reads; refreshed on channel-list cycle  |
| `message_ts`       | LONG_TEXT | unique within channel; format `1234567890.123456`. **Always TEXT — never NUMBER.** |
| `thread_ts`        | LONG_TEXT | nullable; equal to `message_ts` for top-level, ≠ for replies   |
| `is_thread_reply`  | BOOLEAN   | true when `thread_ts` is present and ≠ `message_ts`            |
| `user_id`          | LONG_TEXT | Slack user id (`U0123…`); resolve to name in a future skill    |
| `text`             | LONG_TEXT | message text, raw markdown                                     |
| `blocks_json`      | LONG_TEXT | JSON-encoded Slack blocks (rich layout); nullable              |
| `attachments_json` | LONG_TEXT | JSON-encoded legacy attachments; nullable                      |
| `files_json`       | LONG_TEXT | JSON-encoded file objects on the message; nullable             |
| `reactions_json`   | LONG_TEXT | JSON-encoded reactions array; nullable                         |
| `subtype`          | LONG_TEXT | nullable; e.g. `bot_message`, `channel_join`, `me_message`     |
| `is_edited`        | BOOLEAN   | true if `edited` field present                                 |
| `posted_at`        | DATE      | ISO 8601 with tz, derived from `ts`                            |

Logical primary key: `(connector_id, channel_id, message_ts)`. Dedup query before each bulk-insert:

```ts
// example whereClause for dedup
"`channel_id` = 'C0123ABC' AND `message_ts` IN ('1714872345.001100','1714872400.002000', …)"
```

### `Slack Channels` — one row per channel the bot can see

| internal name        | type      | notes                                                  |
|----------------------|-----------|--------------------------------------------------------|
| `connector_id`       | LONG_TEXT | FK                                                     |
| `team_id`            | LONG_TEXT |                                                        |
| `channel_id`         | LONG_TEXT | unique within team                                     |
| `name`               | LONG_TEXT |                                                        |
| `is_private`         | BOOLEAN   |                                                        |
| `is_archived`        | BOOLEAN   | indexed even when archived; flagged so UIs can filter  |
| `is_general`         | BOOLEAN   | the workspace's default channel                        |
| `num_members`        | NUMBER    |                                                        |
| `topic`              | LONG_TEXT | current topic text                                     |
| `purpose`            | LONG_TEXT | current purpose text                                   |
| `created_at`         | DATE      | ISO 8601 derived from `created` (unix sec)             |
| `last_synced_at`     | DATE      | when this channel's metadata was last refreshed        |

Updates use `PATCH` keyed on `(connector_id, channel_id)` (read row by `whereClause`, then PATCH by id). New channels bulk-insert. **Only PATCH when something meaningful actually changed** — see `backfillSlackChannels` below.

## Cursor encoding (`Integration Connectors.cursor_json` for Slack)

```json
{
  "channel_ts": {
    "C0123ABC": "1714872345.001100",
    "C0456DEF": "1714873012.002300"
  },
  "channels_fetched_at": "2026-05-05T18:42:00.000Z"
}
```

- `channel_ts[<channelId>]` = newest `ts` already inserted for that channel. The `oldest=` param on the next `conversations.history` call is set to this value.
- `channels_fetched_at` = last time `conversations.list` ran. On every poll cycle, refresh the channel list if this is older than **15 minutes**. New channels appear in the next cycle automatically.

## Initial backfill — Channels

> **Upsert rule (learned from production):** A naïve implementation that PATCHes every channel on every pull generates one PATCH request per channel even when nothing changed — 50 channels = 50 unnecessary writes before a single message is fetched. The correct pattern is:
> 1. `bulkInsert` all *new* channels in one request.
> 2. `PATCH` an existing channel **only when** `name`, `num_members`, `topic`, `purpose`, or `is_archived` differs from the stored value.
> 3. Skip channels whose stored metadata is identical — no request at all.

```ts
// src/api/slack.ts (excerpt)
// SLACK_API must be the proxy path — never https://slack.com/api (CORS).

export async function backfillSlackChannels(
  connectorId: string,
  accessToken: string,
): Promise<any[]> {
  const all: any[] = [];
  let cursor: string | undefined;

  do {
    const qs = new URLSearchParams({
      types: 'public_channel,private_channel',
      limit: '200',
      ...(cursor ? { cursor } : {}),
    });
    const data = await slackCall(`conversations.list?${qs}`, accessToken);
    all.push(...(data.channels ?? []));
    cursor = data.response_metadata?.next_cursor || undefined;
  } while (cursor);

  const teamId = all[0]?.shared_team_ids?.[0] ?? all[0]?.context_team_id ?? '';
  const now = new Date().toISOString();

  // Read existing channel rows once — keyed by Slack channel_id.
  const existingChannels = await listRows('Slack Channels', {
    whereClause: `\`connector_id\` = '${connectorId}'`,
    pageSize: 500,
  });
  const existingById = new Map<string, any>(
    existingChannels.map((r: any) => [r.channel_id, r]),
  );

  const toInsert: Record<string, unknown>[] = [];

  for (const c of all) {
    const row = {
      connector_id: connectorId,
      team_id: c.shared_team_ids?.[0] ?? teamId,
      channel_id: c.id,
      name: c.name ?? '',
      is_private: !!c.is_private,
      is_archived: !!c.is_archived,
      is_general: !!c.is_general,
      num_members: c.num_members ?? 0,
      topic: c.topic?.value ?? '',
      purpose: c.purpose?.value ?? '',
      created_at: new Date((c.created ?? 0) * 1000).toISOString(),
      last_synced_at: now,
    };

    const prior = existingById.get(c.id);
    if (!prior) {
      // New channel — batch for bulk insert
      toInsert.push(row);
    } else {
      // Only PATCH if something meaningful changed
      const changed =
        prior.name !== row.name ||
        prior.num_members !== row.num_members ||
        prior.topic !== row.topic ||
        prior.purpose !== row.purpose ||
        prior.is_archived !== row.is_archived;
      if (changed) {
        await updateRow('Slack Channels', prior.id, row);  // PATCH by DataGOL row id
      }
      // unchanged → no request
    }
  }

  // One bulk-insert for all new channels
  if (toInsert.length > 0) {
    await bulkInsertRows('Slack Channels', toInsert);
  }

  return all;
}
```

`slackCall` is the wrapper from "Slack API client" below — it handles the response-envelope quirk and routes through the `/slack-api` Vite proxy.

## Initial backfill — Messages (last 30 days)

> **Rate-limit safety (learned from production):** On the first pull, cap total messages to **100** across all channels. Slack’s `conversations.history` is Tier 3 (~50 calls/min). A workspace with many busy channels exhausts that budget instantly if you paginate without a global cap. 100 is safe for any workspace size and gives the user a fast first result; they click "Pull Again" to fetch the next batch. The cursor is advanced after each batch so subsequent pulls resume exactly where the last one stopped.
>
> Pass `remaining` (how many messages are still allowed) into `syncChannelHistory`. Request `Math.min(100, remaining)` per page. Stop the channel loop once `remaining <= 0`.

```ts
const INITIAL_BACKFILL_LIMIT = 100; // max messages on first pull (rate-limit safety)

export async function backfillSlack(
  connectorId: string,
  onProgress?: (p: SyncProgress) => void,
  maxMessages = INITIAL_BACKFILL_LIMIT,
): Promise<SyncProgress> {
  const { accessToken } = await getTokens(connectorId);
  const prog = { channels: 0, messages: 0 };

  const channels = await backfillSlackChannels(connectorId, accessToken, onProgress, prog);
  const oldest = String(Math.floor((Date.now() - 30 * 86_400_000) / 1000));
  const channelTs: Record<string, string> = {};
  const teamId = channels[0]?.shared_team_ids?.[0] ?? channels[0]?.context_team_id ?? '';

  for (const ch of channels) {
    const remaining = maxMessages - prog.messages;
    if (remaining <= 0) {
      console.log(`[slack] cap (${maxMessages}) reached — skipping remaining channels`);
      break;
    }
    try {
      const newestTs = await syncChannelHistory(
        connectorId, ch.id, ch.name, oldest, accessToken, teamId,
        onProgress, prog, remaining,
      );
      if (newestTs) channelTs[ch.id] = newestTs;
    } catch (err: any) {
      console.warn(`[slack] skip ${ch.id} (${ch.name}):`, err.message);
    }
  }

  // Advance cursor after all inserts succeed
  await updateRow('Integration Connectors', { connector_id: connectorId }, {
    cursor_json: JSON.stringify({ channel_ts: channelTs, channels_fetched_at: new Date().toISOString() }),
    last_synced_at: new Date().toISOString(),
  });
  return prog;
}

async function syncChannelHistory(
  connectorId: string,
  channelId: string,
  channelName: string,
  oldest: string,
  accessToken: string,
  teamId: string,
  onProgress?: (p: SyncProgress) => void,
  prog?: { channels: number; messages: number },
  /** Stop once this many NEW messages have been stored (across all channels). */
  remaining?: number,
): Promise<string | null> {
  let cursor: string | undefined;
  let newestTs: string | null = null;
  const p = prog ?? { channels: 0, messages: 0 };
  let budget = remaining ?? Infinity;

  do {
    if (budget <= 0) break;

    // Request only as many as the remaining budget allows (max 100 per page).
    const pageSize = Math.min(100, budget === Infinity ? 100 : budget);
    const qs = new URLSearchParams({
      channel: channelId,
      oldest,
      limit: String(pageSize),
      ...(cursor ? { cursor } : {}),
    });
    const data = await slackCall(`conversations.history?${qs}`, accessToken);
    const messages: any[] = data.messages ?? [];
    if (messages.length === 0) break;

    // Dedup against the workbook (idempotent re-runs).
    const ids = messages.map(m => m.ts);
    const existing = await listRows('Slack Messages', {
      whereClause: `\`channel_id\` = '${channelId}' AND \`message_ts\` IN (${ids.map(i => `'${i}'`).join(',')})`,
    });
    const seen = new Set(existing.map((r: any) => r.message_ts));
    const fresh = messages.filter(m => !seen.has(String(m.ts)));

    // Fan out into thread replies for messages with reply_count > 0.
    const rows = fresh.map(m => normalizeSlackMessage(connectorId, teamId, channelId, channelName, m));
    for (const m of fresh) {
      if ((m.reply_count ?? 0) > 0) {
        try {
          const replies = await fetchThreadReplies(channelId, m.ts, accessToken);
          rows.push(...replies.slice(1)
            .filter(r => !seen.has(String(r.ts)))
            .map(r => normalizeSlackMessage(connectorId, teamId, channelId, channelName, r)));
        } catch { /* thread fetch failure is non-fatal */ }
      }
    }

    if (rows.length > 0) {
      await bulkInsertRows('Slack Messages', rows);
      p.messages += rows.length;
      budget -= rows.length;
      onProgress?.({ ...p });
    }
    if (rows.length > 0 && !newestTs) newestTs = rows[0].message_ts; // history is newest-first

    cursor = data.response_metadata?.next_cursor || undefined;
  } while (cursor);

  return newestTs;
}
```

## Polling — `syncOne(connectorRow)` for Slack

```ts
export async function syncOne(row: any): Promise<void> {
  if (row.provider !== 'slack' || row.service_type !== 'messages') return;
  const cursor = JSON.parse(row.cursor_json || '{}');
  const channelTs: Record<string, string> = cursor.channel_ts ?? {};
  const channelsFetchedAt = cursor.channels_fetched_at ? new Date(cursor.channels_fetched_at).getTime() : 0;

  if (Object.keys(channelTs).length === 0) {
    // Cursor missing — backfill instead.
    await backfillSlackMessages(row.connector_id);
    return;
  }

  const { accessToken } = await getTokens(row.connector_id);

  // Refresh channel list every 15 min so newly-added channels start syncing.
  let channels: any[];
  if (Date.now() - channelsFetchedAt > 15 * 60_000) {
    channels = await backfillSlackChannels(row.connector_id);
  } else {
    const rows = await listRows('Slack Channels', {
      whereClause: `\`connector_id\` = '${row.connector_id}' AND \`is_archived\` = false`,
    });
    channels = rows.map((r: any) => ({ id: r.channel_id, name: r.channel_name ?? r.name }));
  }

  const nextChannelTs: Record<string, string> = { ...channelTs };
  for (const ch of channels) {
    const oldest = channelTs[ch.id] ?? String(Math.floor((Date.now() - 30 * 86400_000) / 1000));
    const newest = await syncChannelHistory(row.connector_id, ch.id, ch.name, oldest, accessToken);
    if (newest) nextChannelTs[ch.id] = newest;
  }

  // Advance cursor AFTER all inserts succeed.
  await updateRow('Integration Connectors', { connector_id: row.connector_id }, {
    cursor_json: JSON.stringify({
      channel_ts: nextChannelTs,
      channels_fetched_at: new Date(Math.max(channelsFetchedAt, Date.now() - 14 * 60_000)).toISOString(),
    }),
    last_synced_at: new Date().toISOString(),
  });
}
```

On Slack `401` (token revoked or invalid): re-call `getTokens(connectorId)` (force-fresh) and retry once. After two consecutive `401`s, set `oauth_status='error'` on the connector row and surface a "Reconnect Slack" CTA in the UI.

On Slack `429` or `error: "rate_limited"`: respect `Retry-After` (sleep that many seconds), then retry. Don't drop the rest of the channel list.

## Token broker — confirmed response shape

The broker endpoint `GET /connector/api/v1/instance/{connectorId}` returns:

```json
{
  "success": true,
  "data": {
    "id": 7246387,
    "name": "oauth2Connector",
    "connectorType": "SLACK",
    "credentialType": "OAUTH2",
    "userId": 1012,
    "config": [{
      "type": "header",
      "data": {
        "accessToken": "xoxb-...",
        "tokenType": "bot",
        "expiresAt": null,
        "refreshToken": null,
        "scope": "channels:read,..."
      }
    }]
  }
}
```

The access token is at `raw.data.config[0].data.accessToken` — **not** at `raw.accessToken`. Always extract with optional chaining:

```ts
export async function getSlackAccessToken(connectorId: string): Promise<string> {
  const res = await fetch(
    `${DATAGOL_BASE_URL}/connector/api/v1/instance/byId/${connectorId}`,
    { headers: { 'x-auth-token': DATAGOL_SERVICE_TOKEN } },
  );
  if (!res.ok) throw new Error(`Token broker HTTP ${res.status}`);
  const raw = await res.json();
  // Token is nested: data.config[0].data.accessToken
  const token =
    raw.data?.config?.[0]?.data?.accessToken ??
    raw.accessToken ??
    raw.access_token;
  if (!token) throw new Error(`Broker returned no token. Shape: ${JSON.stringify(raw).slice(0, 200)}`);
  return token;
}
```

---

## CORS — Slack API cannot be called directly from the browser

**Slack's Web API does not support browser CORS.** Every call to `https://slack.com/api/*` from a browser `fetch()` fails with a CORS error, regardless of headers or token type. This is Slack's intentional design — their REST API is server-side only.

**DataGOL's backend has no Slack proxy endpoint.** `/connector/api/v1/instance/{id}/proxy` and similar paths all return 404.

**The correct solution: Vite dev proxy + Vercel edge function.**

In `vite.config.ts`, proxy `/slack-api` to `https://slack.com` server-side:

```ts
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/slack-api': {
        // ⚠️  Target must be the ROOT domain, NOT 'https://slack.com/api'.
        // When http-proxy applies a rewritten path that starts with '/', it
        // replaces the entire path on the target — so target 'https://slack.com/api'
        // + rewritten path '/conversations.list' would produce
        // 'https://slack.com/conversations.list' (404), NOT
        // 'https://slack.com/api/conversations.list'.
        // Fix: target the root and inject '/api' in the rewrite instead.
        target: 'https://slack.com',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/slack-api/, '/api'),
      },
    },
  },
});
```

In `src/api/slack.ts`, use the proxy path — **never** `https://slack.com/api` directly:

```ts
// ✅ correct — routes through Vite proxy (dev) or Vercel edge fn (prod)
const SLACK_API = '/slack-api';

// ❌ wrong — CORS error in every browser, every time
// const SLACK_API = 'https://slack.com/api';
```

For production (Vercel), add `api/slack.ts` (edge function) + `vercel.json` rewrite:

```ts
// api/slack.ts — Vercel Edge Function
export const config = { runtime: 'edge' };
export default async function handler(req: Request): Promise<Response> {
  const url = new URL(req.url);
  const slackPath = url.pathname.replace(/^\/api\/slack\/?/, '').replace(/^\/slack-api\/?/, '');
  const slackUrl = `https://slack.com/api/${slackPath}${url.search}`;
  const headers = new Headers(req.headers);
  headers.delete('host');
  const response = await fetch(slackUrl, { method: req.method, headers });
  const out = new Headers(response.headers);
  out.set('Access-Control-Allow-Origin', '*');
  return new Response(response.body, { status: response.status, headers: out });
}
```

```json
// vercel.json
{ "rewrites": [{ "source": "/slack-api/:path*", "destination": "/api/slack/:path*" }] }
```

## Slack API client — `src/api/slack.ts`

The Slack envelope quirk: every response is HTTP 200 even on errors. The body shape is `{ ok: boolean, error?: string, ...payload }`. A wrapper that checks `ok` is mandatory.

```ts
// SLACK_API must be the proxy path, never https://slack.com/api directly (CORS).
const SLACK_API = '/slack-api';

async function slackCall<T = any>(path: string, accessToken: string): Promise<T> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const res = await fetch(`${SLACK_API}/${path}`, {
      headers: { Authorization: `Bearer ${accessToken}` },
    });

    if (res.status === 429) {
      const retry = Number(res.headers.get('retry-after') ?? '1');
      await sleep(retry * 1000);
      continue;
    }
    if (!res.ok) {
      const err = new Error(`Slack ${path} → HTTP ${res.status}`);
      (err as any).status = res.status;
      throw err;
    }

    const body = await res.json();
    if (body.ok === false) {
      if (body.error === 'ratelimited') {
        const retry = Number(res.headers.get('retry-after') ?? '1');
        await sleep(retry * 1000);
        continue;
      }
      const err = new Error(`Slack ${path} → ${body.error ?? 'unknown_error'}`);
      (err as any).slackError = body.error;
      throw err;
    }
    return body as T;
  }
  throw new Error(`Slack ${path} → exhausted retries`);
}

function sleep(ms: number): Promise<void> {
  return new Promise(r => setTimeout(r, ms));
}
```

Endpoints used: `conversations.list`, `conversations.history`, `conversations.replies`. (Future: `users.info` for user resolution; not in v1.)

## Connect button wiring

The parent's Connections page renders sections for Email / Calendar / Storage. Slack adds a fourth — **Messaging** — with a single "Connect Slack" button (Microsoft Teams disabled with "coming soon" until that child skill ships):

```ts
function onConnectSlack() {
  localStorage.setItem('pending_oauth', JSON.stringify({
    service_type: 'messages',
    provider: 'slack',
  }));
  startOAuth('slack');  // navigates the browser
}

// On mount, after reading ?connectorId=&success=true and consuming pending_oauth:
// ⚠️ DataGOL returns ?success=true (NOT ?status=success). Always check both:
const params = new URLSearchParams(window.location.search);
const connectorId = params.get('connectorId');
const isSuccess = params.get('success') === 'true' || params.get('status') === 'success';
const pending = JSON.parse(localStorage.getItem('pending_oauth') ?? '{}');
if (connectorId && isSuccess && pending.provider === 'slack') {
  await insertRow('Integration Connectors', {
    connector_id: connectorId,
    service_type: 'messages',
    provider: 'slack',
    oauth_status: 'connected',
    connected_at: new Date().toISOString().replace(/\.\d{3}Z$/, '+00:00'),
  });
  // ... then hydrate metadata + tokens, then backfillSlackMessages(connectorId)
}
localStorage.removeItem('pending_oauth');
```

`pending_oauth` is the only thing this connector writes to `localStorage`, and it's deleted as soon as it's consumed. Never persist tokens client-side.

## Connections page section (UI)

If the parent's Connections page already exists in the project, **append** a new section — don't rebuild the page. Use the same card pattern:

```
Messaging
Connect Slack to sync channel messages and metadata

  [● Connect Slack]    [■ Connect Microsoft Teams (disabled — coming soon)]
```

Connected state: `team-name@workspace — Disconnect` and `Last sync 2m ago • 14 channels • 3,201 messages`. Counts come from cheap `whereClause` queries against `Slack Channels` / `Slack Messages` filtered by `connector_id`.

Styling follows the host app's convention via `datagol-integrate`. Don't introduce new CSS frameworks.

## Hard rules (Slack-specific)

- **Use `LONG_TEXT` for every text column in `Slack Messages` and `Slack Channels`.** Never use `SINGLE_LINE_TEXT` for any field — including short identifiers like `connector_id`, `team_id`, `channel_id`, `user_id`, `subtype`. This applies even to `message_ts` and `thread_ts` (which must also never be NUMBER — see rule below).
- **Never call `https://slack.com/api/*` directly from the browser.** Slack's REST API has no CORS support — every browser `fetch()` to it fails. Always route through the Vite dev proxy (`/slack-api`) or a Vercel edge function. Set `const SLACK_API = '/slack-api'` and never use the full `https://slack.com/api` base URL in browser-side code.
- **Vite proxy target must be `https://slack.com` (root), not `https://slack.com/api`.** When `http-proxy` applies a rewritten path that starts with `/`, it replaces the entire target path — so `target: 'https://slack.com/api'` + rewritten path `/conversations.list` produces `https://slack.com/conversations.list` (404). The correct config is `target: 'https://slack.com'` with `rewrite: (path) => path.replace(/^\/slack-api/, '/api')`, which produces `https://slack.com/api/conversations.list`.
- **DataGOL's backend has no Slack proxy.** `/connector/api/v1/instance/{id}/proxy` returns 404. There is no pass-through endpoint. The Vite + Vercel proxy pattern is the only working approach for browser apps.
- **Never store `message_ts` (or `thread_ts`) as a NUMBER column.** It's a string like `1234567890.123456`; JS `parseFloat` round-trip loses precision and dedup breaks. Use `LONG_TEXT` (not `SINGLE_LINE_TEXT`, not `NUMBER`).
- **Always check `body.ok` on Slack responses.** Slack returns HTTP 200 with `{ ok: false, error: "..." }` on errors. The `slackFetch()` wrapper above is mandatory; never call `fetch()` directly against Slack endpoints from feature code.
- **On `429` or `error: "ratelimited"`, sleep `Retry-After` seconds then retry.** Surface a one-line warning to the UI; don't crash the poll loop.
- **Always paginate** `conversations.list`, `conversations.history`, `conversations.replies` until `response_metadata.next_cursor` is empty. Slack pages are 200 by default — busy channels need many pages.
- **Always fan out into `conversations.replies` for any message with `reply_count > 0`** during initial backfill. During polling, do this only when the parent message's `latest_reply` exceeds the cursor (otherwise you'll re-fetch the same threads on every cycle).
- **Never request DM scopes (`im:*`, `mpim:*`) in v1.** Bot footprint stays predictable.
- **Never call Slack APIs with `x-auth-token`.** Slack's auth header is `Authorization: Bearer xoxb-…`. Same trap as the Google skill.
- **Always advance `channel_ts[id]` *after* a successful bulk-insert**, never before. If insert fails, the next poll re-runs from the same cursor and dedup keeps it idempotent.
- **Never PATCH a `Slack Channels` row unless its metadata actually changed.** A naïve loop that PATCHes every channel on every pull generates one PATCH per channel even when nothing changed — 50 channels = 50 unnecessary writes before a single message is fetched. The correct pattern: collect *new* channels into a `toInsert[]` array and `bulkInsert` them in one request; PATCH an existing channel only when `name`, `num_members`, `topic`, `purpose`, or `is_archived` differs from the stored value; skip channels whose stored metadata is identical. See `backfillSlackChannels` above for the reference implementation.
- **Don't backfill more than 30 days on first sync** without explicit user confirmation. A workspace with thousands of channels can produce millions of messages.
- **Cap the initial backfill to 100 messages total (across all channels).** Slack `conversations.history` is Tier 3 (~50 calls/min). Fetching without a cap exhausts the rate-limit budget on any workspace with busy channels. Use `INITIAL_BACKFILL_LIMIT = 100` as the default `maxMessages` for `backfillSlack`. Thread a `remaining` budget (= `maxMessages - prog.messages`) into `syncChannelHistory`; request `Math.min(100, remaining)` per page; break out of the channel loop once `remaining <= 0`. The cursor is advanced after each batch so "Pull Again" resumes exactly where the last pull stopped.
- **`channel_name` is a snapshot at sync time.** Channels can be renamed; downstream queries should join through `channel_id`, not `channel_name`. Refresh the snapshot on the 15-min channel-list cycle.
- **The bot only sees channels it's been added to.** Tell the user this in the post-connect message — not seeing a channel's messages is almost always because the bot wasn't invited.
- **Handle `REAUTHENTICATION_REQUIRED` from the token broker explicitly.** When the first `getTokens(connectorId)` call after OAuth returns HTTP 401 with `errorCodes: ["REAUTHENTICATION_REQUIRED"]`, the backend created a connector record but its stored Slack tokens aren't usable. Don't silently swallow. Mark `oauth_status='error'` with `last_error` set to the upstream message, and surface a status line: *"Backend says reauthentication is required. Click Disconnect, then Connect Slack again — and if that still fails, remove the DataGOL app from the Slack workspace before reconnecting."* The retry path lives in the existing Connect button; no separate UI element needed.

## Cross-references

- **`datagol-connector`** — parent. Shared mechanics (workbook schema, OAuth handoff, polling loop, hard rules) all live there.
- **`datagol-app-auth`** — service-token + env-switching foundations. Required before any DataGOL API call.
- **`datagol-google-connector`** — sibling. Same architectural pattern; useful as a structural reference.
- **`datagol-workbook-operations`** — full reference for the workbook read/write APIs the `bulkInsertRows`, `updateRow`, etc. wrappers use.
- **`datagol-integrate`** — when grafting the Connections page (or extending it with the Messaging section) into an existing user repo.
- **`datagol-create-dashboard`** / **`datagol-create-agent`** — common downstream features once Slack data is in workbooks: build a dashboard over channel-message volume, or an agent that answers questions about decisions made in #engineering.
