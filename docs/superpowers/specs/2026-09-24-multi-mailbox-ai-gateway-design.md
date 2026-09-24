# Multi-Mailbox AI Gateway — Design Spec

Date: 2026-09-24
Status: approved in brainstorming, pending written review

## 1. Purpose

A self-hosted service that syncs ten personal mailboxes over IMAP into one
encrypted SQLite database and exposes them to AI agents and scripts through a
REST API and an MCP server. Single user. A few calls per day. Agents can read,
search, and create drafts. Nothing is ever sent, deleted, moved, or flagged.

## 2. Decisions made during brainstorming

| Topic | Decision |
|---|---|
| Providers | Gmail / Google Workspace, Outlook.com (OAuth2), Yahoo, Fastmail, iCloud, 126 (NetEase), QQ, other plain IMAP |
| ProtonMail | Out of scope entirely. No Bridge container. |
| Outlook auth | OAuth2 device-code flow built into the worker CLI; tokens stored in the DB |
| Runtime | Node 24, TypeScript |
| Deployment | Docker Compose on a VPS, behind the user's existing Caddy on a dedicated subdomain |
| Architecture | Two containers from one image: `api` and `worker`, sharing one SQLite volume |
| Database | SQLite via `better-sqlite3-multiple-ciphers`, page-encrypted with a key from env, WAL mode, FTS5 |
| Folders synced | INBOX, Sent, Archive for every account. Gmail: All Mail only, with labels. Full history, no time cutoff. |
| Sync trigger | Timer every 10 minutes plus an explicit `sync` operation that can target all, one, or several accounts |
| Recent bound | Total limit (default 50) with an optional multi-account filter |
| Read operations | unread, recent, search, get_message, get_thread, plus status |
| Message ids | Opaque `msg_<rowid>` |
| Drafts | Plain text only, reply threading via stored headers, no HTML, no attachments |
| Attachments | Metadata only (name, type, size). Never downloaded. |
| Chinese search | Per-character CJK tokenization with phrase queries. No trigram index. |
| Auth | Authentik is the only token issuer. OAuth 2.1 for claude.ai / ChatGPT; long-lived Authentik JWTs for apps without OAuth. No static tokens. |
| Revocation | Authentik token introspection, cached 5 minutes, fail closed |
| Audit log | Every service call recorded in the DB and to stdout, queryable over REST and CLI |
| Clients | Claude Code, claude.ai / Claude Desktop connector, chatgpt.com connector, Codex, OpenClaw, Hermes, scripts over REST, at least one app that only supports REST with a bearer token |

## 3. Out of scope

Sending mail, deleting, moving, marking spam, changing flags from the API,
attachment download, HTML drafts, calendar, contacts, multi-user, ProtonMail,
a web UI.

## 4. Components

One Docker image, two containers, one named volume.

### 4.1 `api` container

- Hono on Node 24.
- Serves REST under `/v1/*`, MCP at `/mcp` (Streamable HTTP, stateless, JSON
  responses), OAuth protected-resource metadata at
  `/.well-known/oauth-protected-resource`, and `GET /healthz`.
- Opens the DB read-mostly. Writes exactly two tables: `sync_requests` and
  `audit_log`.
- Drafts are the one operation where the API itself opens a short-lived IMAP
  connection to APPEND. Routing this through the worker would add a second IPC
  path for one operation.

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
  role TEXT                   -- inbox | sent | archive | all
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
`accounts.drafts_path`. Gmail maps only `\All` → all. Unknown or missing
folders are skipped. New folder rows get `highest_uid = UIDNEXT - 1` and
`backfill_low_uid = UIDNEXT`, so new mail is caught immediately while backfill
walks backwards.

### 6.3 Fetch pipeline

For each UID: fetch flags, envelope, INTERNALDATE, size, body structure, and
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

One service module with nine functions. REST routes and MCP tools are thin
wrappers. Each function has one zod input schema used for REST validation and
as the MCP tool input schema.

### 7.1 Conventions

- `accounts`: optional array of account names; omitted means all enabled.
- `folders`: optional array from `inbox | sent | archive`.
- Every read response includes `synced_at`: the oldest `last_success_at`
  among covered accounts.
- Summary shape: `id, account, folder, from {name, email}, subject, date,
  unread, has_attachments, snippet`.
- Errors: `{error: {code, message}}`. 400 validation, 401 auth, 404 unknown
  id or account, 409 draft to a disabled or errored account, 502 IMAP append
  failure, 503 from `/healthz` and from auth when Authentik is unreachable.
- Reads never fail because the worker is down; they return an older
  `synced_at`.

### 7.2 Operations

