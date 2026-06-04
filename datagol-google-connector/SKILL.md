---
name: datagol-google-connector
description: Google-specific connector implementation for DataGOL-generated apps — Gmail and Calendar. Triggered by "build a gmail connector", "connect google", "sync my inbox", "build a google calendar integration", "sync my calendar", "build a google integration". Layers on top of `datagol-connector` (parent), supplying the Gmail and Calendar workbook schemas, OAuth scopes, backfill logic (Gmail `messages.list` with `newer_than:30d`; Calendar `events.list` with `timeMin`), polling logic (Gmail `history.list`; Calendar `events.list` with `syncToken` + 410 fallback), MIME decoding, and the Google API client wrappers used in the generated app. Drive and Microsoft are out of scope — those are separate child skills.
---

# DataGOL Google Connector — Gmail + Calendar

This skill is a child of **`datagol-connector`**. It supplies *only* the Google-specific bits: scopes, paths, row schemas, backfill queries, sync semantics. The shared mechanics (OAuth handoff, `Integration Connectors` workbook, token broker, polling loop, hard rules) all live in the parent.

> **Read `datagol-connector` and `datagol-app-auth` first.** This skill assumes you've already done the pre-flight checklist (workspace, service account, `Integration Connectors` workbook, Connections page shell) from those skills. Don't duplicate their setup steps here.

## When to use

- The user says "build a gmail connector", "sync my inbox", "connect google", "build a google calendar integration", "sync my calendar", "build a google integration".
- The user wants to surface email or calendar data inside DataGOL — search, dashboards, agents that answer questions over the data, etc. (Those downstream features are scaffolded by other skills like `datagol-create-dashboard` and `datagol-create-agent`; this skill provides the *data*.)
- The user is OK with browser-only polling (page must be open). If they need always-on sync, surface that constraint and offer to flag a follow-up — there's no backend scheduler today.

## OAuth scopes

Default to **read-only**:

| Service  | Scope                                                  |
|----------|--------------------------------------------------------|
| Gmail    | `https://www.googleapis.com/auth/gmail.readonly`       |
| Calendar | `https://www.googleapis.com/auth/calendar.readonly`    |

If the user wants to send mail, label, archive, or modify events, ask before requesting broader scopes:

- Gmail send/modify → `https://www.googleapis.com/auth/gmail.modify` or `gmail.send`.
- Calendar write → `https://www.googleapis.com/auth/calendar`.

Scopes are configured **on the DataGOL backend's OAuth client**, not in the generated app. The frontend never assembles the consent URL — it just calls `/idp/api/v1/oauth2/{service}/authorize`. To request broader scopes, the user (or a backend admin) needs to add them to the OAuth client config in Google Cloud Console first. Mention this if the user asks for write access.

## Service paths

| Service  | `<servicePath>` in `/idp/api/v1/oauth2/{servicePath}/authorize` |
|----------|------------------------------------------------------------------|
| Gmail    | `gmail`                                                          |
| Calendar | `google_calendar`                                                |

Both render in the parent's Connections page as Google buttons under their respective sections (Email / Calendar). The Storage section's Google button (Drive) is `disabled` with "coming soon" — Drive is a separate future child skill.

## Workbook schemas

Two data workbooks. Both reference `connector_id` so the row links back to the row in `Integration Connectors` — useful for filtering by account when multiple Google accounts are connected.

### `Gmail Messages`

| internal name      | type      | notes                                       |
|--------------------|-----------|---------------------------------------------|
| `connector_id`     | LONG_TEXT | FK to `Integration Connectors.connector_id` |
| `message_id`       | LONG_TEXT | unique per (account, message); Gmail `id`   |
| `thread_id`        | LONG_TEXT | Gmail `threadId`                            |
| `from_address`     | LONG_TEXT | parsed `From` header                        |
| `to_addresses`     | LONG_TEXT | comma-joined `To`                           |
| `cc_addresses`     | LONG_TEXT | comma-joined `Cc`                           |
| `bcc_addresses`    | LONG_TEXT | comma-joined `Bcc`                          |
| `subject`          | LONG_TEXT | parsed `Subject`                            |
| `snippet`          | LONG_TEXT | Gmail's preview snippet                     |
| `body_plain`       | LONG_TEXT | decoded `text/plain` part                   |
| `body_html`        | LONG_TEXT | decoded `text/html` part                    |
| `received_at`      | DATE      | ISO 8601 with tz; from `internalDate`       |
| `labels`           | LONG_TEXT | comma-joined Gmail label ids                |
| `is_unread`        | BOOLEAN   | derived from `UNREAD` label                 |
| `has_attachments`  | BOOLEAN   | true if any part has `filename` set         |

