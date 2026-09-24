# Self-Hosted Multi-Mailbox Gateway for AI Agents

For this use case, building a small self-hosted tool is very reasonable.

With only 10 mailboxes, light usage, and a narrow feature set, this can be much smaller than products such as Nylas or EmailEngine. The core is essentially:

- IMAP aggregation
- A small local cache/index
- REST API endpoints
- MCP tools
- Draft creation through IMAP
- Proton Mail handled separately through Proton Bridge

## High-Level Architecture

```text
                         ┌─ Gmail #1
                         ├─ Gmail #2
                         ├─ Outlook
                         ├─ Yahoo
                         ├─ Fastmail
                         ├─ ...
                         └─ Proton Bridge
                              │
                              ▼
                    ┌───────────────────┐
                    │   Mail Gateway    │
                    │                   │
                    │ IMAP connectors   │
                    │ SQLite cache      │
                    │ Search index      │
                    └─────────┬─────────┘
                              │
                   ┌──────────┴──────────┐
                   │                     │
                 REST                   MCP
                   │                     │
              your scripts       Claude / Codex /
                                 other agents
```

I would not build a full synchronizing mail server. That would add a lot of unnecessary complexity.

## Core Operations

The basic REST API could expose:

```text
GET /unread
GET /recent?limit=100
GET /search?q=...
POST /drafts
```

Equivalent MCP tools could be:

```text
get_unread_emails()
get_recent_emails(limit=100)
search_emails(query, ...)
create_draft(account, to, subject, body, ...)
```

For Node.js / TypeScript, ImapFlow is a good fit because it supports modern IMAP, async/await, streaming message retrieval, mailbox locking, TypeScript types, and provider-specific extensions.

The MCP TypeScript SDK can sit inside the same Fastify or Express-style service, so REST and MCP can share the same backend functions.

## Add SQLite

Even with light usage, SQLite is worth adding.

Not because performance is a major concern, but because it makes cross-mailbox search much simpler and faster.

A simple schema could look like this:

```text
accounts
--------
id
name
email
provider

messages
--------
account_id
folder
uid
message_id
from
to
cc
subject
date
text_body
html_body
flags
updated_at
```

Then use SQLite FTS5 for search:

```text
message_search
--------------
subject
from
to
text_body
```

The overall data flow becomes:

```text
IMAP
  ↓
incremental sync
  ↓
SQLite + FTS
  ↓
API / MCP
```

Searching all 10 IMAP servers live would work, but it would be slower and would make the response time depend on the slowest provider.

With SQLite FTS, a search such as:

```text
search_emails("ServiceNow interview")
```

becomes one local database query.

## Synchronization Strategy

There is no need to maintain permanent IMAP IDLE connections for all 10 accounts.

A lightweight approach would be:

```text
for each account:
    connect
    select mailbox
    check UIDVALIDITY
    fetch messages with UID > last_seen_uid
    update flags for recent known messages
    disconnect
```

This could run:

- Every 5 to 10 minutes
- Immediately before an API request
- Or both

For example:

```text
/unread
   ↓
sync accounts
   ↓
query SQLite
   ↓
return results
```

For a few requests per day, this workload is very small.

## Recent Mail Design

Instead of always returning 100 messages from every mailbox, I would default to 100 messages total:

```text
get_recent_emails({
    limit: 100,
    accounts: "all"
})
```

Optionally support:

```text
get_recent_emails({
    limitPerAccount: 100
})
```

Returning 100 messages from each of 10 accounts could produce up to 1,000 emails, which is unnecessarily expensive for an AI agent's context window.

A better pattern is to return lightweight metadata first:

```json
{
  "id": "fastmail:INBOX:82941",
  "account": "personal-fastmail",
  "from": "alice@example.com",
  "subject": "Dinner",
  "date": "...",
  "unread": true,
  "preview": "Are you free next..."
}
```

Then expose:

```text
get_email(id)
```

to retrieve the full body only when needed.

This gives agents a workflow like:

```text
search
   ↓
20 lightweight results
   ↓
agent selects 3
   ↓
get_email() × 3
```

instead of dumping large amounts of HTML email into the context window.

## Drafting Through IMAP

You do not need SMTP if the agent only creates drafts.

The service can build an RFC 822 / MIME message and use IMAP `APPEND` to place it into the account's Drafts mailbox with the `\Draft` flag.

Conceptually:

```text
Agent
  ↓
create_draft(...)
  ↓
build MIME message
  ↓
IMAP APPEND
  ↓
Drafts folder
```

Do not hardcode the folder name as:

```text
Drafts
```

Different providers and language settings may use different names.

