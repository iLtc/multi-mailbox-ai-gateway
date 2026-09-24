# Mail Aggregator — Design Notes

A small self-hosted tool that syncs 10 mailboxes (9 IMAP + ProtonMail via Bridge) and exposes them to AI agents as both a REST API and an MCP server. Single user, light usage (a few calls per day).

## Requirements

1. One endpoint for all unread emails
2. One endpoint for the ~100 latest emails per provider
3. One endpoint to search across all mailboxes
4. One endpoint to draft an email (save to Drafts, never send)

Estimated size: 400–600 lines of TypeScript.

## Core decision: sync into SQLite, serve from SQLite

Don't query 10 IMAP servers live. All three read endpoints want the same thing — a small, fresh, cross-account index. Implementing search as 10 live IMAP `SEARCH` calls has three problems:

- Slow: 10 TLS handshakes + logins; Gmail alone can take several seconds
- IMAP `SEARCH` is a dumb substring match with no ranking
- Providers rate-limit logins, and a retrying agent will trip them

Instead:

- A **sync loop** (every 10–15 min, plus `POST /sync` for on-demand) opens each account, fetches new UIDs since the last seen UID, stores headers + plain-text body + flags in SQLite with an **FTS5** table, then closes the connection.
- **No persistent connections.** At a few calls per day they'd just die and need reconnect logic anyway.
- `unread`, `latest`, and `search` become SQL queries that return in milliseconds. Staleness of ≤15 min is fine; the agent can call `sync` first if it needs fresh data.
- Message identity: `(account, folder, uidvalidity, uid)`. If `UIDVALIDITY` changes, resync that folder.

## Libraries

| Purpose | Library | Notes |
|---|---|---|
| IMAP | `imapflow` | By the EmailEngine author. Handles SPECIAL-USE folder detection, XOAUTH2, and fetches with `BODY.PEEK` so reads don't mark messages seen (verify on Gmail — accidentally marking mail read is the classic bug here) |
| Parsing | `mailparser` | Raw RFC822 → text |
| Draft MIME | `nodemailer` (`MailComposer`) | Builds the draft message |
| MCP | `@modelcontextprotocol/sdk` | Streamable HTTP transport, stateless mode |
| REST | Hono or Fastify | Thin routes over the same service layer |

REST routes and MCP tools should both be thin wrappers over one shared service module with the same four (five) functions.

## Drafts

The only IMAP write, and simpler than it looks:

1. Build the MIME with `MailComposer`
2. `client.append(draftsFolder, mime, ['\\Draft'])`

Find the Drafts folder per account via `client.list()` and the `specialUse === '\\Drafts'` attribute rather than hardcoding names — Gmail uses `[Gmail]/Drafts`, Outlook `Drafts`, Proton via Bridge `Drafts`, and localized accounts vary.

Parameters: `account` (the agent must pick which identity the draft belongs to), `to`, `subject`, `body`, and optionally `in_reply_to` so replies thread correctly.

Keep **send** out of the codebase entirely. The worst an agent can do is leave a bad draft.

## Provider gotchas

- **Outlook.com** is OAuth2-only for IMAP. Needs an Azure app registration and a refresh-token loop. `imapflow` accepts `auth: { user, accessToken }`.
- **Gmail, Yahoo, Fastmail, iCloud** work with app passwords.
- **Gmail:** sync `INBOX` rather than `[Gmail]/All Mail` unless archived mail is wanted. "Unread" = no `\Seen` flag.
- **ProtonMail:** run the Bridge container in the same compose file and talk to it over the internal Docker network. Bridge uses a self-signed cert — pin it, or set `tls.rejectUnauthorized: false` for that one account only.

## Shape responses for an LLM, not a UI

- `unread` / `latest` / `search` return **compact summaries**: account, from, subject, date, ~200-char snippet, id.
- Add a fifth tool, **`get_message(id)`**, for the full body.
- Otherwise "100 latest per provider" is 1,000 message bodies and blows the agent's context on every call.
- Cap search results at ~50.

## Auth and exposure

- MCP clients can't do a browser login, so **Authentik forward-auth won't work** on this path.
- Use a **static bearer token** checked in middleware (one token per agent if individual revocation is wanted).
- Put it behind Caddy on a dedicated subdomain, or keep it **Tailscale-only**.
- Store the 10 app passwords as Docker secrets or an env file — never in the SQLite DB.

## Deployment

Three pieces in `docker-compose.yml`:

- the app
- the Proton Bridge container
- a volume for SQLite

No Redis, no Postgres.