### `Calendar Events`

| internal name        | type      | notes                                            |
|----------------------|-----------|--------------------------------------------------|
| `connector_id`       | LONG_TEXT | FK to `Integration Connectors.connector_id`      |
| `event_id`           | LONG_TEXT | unique per (calendar, event); Google `id`        |
| `calendar_id`        | LONG_TEXT | usually `primary`                                |
| `title`              | LONG_TEXT | Google `summary` field                           |
| `description`        | LONG_TEXT |                                                  |
| `location`           | LONG_TEXT |                                                  |
| `start_at`           | DATE      | ISO 8601                                         |
| `end_at`             | DATE      | ISO 8601                                         |
| `is_all_day`         | BOOLEAN   | true if `start.date` (not `start.dateTime`)      |
| `organizer_email`    | LONG_TEXT |                                                  |
| `attendees`          | LONG_TEXT | JSON-encoded array of `{email, responseStatus}`  |
| `status`             | LONG_TEXT | `confirmed` \| `tentative` \| `cancelled`        |
| `recurrence`         | LONG_TEXT | RRULE string(s); empty for non-recurring         |
| `hangout_link`       | LONG_TEXT | Google Meet link if present                      |

Use `datagol_create_workbook` for both. **Before creating either workbook, always ask the user which workspace to use** — call `datagol_list_workspaces`, present the list, and get explicit confirmation. Never silently pick a default. Discover existing workbooks with `datagol_get_workspace_schema` on the confirmed workspace first — if they already exist (e.g., the user re-connects after disconnecting), reuse them.

## Cursor encoding (`Integration Connectors.cursor_json`)

| service_type | shape                                                              |
|--------------|--------------------------------------------------------------------|
| `gmail`      | `{ "history_id": "<gmail historyId>" }`                            |
| `calendar`   | `{ "sync_token": "<google nextSyncToken>", "calendar_id": "primary" }` |

Encoding as JSON inside `LONG_TEXT` keeps the workbook schema generic across all providers. Read with `JSON.parse(row.cursor_json || '{}')`, write with `JSON.stringify({...})`.

## Initial backfill — Gmail

Last 30 days, paginated. Uses two Gmail API calls per page: `messages.list` (ids only) then `messages.batchGet` (full content).

```ts
// src/api/google.ts (excerpt)
const GMAIL = 'https://gmail.googleapis.com/gmail/v1/users/me';

export async function backfillGmail(connectorId: string): Promise<void> {
  const { accessToken } = await getTokens(connectorId);
  let pageToken: string | undefined;
  let total = 0;

  do {
    const list = await gFetch(`${GMAIL}/messages?q=newer_than:30d&maxResults=100`
      + (pageToken ? `&pageToken=${pageToken}` : ''), accessToken);
    const ids: string[] = (list.messages ?? []).map((m: any) => m.id);
    if (ids.length === 0) break;

    // Skip ids already in the workbook (idempotent re-runs).
    const newIds = await filterExistingMessageIds(connectorId, ids);
    if (newIds.length > 0) {
      const fulls = await batchGetMessages(accessToken, newIds);
      const rows = fulls.map((m) => normalizeGmailMessage(connectorId, m));
      await bulkInsertRows('Gmail Messages', rows);
      total += rows.length;
    }

    pageToken = list.nextPageToken;
  } while (pageToken);

  // Capture the historyId AFTER the backfill so polling picks up
  // anything that arrived during the backfill itself.
  const profile = await gFetch(`${GMAIL}/profile`, accessToken);
  await updateRow('Integration Connectors', { connector_id: connectorId }, {
    cursor_json: JSON.stringify({ history_id: profile.historyId }),
    last_synced_at: new Date().toISOString(),
  });
}
```