Instead, inspect the mailbox metadata and locate the folder advertising the IMAP special-use attribute:

```text
\Drafts
```

Then cache that folder name per account.

A draft API could accept:

```json
{
  "account": "personal-gmail",
  "to": ["foo@example.com"],
  "cc": [],
  "subject": "Re: Something",
  "text": "Hi...",
  "inReplyTo": "..."
}
```

For reply drafts, preserve:

```text
Message-ID
In-Reply-To
References
```

so the message remains part of the correct conversation when manually sent later.

## Proton Mail

Proton Mail is the main exception.

Normal providers can connect directly:

```text
container → imap.gmail.com
container → outlook.office365.com
container → imap.fastmail.com
...
```

For Proton:

```text
Proton
   ↕ encrypted
Proton Bridge
   ↕ local IMAP
your gateway
```

Proton Bridge exposes a local IMAP/SMTP interface that normal mail clients can use.

The main complication is Docker networking because Proton Bridge typically exposes its IMAP interface locally.

It is probably best to get the other nine accounts working first and solve Proton separately afterward.

A clean connector abstraction might look like:

```ts
interface MailAccount {
  search(...)
  getUnread(...)
  getRecent(...)
  getMessage(...)
  createDraft(...)
}
```

Then Proton can eventually become another IMAP implementation pointed at Bridge.

## Recommended API Surface

Rather than exposing only four operations, I would use six:

```text
get_unread_emails()
get_recent_emails()
search_emails()
get_email()
get_thread()
create_draft()
```

The extra two are especially useful for AI agents.

For example:

```text
search_emails({
  query: "recruiter ServiceNow"
})
```

could return:

```text
1. Sep 22 - Anisha - Technical interview availability
2. Sep 18 - Recruiter - ServiceNow opportunity
3. Sep 15 - ...
```

Then the agent can call:

```text
get_thread(messageId)
```

instead of loading every full message body during search.

## Normalize Message IDs

Do not expose raw IMAP UIDs as your public IDs.

An IMAP UID is only unique within a mailbox and can become invalid when `UIDVALIDITY` changes.

Instead, expose your own stable IDs:

```text
msg_01K...
```

and internally map them to:

```text
account_id
folder
uidvalidity
uid
```

That keeps agents independent from IMAP internals.

## Security

This service effectively becomes the master key to 10 email accounts, so it should not be exposed directly to the public internet.

Avoid:

```text
0.0.0.0:3000
        ↓
public internet
```

Prefer:

```text
                  Tailscale / private network
                           │
                           ▼
Agent ───────────────── Mail Gateway
                           │
                       IMAP accounts
```

or put it behind proper TLS and authentication.

The AI-facing capability set should also be deliberately limited.

Allow:

```text
READ
SEARCH
CREATE_DRAFT
```

Do not expose:

```text
SEND
DELETE
MOVE
MARK_SPAM
CHANGE_PASSWORD
```

That gives you an important safety property:

Even if an email contains prompt-injection content, the most dangerous email-related write action available to the agent is creating a draft.

## Suggested Stack

A practical implementation could use:

```text
TypeScript / Node 22+
│
├── ImapFlow
│      IMAP
│
├── mailparser
│      MIME → text/html/attachments
│
├── Nodemailer MailComposer
│      construct MIME drafts
│
├── SQLite
│
├── SQLite FTS5
│      cross-account search
│
├── Fastify
│      REST API
│
└── MCP TypeScript SDK
       MCP
```

The project could stay very small:

```text
Dockerfile
docker-compose.yml

/data/mail.db

/config/accounts.yaml
```

Store passwords, app passwords, or OAuth tokens using Docker secrets or environment variables rather than directly inside the YAML configuration file.

## Shared Backend for REST and MCP

REST and MCP should call the same internal service functions:

```text
                         ┌── REST route
searchMessages() ────────┤
                         └── MCP tool
```

That means adding MCP does not require maintaining a second email backend.

## Overall Recommendation

For:

- 10 personal mailboxes
- A few accesses per day
- Read-only access
- Search
- Draft creation
- No automated sending

a self-hosted service is a very good fit.

You avoid most of the complexity of building an email platform:

```text
❌ SMTP delivery
❌ bounce handling
❌ webhooks
❌ calendar
❌ contacts
❌ tenant onboarding
❌ billing
❌ account provisioning
❌ high availability
❌ thousands of concurrent IMAP connections
❌ multi-user authorization

✅ fetch
✅ index
✅ search
✅ draft
```

For this specific scope, a small Dockerized service with IMAP + SQLite + REST + MCP is likely simpler, cheaper, and easier to control than introducing a commercial aggregation platform.
