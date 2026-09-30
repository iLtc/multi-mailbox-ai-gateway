# Multi-Mailbox AI Gateway — Design Spec

Date: 2026-09-24
Status: approved in brainstorming, pending written review
Revised: 2026-09-29, merged the selected items from
`2026-09-26-spark-cli-gap-analysis.md`

## 1. Purpose

A self-hosted service that syncs ten personal mailboxes over IMAP into one
encrypted SQLite database and exposes them to AI agents and scripts through a
REST API and an MCP server. Single user. A few calls per day. Agents can read,
search, and create or edit drafts. Nothing is ever sent, moved, or flagged,
and nothing is deleted except the previous copy of a gateway-created draft
when that draft is edited (7.4).

## 2. Decisions made during brainstorming

| Topic | Decision |
|---|---|
| Providers | Gmail / Google Workspace, Outlook.com (OAuth2), Yahoo, Fastmail, iCloud, 126 (NetEase), QQ, other plain IMAP |
| ProtonMail | Out of scope entirely. No Bridge container. |
| Outlook auth | OAuth2 device-code flow built into the worker CLI; tokens stored in the DB |
| Runtime | Node 24, TypeScript |`
| Deployment | Docker Compose on a VPS, behind the user's existing Caddy on a dedicated subdomain |
| Architecture | Two containers from one image: `api` and `worker`, sharing one SQLite volume |
| Database | SQLite via `better-sqlite3-multiple-ciphers`, page-encrypted with a key from env, WAL mode, FTS5 |
| Folders synced | INBOX, Sent, Archive for every account, plus any folder listed in the account's opt-in `extra_folders` (Trash, Spam, Drafts, custom folders). Gmail: All Mail only, with labels. Full history, no time cutoff. |
| Aliases | `aliases` per account, exact addresses or `*@domain` catch-all patterns. A reply draft is sent From the address the original was received at. |
| Sync trigger | Timer every 10 minutes plus an explicit `sync` operation that can target all, one, or several accounts |
| Recent bound | Total limit (default 50) with an optional multi-account filter |
| Read operations | unread, recent, search, get_message, get_thread, plus status |
| Filters and paging | Search filters on from, to, cc, subject, has_attachment, flagged, dates, unread, and Gmail labels. `offset` pagination on list operations. Optional full bodies on search. |
| Message ids | Opaque `msg_<rowid>` |
| Drafts | Plain text only, reply and reply-all threading via stored headers, From chosen among the account address and its aliases, gateway-created drafts can be edited, no HTML, no attachments |
| HTML bodies | Converted to text with links preserved as `[text](url)` |
| Attachments | Metadata only (name, type, size). Never downloaded. |
| Chinese search | Per-character CJK tokenization with phrase queries. No trigram index. |
| Auth | Authentik is the only token issuer. OAuth 2.1 for claude.ai / ChatGPT; long-lived Authentik JWTs for apps without OAuth. No static tokens. |
| Revocation | Authentik token introspection, cached 5 minutes, fail closed |
| Audit log | Every service call recorded in the DB and to stdout, queryable over REST and CLI |
| Clients | Claude Code, claude.ai / Claude Desktop connector, chatgpt.com connector, Codex, OpenClaw, Hermes, scripts over REST, at least one app that only supports REST with a bearer token |
| Agent skill file | `SKILL.md` alongside the README, for Codex, scripts, and other REST-only clients |

## 3. Out of scope

Sending mail, deleting (other than replacing a gateway-created draft on
edit), moving, marking spam, changing flags from the API, attachment download,
HTML drafts, draft attachments, forwarding, signatures, calendar, contacts,
multi-user, ProtonMail, a web UI.

## 4. Components

One Docker image, two containers, one named volume.

### 4.1 `api` container

- Hono on Node 24.
- Serves REST under `/v1/*`, MCP at `/mcp` (Streamable HTTP, stateless, JSON
  responses), OAuth protected-resource metadata at
  `/.well-known/oauth-protected-resource`, and `GET /healthz`.
- Opens the DB read-mostly. Writes exactly two tables: `sync_requests` and
  `audit_log`.
- Creating and editing drafts are the only operations where the API itself
  opens a short-lived IMAP connection, to APPEND and, on edit, to fetch and
  remove the previous copy. Routing this through the worker would add a second
  IPC path for two operations.

### 4.2 `worker` container

- No HTTP.
- Runs the scheduler: incremental sync every `SYNC_INTERVAL_MINUTES` (default
  10), backfill continuously until every folder is done, nightly
  reconciliation, nightly audit pruning.
- Owns all IMAP reads and Outlook token refresh.
- Polls `sync_requests` every 2 seconds and writes a heartbeat to `meta` every
  tick.
- Ships the CLI (`auth`, `status`, `resync`, `audit`).

### 4.3 Shared volume `/data`

- `mail.db` plus WAL and SHM files. Both containers open it with the same
  `DB_KEY`.

### 4.4 IPC

SQLite is the only IPC. There is no broker or queue.

- `sync`: API inserts a `sync_requests` row, polls it for up to 60 s, returns
  the finished result or `{status: "queued", request_id}`.
- Worker liveness: `meta.worker_heartbeat_at`, read by `/healthz` and
  `get_status`.

## 5. Data model

All tables live in one encrypted database. Migrations are numbered SQL files
applied at startup by whichever process starts first, under `BEGIN IMMEDIATE`.

```
accounts
  id INTEGER PK
  name TEXT UNIQUE            -- from accounts.yaml, the public account name
  email TEXT
  aliases_json TEXT           -- JSON array from accounts.yaml: addresses or *@domain patterns
  provider TEXT               -- gmail | outlook | yahoo | fastmail | icloud | 126 | qq | generic
  enabled INTEGER
  drafts_path TEXT            -- cached special-use \Drafts path
  last_success_at TEXT
  last_error TEXT
  last_error_at TEXT

folders
  id INTEGER PK
  account_id INTEGER FK
  path TEXT                   -- IMAP mailbox path
  role TEXT                   -- inbox | sent | archive | all | custom
  uidvalidity INTEGER
  highest_uid INTEGER         -- incremental cursor; fetch UID > highest_uid
  backfill_low_uid INTEGER    -- backfill cursor; next chunk is [low-200, low-1]
  backfill_done INTEGER
  highest_modseq INTEGER      -- CONDSTORE cursor, NULL if unsupported
  supports_condstore INTEGER
  supports_qresync INTEGER
  last_reconciled_at TEXT
  UNIQUE (account_id, path)

messages
  id INTEGER PK               -- exposed as msg_<id>
  folder_id INTEGER FK
  uid INTEGER
  message_id TEXT
  in_reply_to TEXT
  references_json TEXT        -- JSON array of Message-IDs
  from_addr TEXT
  from_name TEXT
  reply_to_addr TEXT
  to_json TEXT                -- JSON array of {name, address}
  cc_json TEXT
  delivered_to_json TEXT      -- JSON array of addresses from Delivered-To / X-Delivered-To / X-Original-To
  subject TEXT
  date TEXT                   -- ISO 8601 UTC from Date header, fallback INTERNALDATE
  unread INTEGER              -- 1 when \Seen absent
  flagged INTEGER
  labels_json TEXT            -- Gmail X-GM-LABELS, NULL elsewhere
  snippet TEXT                -- first 200 chars of text_body, whitespace collapsed
  text_body TEXT              -- plain text, HTML converted if no text part, capped 512 KB
  has_attachments INTEGER
  attachments_json TEXT       -- JSON array of {filename, content_type, size}
  size INTEGER
  synced_at TEXT
  UNIQUE (folder_id, uid)
  INDEX (date DESC), INDEX (message_id), INDEX (in_reply_to), INDEX (folder_id, unread)

messages_fts  (FTS5, external content = messages, tokenize = 'unicode61 remove_diacritics 2')
  subject_idx, from_idx, to_idx, body_idx
  -- each *_idx column holds the CJK-expanded form of the source column (see 5.2)
  -- kept in sync by INSERT / UPDATE / DELETE triggers on messages

oauth_tokens
  account_id INTEGER PK FK
  refresh_token TEXT
  access_token TEXT
  expires_at TEXT
  updated_at TEXT

sync_requests
  id INTEGER PK
  accounts_json TEXT          -- JSON array of account names, or NULL for all
  status TEXT                 -- queued | running | done | error
  result_json TEXT            -- per-account {new, updated, error}
  created_at TEXT
  started_at TEXT
  finished_at TEXT

audit_log
  id INTEGER PK
  ts TEXT
  request_id TEXT
  client_ip TEXT
  subject TEXT                -- JWT sub / preferred_username
  client_id TEXT              -- JWT azp / client_id
  client_name TEXT            -- Authentik client name when known
  transport TEXT              -- rest | mcp
  operation TEXT
  params_json TEXT            -- redacted, see 9
  result_json TEXT
  error_code TEXT
  duration_ms INTEGER

meta
  key TEXT PK
  value TEXT                  -- worker_heartbeat_at, schema_version
```

### 5.1 Gmail stored once

For Gmail accounts the worker syncs only the `\All` folder (`[Gmail]/All Mail`)
and stores `X-GM-LABELS` per message. Role membership is derived:

- inbox: labels contain `\Inbox`
- sent: labels contain `\Sent`
- archive: neither `\Inbox` nor `\Sent`, and not `\Trash` / `\Spam`

For all other providers, INBOX, Sent, and Archive are distinct `folders` rows.
Accounts whose provider has no Archive folder simply have no archive row.

Every Gmail label, including user labels, is reachable through the `labels`
filter (7.2) because All Mail carries them all. Gmail's Trash and Spam are not
part of All Mail; they are synced only when listed in `extra_folders` (5.4).

### 5.4 Extra folders

Each account may list additional IMAP folder paths under `extra_folders` in
`accounts.yaml`: Trash, Spam, Drafts, or any custom folder (for example the
Chinese-named folders on 126). Each is synced as a `folders` row with
`role = custom`, through the same incremental, backfill, and reconciliation
paths as the role folders.

- On Gmail only the `\Trash` and `\Junk` special-use folders are accepted.
  Any other path is a label folder that would duplicate All Mail; it is
  skipped with a warning.
- A path that does not exist on the server is skipped with a warning and
  reported in `accounts.last_error`.
- A path removed from `extra_folders` has its `folders` row and messages
  deleted on the next pass.
- Custom folders are never included by default. Reads return them only when
  the `folders` filter names the path explicitly (7.1).

### 5.2 Chinese and CJK search

FTS5's `unicode61` tokenizer treats a run of CJK ideographs as one token, so a
two-character word would not match inside a longer run. The `trigram`
tokenizer needs three-character terms and most Chinese words are two
characters. Neither is acceptable.

Instead, at index time a text transform inserts a space between every pair of
adjacent CJK characters (Unicode blocks CJK Unified Ideographs, Extension A,
Compatibility Ideographs, Hiragana, Katakana, Hangul) before the text enters
the FTS columns. Latin text is untouched. At query time the same transform is
applied to the user's query, and each CJK run becomes a double-quoted phrase
so the characters must appear adjacent and in order. Result: any CJK substring
of any length matches exactly; Latin words keep normal word matching with
diacritic folding.

Charset decoding for GB2312 / GBK / GB18030 subjects and bodies from 126 and
QQ is done by mailparser.

### 5.3 Body storage

- HTML is converted to plain text with `html-to-text` at sync time when no
  `text/plain` part exists.
- Links survive the conversion. `html-to-text` is configured so an anchor
  renders as `[text](url)`, or as the bare URL when the text equals the href.
  `mailto:` links keep the address. Image sources are dropped.
- `text_body` is capped at 512 KB. Raw HTML and raw MIME are not stored.
- Rough sizing: 10 KB per message average, 200k messages ≈ 2 GB plus FTS.

## 6. Sync engine

### 6.1 Connections and provider profiles

- One short-lived imapflow connection per account per pass: connect, work,
  logout. Up to 3 accounts in parallel.
- Provider profiles supply host, port, and TLS defaults for gmail, outlook,
  yahoo, fastmail, icloud, 126, qq, and generic. `accounts.yaml` entries may
  override host, port, and user.
- Connection and command timeouts: 30 s.
- All fetches use PEEK. Nothing is ever marked `\Seen` by the gateway.

### 6.2 Folder discovery

Each pass lists folders and maps special-use attributes to roles: `INBOX`
→ inbox, `\Sent` → sent, `\Archive` → archive, `\Drafts` → cached in
`accounts.drafts_path`. Gmail maps only `\All` → all. Paths listed in the
account's `extra_folders` map to custom (5.4). Unknown or missing folders are
skipped. New folder rows get `highest_uid = UIDNEXT - 1` and
`backfill_low_uid = UIDNEXT`, so new mail is caught immediately while backfill
walks backwards.

### 6.3 Fetch pipeline

For each UID: fetch flags, envelope, INTERNALDATE, size, body structure, the
`Delivered-To`, `X-Delivered-To`, and `X-Original-To` header fields, and
(Gmail) labels. From the body structure, download only `text/plain` and
`text/html` leaf parts. Attachments are never downloaded; their filename,
content type, and size come from the body structure. Parse with mailparser,
convert HTML to text if needed, compute snippet, upsert the `messages` row.
Chunks are committed in one transaction each.

### 6.4 Incremental sync (every tick)

Per folder:

1. Select. If UIDVALIDITY changed: delete all rows for the folder, reset both
   cursors and `highest_modseq`, log, continue as first sight.
2. Fetch UIDs `> highest_uid` through the pipeline. Advance cursor on commit.
3. Flag refresh:
   - If the server supports CONDSTORE: fetch flags (and Gmail labels)
     `CHANGEDSINCE highest_modseq` across the whole folder, update `unread`,
     `flagged`, `labels_json`, store the new `highest_modseq`. With QRESYNC,
     also delete rows for VANISHED UIDs.
   - Otherwise: fetch flags for messages dated within the last 30 days,
     update them, delete rows whose UID is missing from the response.
4. Update `accounts.last_success_at`.

### 6.5 Backfill (continuous until done)

Newest first in chunks of 200 UIDs: `[backfill_low_uid - 200,
backfill_low_uid - 1]`, through the pipeline, commit, move the cursor down.
Done when the cursor reaches 1. Resumable at any restart since cursors only
advance on commit. The worker interleaves: each tick runs incremental sync for
all accounts first, then spends the remaining interval on backfill chunks
round-robin across accounts with a 1 s pause between chunks to stay under
Gmail's daily IMAP bandwidth limit.

### 6.6 Nightly reconciliation

Once per day per folder, fetch the complete UID list (UIDs only) and delete
rows whose UID no longer exists. For folders without CONDSTORE, also run a
full flags fetch so read-state staleness for old mail is at most one day.
Also prune `audit_log` rows older than `AUDIT_RETENTION_DAYS`.

### 6.7 Sync requests

Between chunks the worker checks `sync_requests` for `queued` rows. It marks
the row `running`, runs an incremental pass for the listed accounts (or all)
ahead of any backfill work, writes per-account `{new, updated, error}` into
`result_json`, and marks it `done` or `error`.

### 6.8 Outlook OAuth2

- Hand-rolled device-code flow against
  `https://login.microsoftonline.com/consumers/oauth2/v2.0/{devicecode,token}`.
  Scopes: `https://outlook.office.com/IMAP.AccessAsUser.All offline_access`.
- One-time login: `docker compose exec worker node dist/cli.js auth <account>`
  prints the URL and code, polls until complete, writes `oauth_tokens`.
- The worker refreshes the access token when within 5 minutes of expiry and
  passes it to imapflow as XOAUTH2 (`auth: {user, accessToken}`).
- Refresh failure: `accounts.last_error` is set to a message naming the CLI
  command to rerun; status becomes `error` immediately.
- User prerequisite: one Azure app registration, public client, personal
  Microsoft accounts audience, device code flow allowed. Client id in
  `MS_CLIENT_ID`.

### 6.9 Error handling

- Account failures are isolated. Errors are written to `accounts.last_error`
  and logged with pino. The account is retried next tick with exponential
  backoff up to 1 hour. Auth failures (bad password, rejected XOAUTH2) stop
  retries until the worker restarts.
- A crash mid-chunk loses at most that chunk.
- Status rules (computed at read time, see 7.3): `error` if no successful pass
  in the last hour or the last attempt was an auth failure; `backfilling` while
  any folder has backfill work; `never_synced` before the first success;
  otherwise `ok`.

## 7. API surface

One service module with ten functions. REST routes and MCP tools are thin
wrappers. Each function has one zod input schema used for REST validation and
as the MCP tool input schema.

### 7.1 Conventions

- `accounts`: optional array of account names; omitted means all enabled.
- `folders`: optional array whose entries are a role (`inbox | sent |
  archive`) or the exact IMAP path of a custom folder from `extra_folders`.
  "All" as a default means the three roles; custom folders are returned only
  when named.
- `labels`: optional array of Gmail label names as stored in `X-GM-LABELS`
  (`\Starred`, `\Important`, or a user label name). A message must carry
  every listed label. Giving `labels` restricts the call to Gmail accounts.
- `offset`: optional, default 0. Rows to skip, for walking past the first
  page. A page shorter than `limit` is the last one.
- Every read response includes `synced_at`: the oldest `last_success_at`
  among covered accounts.
- Summary shape: `id, account, folder, from {name, email}, subject, date,
  unread, has_attachments, snippet`. `folder` is the role, or the path for a
  custom folder.
- Errors: `{error: {code, message}}`. 400 validation, 401 auth, 404 unknown
  id, account, or draft uid, 409 draft to a disabled or errored account or an
  edit of a draft the gateway did not create, 502 IMAP append failure, 503
  from `/healthz` and from auth when Authentik is unreachable.
- Reads never fail because the worker is down; they return an older
  `synced_at`.

### 7.2 Operations

| Function | REST | Input | Output |
|---|---|---|---|
| `get_unread` | `GET /v1/unread` | accounts, limit (50, max 200), offset | summaries, INBOX only, newest first |
| `get_recent` | `GET /v1/recent` | accounts, folders (default inbox), labels, limit (50, max 200), offset, order (`desc` default, or `asc`) | summaries by date in the given order |
| `search` | `GET /v1/search` | query, accounts, folders (default all), labels, from, to, cc, subject, after, before, unread_only, has_attachment, flagged, include_body, limit (25, max 100), offset | summaries ranked by bm25; newest first when `query` is omitted |
| `get_message` | `GET /v1/messages/:id` | id | full headers including labels and delivered-to, text_body, attachments metadata, thread_root |
| `get_thread` | `GET /v1/messages/:id/thread` | id | up to 30 messages, oldest first, bodies capped at 8 KB with `truncated` |
| `create_draft` | `POST /v1/drafts` | account, from, to[], cc[], bcc[], subject, text, reply_to_message_id, reply_all | account, folder, uid, from |
| `update_draft` | `PATCH /v1/drafts/:account/:uid` | account, uid, from, to[], cc[], bcc[], subject, text (all but account and uid optional) | account, folder, uid (new), from |
| `sync` | `POST /v1/sync` | accounts, wait (true, ≤ 60 s) | per-account counts, or queued status |
| `get_status` | `GET /v1/status` | none | see 7.3 |
| `get_audit` | `GET /v1/audit` (REST only) | client, operation, since, limit | audit rows |
| health | `GET /healthz` (no auth) | none | 200 / 503 |

Search query handling: terms are split on whitespace, each term is
double-quoted before entering the FTS MATCH expression so user input cannot
cause an FTS syntax error, CJK runs are expanded per 5.2, and terms are ANDed.
`from` filters on `from_addr` or `from_name` with LIKE. `after` / `before` are
ISO dates.

Additional search filters, all optional and ANDed with the rest:

- `to`: LIKE on names and addresses in `to_json`, and on
  `delivered_to_json` so mail received at an alias through Bcc or a catch-all
  is found.
- `cc`: LIKE on names and addresses in `cc_json`.
- `subject`: LIKE on `subject`. Substring match, so it works for CJK without
  going through FTS.
- `has_attachment`, `flagged`: booleans on the existing columns.
- `labels`: per 7.1, matched with `json_each` over `labels_json`.

`query` is optional when at least one other filter is given; the results are
then ordered newest first instead of by bm25. A call with neither a query nor
a filter is a 400.

`include_body: true` adds `text_body` to each search result, capped at 8 KB
with `truncated`, the same as `get_thread`.

Thread walking: starting from the message, collect the set of Message-IDs from
its `message_id`, `in_reply_to`, and `references_json`; repeatedly select
messages in the same account whose `message_id` or `in_reply_to` or
`references_json` intersect the set until it stops growing or reaches 30
messages. Cross-folder within one account.

### 7.3 Status response

```
{
  worker: { heartbeat_at, alive },          // alive = heartbeat within 15 min
  db: { size_bytes, message_count },
  sync_requests: [ { id, status, accounts, created_at } ],   // queued or running
  accounts: [ {
    name, email, aliases, provider, enabled,
    status,                                  // ok | backfilling | error | never_synced
    last_success_at, last_error, last_error_at,
    message_count, unread_count,
    folders: [ { path, role, indexed, backfill_done, estimated_remaining } ],
    labels: [ { name, count } ]              // Gmail only
  } ]
}
```

`folders` includes custom folders, so this is where an agent learns the paths
it can pass to the `folders` filter, the label names it can pass to `labels`,
and the aliases it can pass as `from` on a draft.

`/healthz` returns 503 if the worker heartbeat is older than 15 minutes or the
database cannot be opened.

### 7.4 Draft semantics

1. Resolve the account, its IMAP settings, and cached `drafts_path`. If the
   account is disabled or in `error`, return 409.
2. If `reply_to_message_id` is given: load the original, set `In-Reply-To` to
   its Message-ID, set `References` to its references plus its Message-ID,
   prefix subject with `Re: ` if not already present, and default `to` to the
   original's Reply-To or From when `to` is omitted.
3. With `reply_all: true` (valid only with `reply_to_message_id`): To
   defaults to the original's Reply-To or From plus the original To
   recipients, and Cc defaults to the original Cc. The account's own address
   and anything matching its aliases are removed, as are duplicates. An
   explicit `to` or `cc` replaces the corresponding default.
4. Resolve From (see "From address" below).
5. Build MIME with nodemailer's MailComposer: the resolved From, text body,
   a fresh Message-ID, and the marker header `X-Mail-Gateway-Draft: 1`.
6. Open an IMAP connection, APPEND to `drafts_path` with `\Draft`, logout.
7. Return `{account, folder, uid, from}`. Nothing is written to `messages`.
   The Drafts folder is not synced unless the account lists it in
   `extra_folders`.

#### From address

An account owns its `email` plus every entry in `aliases`. An alias is an
exact address or a `*@domain` pattern, which covers a personal domain hosted
on the account (the Fastmail case, where mail arrives at many different
addresses under one domain). Matching is case-insensitive.

From is resolved in this order:

1. An explicit `from`. It must equal the account email or match an alias,
   otherwise 400.
2. On a reply, the address the original was received at: the first address
   that the account owns among the original's To, then Cc, then
   `delivered_to_json`. If the original was sent by the account itself (its
   From is an owned address), that From is reused.
3. The account email.

The reply therefore always goes out from the same address the correspondent
wrote to, including when that address only appears in a delivery header
(Bcc, catch-all, mailing list).

#### Editing a draft

`update_draft` replaces a draft the gateway created earlier. IMAP has no
in-place replace, so the new version is appended before the old one is
removed; a failure in between leaves both copies rather than none.

1. Resolve the account as in step 1 above. Open an IMAP connection, select
   `drafts_path`, fetch the message with the given uid using PEEK. Missing
   uid: 404.
2. Require the `\Draft` flag and the `X-Mail-Gateway-Draft` header. Anything
   else is a draft the user or another client wrote: 409, untouched.
3. Parse it. Each supplied field replaces the stored value; omitted fields
   are kept. `In-Reply-To`, `References`, and the Message-ID are carried over.
   A supplied `from` is validated as above.
4. APPEND the rebuilt message with `\Draft`.
5. Remove the old copy with `UID STORE +FLAGS \Deleted` and `UID EXPUNGE` on
   that single uid. A server without UIDPLUS cannot expunge one message
   safely, so the operation is refused up front with 409 `not_supported` and
   nothing is changed.
6. Return `{account, folder, uid, from}` with the new uid. The old uid is no
   longer valid.

This is the only place the gateway deletes anything, and it can only ever
remove a message that carries the gateway's own marker header.

### 7.5 MCP

- `@modelcontextprotocol/sdk`, Streamable HTTP transport, stateless, JSON
  responses, mounted at `/mcp`.
- Tools: `get_unread`, `get_recent`, `search`, `get_message`, `get_thread`,
  `create_draft`, `update_draft`, `sync`, `get_status`. Not `get_audit`.
- Tool descriptions are written for the model. `sync` says data refreshes
  automatically every 10 minutes and the tool should only be called when the
  user explicitly needs mail newer than `synced_at`. `create_draft` says
  nothing is ever sent, the draft appears in the account's Drafts folder for
  the user to review, and on a reply From is chosen automatically to match
  the address the original was received at, so `from` should be left out
  unless the user asks for a specific address. `update_draft` says it only
  works on drafts created through this gateway and returns a new uid.
  `get_status` is described as the way to learn account names, addresses,
  aliases, custom folder paths, and Gmail label names.

### 7.6 Agent skill file

`SKILL.md` sits alongside the README for clients that talk REST and never see
the MCP tool descriptions (Codex, scripts, the bearer-token-only app). It
covers the base URL and bearer auth, every `/v1` route with its inputs, the
conventions in 7.1, the same usage guidance as the MCP tool descriptions
(when to call `sync`, that drafts are never sent, how From is chosen), and one
curl example per operation. It is written by hand and kept in step with the
zod schemas; a unit test fails if an operation or input name in the schemas
is missing from the file.

## 8. Auth

### 8.1 Single issuer: Authentik

Every request to `/v1/*` and `/mcp` must carry `Authorization: Bearer <JWT>`
issued by Authentik. There are no static tokens.

Middleware:

1. Parse the JWT. Verify signature against Authentik's JWKS (discovered from
   `AUTHENTIK_ISSUER` OIDC metadata, cached), issuer, expiry, and that
   `azp`/`client_id` is in `OAUTH_ALLOWED_CLIENT_IDS`.
2. Call Authentik's token introspection endpoint, authenticated with
   `AUTHENTIK_INTROSPECTION_CLIENT_ID` / `_SECRET`. Cache the result per token
   hash for 5 minutes. If `active` is false, 401. If Authentik is unreachable
   and the cache has no entry, 503 (fail closed).
3. Attach `{subject, client_id, client_name}` to the request for the audit
   log.

On 401 the response carries
`WWW-Authenticate: Bearer resource_metadata="<PUBLIC_URL>/.well-known/oauth-protected-resource"`.

### 8.2 Discovery

`GET /.well-known/oauth-protected-resource` (and the `/mcp` suffixed variant)
returns:

```
{ resource: PUBLIC_URL, authorization_servers: [AUTHENTIK_ISSUER],
  bearer_methods_supported: ["header"] }
```

Clients then read Authentik's own metadata. Authentik has no dynamic client
registration, so the user creates one OAuth provider + application per client
in Authentik and pastes client id and secret into the connector settings.
claude.ai and ChatGPT both support manual client credentials. Redirect URIs
per client are documented in the README (claude.ai:
`https://claude.ai/api/mcp/auth_callback`; ChatGPT:
`https://chatgpt.com/connector_platform_oauth_redirect`; verify current
values at implementation time).

### 8.3 Long-lived tokens for non-OAuth apps

A second Authentik provider (for example `mail-gateway-tokens`) with a long
access token validity. The user mints a JWT with one client credentials call,
authenticating with an Authentik app password so the token is issued as the
user. The JWT is pasted into the app as a static bearer. Revoking it in
Authentik takes effect within the 5-minute introspection cache window.

### 8.4 Authentik access policy

The user binds a policy so only their own account may authorize each
application. The gateway trusts any active token from an allowed client id.

## 9. Audit log

A single wrapper around all ten service functions writes one `audit_log` row
per call, whichever transport it came from, and emits the same record as a
pino structured log line.

- Recorded: timestamp, request id, client ip (from `X-Forwarded-For` set by
  Caddy), subject, client id, client name, transport, operation, redacted
  params, result summary, error code, duration.
- Redaction: params keep query, accounts, folders, labels, limit, offset,
  order, ids, from/to/cc/subject/after/before and the boolean filters, and
  for drafts the account, uid, from, recipients, subject, and body length.
  The draft text is never stored.
- Result summary: row count and returned message ids for reads; account,
  folder, uid for drafts; request id for sync; error code on failure.
- Auth failures are logged with the parsed client id when available.
- Retrieval: `GET /v1/audit` with `client`, `operation`, `since`, `limit`;
  `cli.js audit` in the worker image. Not an MCP tool.
- Retention: worker prunes rows older than `AUDIT_RETENTION_DAYS` (default
  365) nightly.

## 10. Configuration

### 10.1 `accounts.yaml` (mounted read-only)

```yaml
accounts:
  - name: personal-gmail
    provider: gmail
    email: me@gmail.com
    password_env: GMAIL_PERSONAL_PASSWORD
  - name: work-outlook
    provider: outlook
    email: me@outlook.com
    auth: oauth
  - name: qq
    provider: qq
    email: 12345@qq.com
    password_env: QQ_PASSWORD
    enabled: true
  - name: fastmail
    provider: fastmail
    email: me@fastmail.com
    password_env: FASTMAIL_PASSWORD
    aliases:                # optional; addresses this account also receives at
      - alan@example.com
      - "*@example.com"     # catch-all for a domain hosted on this account
    extra_folders:          # optional; IMAP paths synced with role = custom
      - Trash
      - Spam
  - name: custom
    provider: generic
    email: me@example.org
    host: imap.example.org
    port: 993
    user: me            # optional, defaults to email
    password_env: CUSTOM_PASSWORD
```

### 10.2 Environment (`.env`, loaded by compose into both containers)

```
DB_KEY                              SQLite encryption key
PUBLIC_URL                          https://mail.example.com
AUTHENTIK_ISSUER                    https://auth.example.com/application/o/mail-gateway/
OAUTH_ALLOWED_CLIENT_IDS            comma-separated
AUTHENTIK_INTROSPECTION_CLIENT_ID
AUTHENTIK_INTROSPECTION_CLIENT_SECRET
MS_CLIENT_ID                        Azure app id, only if an outlook account exists
SYNC_INTERVAL_MINUTES               default 10
AUDIT_RETENTION_DAYS                default 365
LOG_LEVEL                           default info
<one variable per password_env referenced in accounts.yaml>
```

Both files are validated with zod at startup; the process exits with a clear
message on any problem, including a `password_env` that names an unset
variable.

## 11. Deployment

- Multi-stage Dockerfile on `node:24-slim`: build TypeScript, install
  production dependencies, copy `dist`.
- `docker-compose.yml`: services `api` (entrypoint `dist/api.js`) and `worker`
  (entrypoint `dist/worker.js`), both with `env_file: .env`, `accounts.yaml`
  mounted read-only, volume `data:/data`, `restart: unless-stopped`. `api`
  joins the existing external Caddy network and publishes no host port. `api`
  has a healthcheck on `/healthz`.
- Caddyfile snippet in the README: `reverse_proxy api:8080` on the subdomain.
- Migrations: numbered SQL files in `migrations/`, applied at startup by either
  process under `BEGIN IMMEDIATE`; `meta.schema_version` records the current
  version.
- CLI (worker image): `auth <account>`, `status`, `resync <account>
  [folder]` (resets cursors, restarts backfill), `audit [--client] [--since]`.

## 12. Repository layout

```
src/
  config/        accounts.yaml + env loading and zod schemas
  db/            open (with key), migrations, typed queries, CJK transform
  imap/          provider profiles, connection factory, outlook oauth
  sync/          folder discovery, fetch pipeline, incremental, backfill,
                 reconciliation, sync request handling, scheduler
  service/       the ten functions + audit wrapper
  http/          Hono app, auth middleware, well-known, REST routes
  mcp/           tool registration over the service module
  cli/           auth, status, resync, audit
  api.ts         api entrypoint
  worker.ts      worker entrypoint
migrations/
test/
  unit/
  fixtures/      .eml files
  integration/   docker-compose.dovecot.yml + tests
Dockerfile
docker-compose.yml
accounts.example.yaml
.env.example
README.md
SKILL.md         agent instructions for REST-only clients (7.6)
```

## 13. Testing

vitest throughout.

### 13.1 Unit (fast, every change)

- Service layer against an in-memory encrypted SQLite seeded with fixtures:
  unread, recent with folder and multi-account filters, search ranking, search
  input that would break FTS syntax, Chinese substring search (1, 2, and 4
  character queries), thread walking across folders, id mapping and 404s,
  `synced_at` computation. Each search filter alone and combined (to, cc,
  subject, has_attachment, flagged, labels), filter-only search without a
  query, `to` matching a delivered-to address, `offset` paging and
  `order: asc`, `include_body` truncation, custom folders excluded by default
  and returned when named.
- Sync logic against a fake IMAP client interface: folder role mapping
  including Gmail `\All`, `extra_folders` mapped to custom, Gmail label
  folders in `extra_folders` rejected, a removed extra folder purged,
  first-sight cursor initialization, incremental
  advance, UIDVALIDITY reset, CONDSTORE path versus 30-day fallback, VANISHED
  handling, chunked backfill resume after a simulated crash, status rule
  computation, sync request lifecycle.
- MIME parsing from fixture `.eml` files: GBK-encoded subject, HTML-only
  body, multipart with attachments (metadata extracted, nothing downloaded),
  oversized body capped, HTML links rendered as `[text](url)`, delivered-to
  headers extracted.
- Draft building: reply headers, `Re:` prefix idempotence, default recipient
  from Reply-To. Reply-all recipients with own address and aliases removed.
  From resolution: explicit alias accepted, unowned address rejected,
  `*@domain` pattern matched, reply picks the received-at address from To,
  from Cc, and from delivered-to only, fallback to the account email.
- Draft editing against the fake IMAP client: omitted fields kept, threading
  headers carried over, append happens before delete, draft without the
  marker header refused, missing uid, server without UIDPLUS refused.
- `SKILL.md` names every operation and input in the zod schemas.
- Auth middleware with a locally generated JWKS: valid token, expired,
  wrong issuer, disallowed client id, introspection inactive, introspection
  unreachable with and without cache.
- Audit wrapper: one row per call, redaction, error path.
- Config validation failures.

### 13.2 Integration (on demand)

A Dovecot container in `test/integration/docker-compose.dovecot.yml`. Append
fixture messages, run a real worker pass, assert rows and FTS results, append
a draft through the service and assert it appears in Drafts with `\Draft`,
then edit it and assert exactly one copy remains with the new content. Sync
an extra folder and assert it is searchable only when named.

### 13.3 Manual pre-deploy checklist (README)

- PEEK verified on a real Gmail account: syncing does not mark mail read.
- A reply drafted to a message received at a personal-domain address on
  Fastmail shows that address as From in the Fastmail Drafts folder.
- Outlook device-code login completes and a subsequent sync succeeds.
- claude.ai connector completes the Authentik handshake and lists tools.
- A long-lived token works over REST and stops working within 5 minutes of
  revocation in Authentik.
- `/healthz` flips to 503 when the worker is stopped.

## 14. Items to verify during planning

These are not open design questions; they are library facts to confirm before
the implementation plan fixes package versions.

- `better-sqlite3-multiple-ciphers` ships prebuilt binaries for Node 24; fall
  back to building from source in the Dockerfile if not.
- imapflow API for `changedSince` fetches, QRESYNC, `X-GM-LABELS`, and
  per-part download by body-structure part id.
- The MCP SDK transport to use with Hono on `@hono/node-server` (Node
  req/res transport versus the web-standard transport).
- Authentik client credentials grant with app-password authentication, and the
  introspection endpoint's exact URL and client auth method.
- Whether 126 and QQ IMAP advertise CONDSTORE. If not, they use the fallback
  path, which is already designed.
- Which delivery header each provider writes (`Delivered-To`,
  `X-Delivered-To`, `X-Original-To`), Fastmail in particular, and that
  imapflow can fetch named header fields alongside the envelope.
- Which providers advertise UIDPLUS, and imapflow's call for a single-uid
  expunge. Providers without it cannot use `update_draft`.
- The `html-to-text` selector options that produce `[text](url)` output.