`gFetch` is a thin wrapper that sets `Authorization: Bearer <accessToken>` and parses JSON. `batchGetMessages` is `messages.batchGet` (preferred) or N parallel `messages.get` calls if batchGet is unavailable.

`normalizeGmailMessage` does the heavy lifting:

```ts
function normalizeGmailMessage(connectorId: string, m: any) {
  const headers: Record<string, string> = {};
  for (const h of m.payload?.headers ?? []) {
    headers[h.name.toLowerCase()] = h.value;
  }
  const { plain, html } = extractMimeBodies(m.payload);
  const labels: string[] = m.labelIds ?? [];

  return {
    connector_id: connectorId,
    message_id: m.id,
    thread_id: m.threadId,
    from_address: headers['from'] ?? '',
    to_addresses: headers['to'] ?? '',
    cc_addresses: headers['cc'] ?? '',
    bcc_addresses: headers['bcc'] ?? '',
    subject: headers['subject'] ?? '',
    snippet: m.snippet ?? '',
    body_plain: plain,
    body_html: html,
    received_at: new Date(Number(m.internalDate)).toISOString(),
    labels: labels.join(','),
    is_unread: labels.includes('UNREAD'),
    has_attachments: hasAttachments(m.payload),
  };
}
```

`extractMimeBodies` walks `m.payload.parts` recursively, finds `text/plain` and `text/html`, and base64url-decodes:

```ts
function extractMimeBodies(part: any): { plain: string; html: string } {
  let plain = '';
  let html = '';
  function walk(p: any) {
    if (!p) return;
    if (p.mimeType === 'text/plain' && p.body?.data && !plain) {
      plain = decodeBase64Url(p.body.data);
    } else if (p.mimeType === 'text/html' && p.body?.data && !html) {
      html = decodeBase64Url(p.body.data);
    }
    for (const child of p.parts ?? []) walk(child);
  }
  walk(part);
  return { plain, html };
}

function decodeBase64Url(s: string): string {
  const padded = s.replace(/-/g, '+').replace(/_/g, '/')
    + '==='.slice(0, (4 - (s.length % 4)) % 4);
  // browsers: atob produces a "binary string"; convert to UTF-8.
  const binary = atob(padded);
  const bytes = Uint8Array.from(binary, (c) => c.charCodeAt(0));
  return new TextDecoder('utf-8').decode(bytes);
}
```

`filterExistingMessageIds` uses a single `POST /cursor` query against the data workbook with a `whereClause` like `` `message_id` IN ('id1','id2',...) `` — escape single quotes, batch in groups of 100 ids per query.

## Initial backfill — Calendar

```ts
const CAL = 'https://www.googleapis.com/calendar/v3/calendars/primary';

export async function backfillCalendar(connectorId: string): Promise<void> {
  const { accessToken } = await getTokens(connectorId);
  const timeMin = new Date(Date.now() - 30 * 86400_000).toISOString();
  let pageToken: string | undefined;
  let nextSyncToken: string | undefined;

  do {
    const qs = new URLSearchParams({
      timeMin,
      singleEvents: 'true',
      orderBy: 'startTime',
      maxResults: '250',
    });
    if (pageToken) qs.set('pageToken', pageToken);

    const data = await gFetch(`${CAL}/events?${qs}`, accessToken);
    const events = (data.items ?? []).map((e: any) => normalizeCalendarEvent(connectorId, 'primary', e));
    const newEvents = await filterExistingEventIds(connectorId, events);
    if (newEvents.length > 0) await bulkInsertRows('Calendar Events', newEvents);

    pageToken = data.nextPageToken;
    nextSyncToken = data.nextSyncToken;
  } while (pageToken);

  await updateRow('Integration Connectors', { connector_id: connectorId }, {
    cursor_json: JSON.stringify({ sync_token: nextSyncToken, calendar_id: 'primary' }),
    last_synced_at: new Date().toISOString(),
  });
}

function normalizeCalendarEvent(connectorId: string, calendarId: string, e: any) {
  const isAllDay = !!e.start?.date;
  return {
    connector_id: connectorId,
    event_id: e.id,
    calendar_id: calendarId,
    title: e.summary ?? '',
    description: e.description ?? '',
    location: e.location ?? '',
    start_at: isAllDay
      ? `${e.start.date}T00:00:00+00:00`
      : new Date(e.start.dateTime).toISOString(),
    end_at: isAllDay
      ? `${e.end.date}T00:00:00+00:00`
      : new Date(e.end.dateTime).toISOString(),
    is_all_day: isAllDay,
    organizer_email: e.organizer?.email ?? '',
    attendees: JSON.stringify((e.attendees ?? []).map((a: any) =>
      ({ email: a.email, responseStatus: a.responseStatus }))),
    status: e.status ?? 'confirmed',
    recurrence: (e.recurrence ?? []).join('\n'),
    hangout_link: e.hangoutLink ?? '',
  };
}
```