| Function | REST | Input | Output |
|---|---|---|---|
| `get_unread` | `GET /v1/unread` | accounts, limit (50, max 200) | summaries, INBOX only, newest first |
| `get_recent` | `GET /v1/recent` | accounts, folders (default inbox), limit (50, max 200) | summaries newest first |
| `search` | `GET /v1/search` | query, accounts, folders (default all), from, after, before, unread_only, limit (25, max 100) | summaries ranked by bm25 |
| `get_message` | `GET /v1/messages/:id` | id | full headers, text_body, attachments metadata, thread_root |
| `get_thread` | `GET /v1/messages/:id/thread` | id | up to 30 messages, oldest first, bodies capped at 8 KB with `truncated` |
| `create_draft` | `POST /v1/drafts` | account, to[], cc[], bcc[], subject, text, reply_to_message_id | account, folder, uid |
| `sync` | `POST /v1/sync` | accounts, wait (true, ≤ 60 s) | per-account counts, or queued status |
| `get_status` | `GET /v1/status` | none | see 7.3 |
| `get_audit` | `GET /v1/audit` (REST only) | client, operation, since, limit | audit rows |
| health | `GET /healthz` (no auth) | none | 200 / 503 |

Search query handling: terms are split on whitespace, each term is
double-quoted before entering the FTS MATCH expression so user input cannot
cause an FTS syntax error, CJK runs are expanded per 5.2, and terms are ANDed.
`from` filters on `from_addr` or `from_name` with LIKE. `after` / `before` are
ISO dates.

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
    name, email, provider, enabled,
    status,                                  // ok | backfilling | error | never_synced
    last_success_at, last_error, last_error_at,
    message_count, unread_count,
    folders: [ { path, role, indexed, backfill_done, estimated_remaining } ]
  } ]
}
```

`/healthz` returns 503 if the worker heartbeat is older than 15 minutes or the
database cannot be opened.

### 7.4 Draft semantics

1. Resolve the account, its IMAP settings, and cached `drafts_path`. If the
   account is disabled or in `error`, return 409.
2. If `reply_to_message_id` is given: load the original, set `In-Reply-To` to
   its Message-ID, set `References` to its references plus its Message-ID,
   prefix subject with `Re: ` if not already present, and default `to` to the
   original's Reply-To or From when `to` is omitted.
3. Build MIME with nodemailer's MailComposer: From = account email, text body,
   a fresh Message-ID.
4. Open an IMAP connection, APPEND to `drafts_path` with `\Draft`, logout.
5. Return `{account, folder, uid}`. Nothing is written to `messages`. The
   Drafts folder is never synced.

### 7.5 MCP

- `@modelcontextprotocol/sdk`, Streamable HTTP transport, stateless, JSON
  responses, mounted at `/mcp`.
- Tools: `get_unread`, `get_recent`, `search`, `get_message`, `get_thread`,
  `create_draft`, `sync`, `get_status`. Not `get_audit`.
- Tool descriptions are written for the model. `sync` says data refreshes
  automatically every 10 minutes and the tool should only be called when the
  user explicitly needs mail newer than `synced_at`. `create_draft` says
  nothing is ever sent and the draft appears in the account's Drafts folder for
  the user to review. `get_status` is described as the way to learn account
  names and addresses.

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

A single wrapper around all nine service functions writes one `audit_log` row
per call, whichever transport it came from, and emits the same record as a
pino structured log line.

- Recorded: timestamp, request id, client ip (from `X-Forwarded-For` set by
  Caddy), subject, client id, client name, transport, operation, redacted
  params, result summary, error code, duration.
- Redaction: params keep query, accounts, folders, limit, ids, from/after/
  before, and for drafts the account, recipients, subject, and body length.
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
  service/       the nine functions + audit wrapper
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
```

## 13. Testing

vitest throughout.

### 13.1 Unit (fast, every change)

- Service layer against an in-memory encrypted SQLite seeded with fixtures:
  unread, recent with folder and multi-account filters, search ranking, search
  input that would break FTS syntax, Chinese substring search (1, 2, and 4
  character queries), thread walking across folders, id mapping and 404s,
  `synced_at` computation.
- Sync logic against a fake IMAP client interface: folder role mapping
  including Gmail `\All`, first-sight cursor initialization, incremental
  advance, UIDVALIDITY reset, CONDSTORE path versus 30-day fallback, VANISHED
  handling, chunked backfill resume after a simulated crash, status rule
  computation, sync request lifecycle.
- MIME parsing from fixture `.eml` files: GBK-encoded subject, HTML-only
  body, multipart with attachments (metadata extracted, nothing downloaded),
  oversized body capped.
- Draft building: reply headers, `Re:` prefix idempotence, default recipient
  from Reply-To.
- Auth middleware with a locally generated JWKS: valid token, expired,
  wrong issuer, disallowed client id, introspection inactive, introspection
  unreachable with and without cache.
- Audit wrapper: one row per call, redaction, error path.
- Config validation failures.

### 13.2 Integration (on demand)

A Dovecot container in `test/integration/docker-compose.dovecot.yml`. Append
fixture messages, run a real worker pass, assert rows and FTS results, append
a draft through the service and assert it appears in Drafts with `\Draft`.

### 13.3 Manual pre-deploy checklist (README)

- PEEK verified on a real Gmail account: syncing does not mark mail read.
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