`singleEvents=true` is **mandatory** — without it, recurring events come back as a single master event with an RRULE and the workbook can't represent the actual instances. With `singleEvents=true`, each occurrence in the time window is its own event.

## Polling — `syncOne(connectorRow)`

The parent's polling loop calls into this function once per connected connector per cycle. Dispatch on `connectorRow.service_type`:

```ts
// src/api/google.ts (excerpt)
export async function syncOne(row: any): Promise<void> {
  if (row.provider !== 'google') return;  // not ours
  const cursor = JSON.parse(row.cursor_json || '{}');
  if (row.service_type === 'gmail')    await syncGmail(row, cursor);
  if (row.service_type === 'calendar') await syncCalendar(row, cursor);
}

async function syncGmail(row: any, cursor: { history_id?: string }): Promise<void> {
  if (!cursor.history_id) {
    // Cursor missing — backfill instead.
    await backfillGmail(row.connector_id);
    return;
  }
  const accessToken = await accessTokenWithRetry(row.connector_id);
  let pageToken: string | undefined;
  let lastHistoryId = cursor.history_id;

  do {
    const qs = new URLSearchParams({
      startHistoryId: cursor.history_id,
      historyTypes: 'messageAdded',
    });
    if (pageToken) qs.set('pageToken', pageToken);

    const data = await gFetch(`${GMAIL}/history?${qs}`, accessToken);
    const ids = new Set<string>();
    for (const h of data.history ?? []) {
      lastHistoryId = h.id ?? lastHistoryId;
      for (const m of h.messagesAdded ?? []) ids.add(m.message.id);
    }
    if (ids.size > 0) {
      const newIds = await filterExistingMessageIds(row.connector_id, [...ids]);
      if (newIds.length > 0) {
        const fulls = await batchGetMessages(accessToken, newIds);
        await bulkInsertRows('Gmail Messages',
          fulls.map((m) => normalizeGmailMessage(row.connector_id, m)));
      }
    }
    pageToken = data.nextPageToken;
  } while (pageToken);

  await updateRow('Integration Connectors', { connector_id: row.connector_id }, {
    cursor_json: JSON.stringify({ history_id: lastHistoryId }),
    last_synced_at: new Date().toISOString(),
  });
}

async function syncCalendar(row: any, cursor: { sync_token?: string; calendar_id?: string }): Promise<void> {
  if (!cursor.sync_token) {
    await backfillCalendar(row.connector_id);
    return;
  }
  const accessToken = await accessTokenWithRetry(row.connector_id);
  const calendarId = cursor.calendar_id ?? 'primary';
  let pageToken: string | undefined;
  let nextSyncToken = cursor.sync_token;

  try {
    do {
      const qs = new URLSearchParams({ syncToken: cursor.sync_token });
      if (pageToken) qs.set('pageToken', pageToken);

      const data = await gFetch(`${CAL.replace('/primary', `/${calendarId}`)}/events?${qs}`, accessToken);
      const events = (data.items ?? []).map((e: any) => normalizeCalendarEvent(row.connector_id, calendarId, e));
      const newEvents = await filterExistingEventIds(row.connector_id, events);
      if (newEvents.length > 0) await bulkInsertRows('Calendar Events', newEvents);

      pageToken = data.nextPageToken;
      nextSyncToken = data.nextSyncToken ?? nextSyncToken;
    } while (pageToken);
  } catch (err: any) {
    if (err.status === 410) {
      // syncToken is dead — full re-sync.
      await updateRow('Integration Connectors', { connector_id: row.connector_id }, {
        cursor_json: JSON.stringify({ calendar_id: calendarId }),  // drop sync_token
      });
      await backfillCalendar(row.connector_id);
      return;
    }
    throw err;
  }

  await updateRow('Integration Connectors', { connector_id: row.connector_id }, {
    cursor_json: JSON.stringify({ sync_token: nextSyncToken, calendar_id: calendarId }),
    last_synced_at: new Date().toISOString(),
  });
}
```

`accessTokenWithRetry` wraps `getTokens` and force-refreshes on cache miss; on a `401` from Google, the caller bypasses cache once more and retries. After two `401`s in a row, mark `oauth_status: 'error'` and set `last_error`.

## Google API client — `src/api/google.ts`

Tiny wrapper around `fetch` with bearer auth and JSON handling:

```ts
async function gFetch<T = any>(url: string, accessToken: string): Promise<T> {
  const res = await fetch(url, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
  if (!res.ok) {
    const err = new Error(`Google ${url} → HTTP ${res.status}`);
    (err as any).status = res.status;
    throw err;
  }
  return res.json() as Promise<T>;
}

async function batchGetMessages(accessToken: string, ids: string[]): Promise<any[]> {
  // Gmail's batchGet endpoint: messages.batchGet doesn't exist in v1 REST —
  // use parallel messages.get calls (max ~10 in flight to respect quota).
  const out: any[] = [];
  const chunks = chunk(ids, 10);
  for (const c of chunks) {
    const got = await Promise.all(c.map((id) =>
      gFetch(`${GMAIL}/messages/${id}?format=full`, accessToken),
    ));
    out.push(...got);
  }
  return out;
}

function chunk<T>(arr: T[], size: number): T[][] {
  const out: T[][] = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}
```

If Gmail releases a real `batchGet` REST endpoint, swap the implementation — the call site stays the same.

## Connect button wiring

The parent's Connections page renders three sections. The Email and Calendar sections' Google buttons each set a per-button "pending service" hint right before navigating, so the page knows which row to insert when the redirect lands:

```ts
function onConnectGmail() {
  localStorage.setItem('pending_oauth', JSON.stringify({
    service_type: 'gmail',
    provider: 'google',
  }));
  startOAuth('gmail');   // navigates the browser
}

function onConnectCalendar() {
  localStorage.setItem('pending_oauth', JSON.stringify({
    service_type: 'calendar',
    provider: 'google',
  }));
  startOAuth('google_calendar');
}

// On mount, after reading ?connectorId=&status=success:
const pending = JSON.parse(localStorage.getItem('pending_oauth') ?? '{}');
localStorage.removeItem('pending_oauth');  // one-shot
await insertRow('Integration Connectors', {
  connector_id: connectorId,
  service_type: pending.service_type,
  provider: pending.provider,
  oauth_status: 'connected',
  connected_at: new Date().toISOString(),
});
```

`pending_oauth` is the only thing this connector writes to `localStorage`, and it's deleted as soon as it's consumed. Don't leave it sitting around.

## Disconnect wiring

When the user disconnects a connector, update the workbook row and then update React state directly — **do NOT re-fetch connectors from the DB immediately after disconnect**. Re-fetching races with the DB write: the query can return before the update propagates and will set the connector back to "Connected" in the UI.

```ts
// ✅ Correct — trust the state update, no re-fetch
onGmailDisconnect={() => setGmailConnector(null)}
onCalDisconnect={() => setCalConnector(null)}

// ❌ Wrong — loadConnector() races with the DB write and reverts the UI
onGmailDisconnect={() => { setGmailConnector(null); loadConnector(); }}
onCalDisconnect={() => { setCalConnector(null); loadConnector(); }}
```

The disconnect handler itself should re-fetch the row by `connector_id` before updating (to get the latest row id), then call the `onDisconnect` prop:

```ts
const handleDisconnect = async () => {
  if (!connector) return;
  setDisconnecting(true);
  setDisconnectError(null);
  try {
    // Re-fetch by connector_id to guarantee the correct row id
    const row = await connectors.findByConnectorId(connector.connector_id);
    const rowId = row?.id ?? connector.id;
    await connectors.update(rowId, {
      oauth_status: 'disconnected',
      disconnected_at: toDataGolDate(),
    });
    onDisconnect(); // parent calls setXxxConnector(null) — no loadConnector()
  } catch (e: any) {
    setDisconnectError(e?.message ?? 'Disconnect failed');
  } finally {
    setDisconnecting(false);
  }
};
```

## Hard rules (Google-specific)

- **Always ask the user which workspace to use before creating `Gmail Messages` or `Calendar Events` workbooks.**
- **Use `LONG_TEXT` for every text column in `Gmail Messages` and `Calendar Events`.** Never use `SINGLE_LINE_TEXT` for any field — including short ones like `connector_id`, `message_id`, `event_id`, `status`, `calendar_id`. Gmail message IDs, subjects, and headers can be arbitrarily long; `SINGLE_LINE_TEXT` truncates silently and corrupts dedup. Call `datagol_list_workspaces`, show the list, and wait for explicit confirmation. Never silently use a default workspace.
- **Never put any Google OAuth client credentials (id or secret) into a generated app or any file in this repo.** Both live on the DataGOL backend. The frontend only ever sees the `connectorId`, then later the broker-issued `accessToken`/`refreshToken`.
- **Always decode Gmail MIME with the standard base64url + `Content-Transfer-Encoding` handling.** Don't paste base64 strings into `body_plain` — search/AI features over the workbook will return garbled hits.
- **Always set `singleEvents=true` on the Calendar backfill query.** Recurring events must expand to instances; an unexpanded master event with just an RRULE is unusable downstream.
- **On `410 Gone` from Calendar, the `syncToken` is dead.** Full re-sync is the *only* recovery — drop the token, re-run `backfillCalendar`, write a new token. Don't try to "repair" or "patch" the cursor.
- **Filter Gmail `messages.list` to `q=newer_than:30d` on first sync.** Don't try to index a multi-year mailbox without explicit user confirmation — Gmail rate limits will kick in and the workbook will balloon.
- **Capture Gmail `historyId` *after* the backfill completes**, not before. If you capture it first and the backfill takes three minutes, you'll miss any messages that arrived in those three minutes.
- **Use `singleEvents=true` for the Calendar *backfill* but pass the `syncToken` directly for *deltas*** — Google's incremental sync API doesn't accept `singleEvents` together with `syncToken` (or rather, the token already has that mode baked in). Don't try to send both.
- **Never call Google APIs with `x-auth-token`.** Google's auth header is `Authorization: Bearer <accessToken>`. Mixing them up is the #1 first-try failure.
- **After disconnect, call `setXxxConnector(null)` directly — never call `loadConnector()` in the disconnect callback.** Re-fetching immediately races with the DB write: the query lands before the update propagates and reverts the UI back to "Connected". The DB update already succeeded; just clear local state.
- **Use `google_calendar` as the Calendar service path** in `/idp/api/v1/oauth2/{servicePath}/authorize` — not `gcalendar`.

## Cross-references

- **`datagol-connector`** — parent. The shared mechanics (workbook schema, OAuth handoff, polling loop, hard rules) all live there.
- **`datagol-app-auth`** — service-token + env-switching foundations. Required before any DataGOL API call.
- **`datagol-workbook-operations`** — full reference for the workbook read/write APIs. The `bulkInsertRows`, `updateRow`, etc. wrappers from the parent skill use it.
- **`datagol-integrate`** — when grafting the Connections page into an existing user repo. Apply that skill's mounting and styling rules.
- **`datagol-create-dashboard`** / **`datagol-create-agent`** — common downstream features once Gmail/Calendar data is in workbooks: build a dashboard over inbox volume, or an agent that answers questions about meetings.
