# Gap Analysis: Design Spec vs. Spark CLI

Date: 2026-09-26
Compared against: `docs/superpowers/specs/2026-09-24-multi-mailbox-ai-gateway-design.md`
Spark CLI version: 1.3.1 (thin client over IPC to Spark Desktop; 9 accounts visible, all set to read-only access, so drafts and actions were not exercised)

How to use this file: tick `[x]` on anything you want added to the spec, and
leave a note under it if the shape should differ from what is described.

Legend: **Cost** is a rough implementation estimate given the current design.
S = small (existing columns / config), M = medium (new sync or IMAP work),
L = large (new subsystem or a change to a design decision).

---

## 1. Scope of mail exposed

- [ ] **All folders, not just inbox / sent / archive** (Cost: M)
  Spark lists and searches Trash, Spam, Drafts, Starred, custom IMAP folders
  (the Chinese folders on 126, `存档` on Aliyun), and every Gmail label as a
  first-class folder. The spec syncs only inbox, sent, archive, and the
  `folders` filter accepts only those three roles.
  Possible shape: an opt-in `extra_folders` list per account in `accounts.yaml`,
  each synced as a `folders` row with `role = custom`.

- [ ] **Gmail label filter** (Cost: S)
  `labels_json` is already stored per message but no API input can filter on
  it. One Gmail label in your account holds 1,189 messages and is unreachable.
  Possible shape: `labels: string[]` on `search` and `get_recent`, Gmail only.

- [ ] **Account aliases** (Cost: S)
  Spark shows aliases (the NYU account has one). The spec has no alias
  concept. Aliases matter for a draft's From address and for "sent to me"
  filtering.
  Possible shape: `aliases: string[]` per account in `accounts.yaml`,
  surfaced in `get_status`, selectable as `from` on `create_draft`.

- [ ] **Aliyun provider profile** (Cost: S)
  Spark holds an Aliyun account that the spec's provider table does not name.
  The generic profile covers it, but a named row (`imap.aliyun.com:993`) keeps
  host and credentials from being forgotten.

## 2. Filters and listing

- [ ] **Extra search filters: `to`, `cc`, `subject`, `has_attachment`, `flagged`** (Cost: S)
  Spark's Gmail-style grammar has `to:`, `cc:`, `subject:`, `has:attachment`,
  `filename:`, `is:starred`, `is:read`, `is:unreplied`. The spec's search takes
  only query, from, after, before, unread_only. Columns for to, cc, flagged,
  and has_attachments already exist. `filename` would need a query against
  `attachments_json`. `is:unreplied` would need a Sent-folder thread join.

- [ ] **Relative dates (`newer_than:7d`, `older_than:1m`)** (Cost: S)
  Spec accepts ISO dates only. Agents can compute ISO dates themselves, so
  this is a convenience rather than a capability gap.

- [ ] **Pagination: offset or cursor, plus sort order** (Cost: S)
  Spark paginates with page, page size, and ascending or descending order.
  The spec has only `limit`, so an agent cannot walk past the first page.
  Possible shape: `offset` on `search`, `get_recent`, `get_unread`; `order:
  asc | desc` on `get_recent`.

- [ ] **`include_body` option on search** (Cost: S)
  Spark's topic search returns the top 20 hits with full bodies in one call.
  The spec returns 200-character snippets, forcing a `get_message` round trip
  per result.
  Possible shape: `include_body: boolean`, bodies capped at 8 KB with
  `truncated`, same as `get_thread`.

- [ ] **Semantic search** (Cost: L)
  Spark supports hybrid keyword plus semantic search when Spark +AI is
  enabled. Not enabled on this machine. Would need an embedding pipeline and
  a vector index; the spec is bm25 only.

## 3. Message identity and navigation

- [ ] **Deep link / web URL per message** (Cost: S for Gmail, none for others)
  Every Spark result carries a deep link that opens the message in the
  client, and its bundled agent skill tells agents to always hand that link to
  the user. The spec's opaque `msg_<rowid>` gives the user no way to jump to
  the message.
  Possible shape: store `X-GM-MSGID` for Gmail and return
  `https://mail.google.com/mail/u/0/#all/<hex>` as `web_url`. Other providers
  have no stable URL scheme.

## 4. Drafts

- [ ] **`reply_all` mode** (Cost: S)
  Spark carries the original To recipients into To and the original CCs into
  CC, minus your own address and aliases. The spec's reply defaults `to` to
  Reply-To or From only.

- [ ] **`forward` mode** (Cost: S)
  Quoted original body in plain text, `Fwd:` subject prefix, `References`
  header. Attachments cannot be forwarded since the spec never downloads them.

- [ ] **Per-account signature** (Cost: S)
  Spark appends the account's default signature automatically and offers
  `--no-signature`. Possible shape: `signature` string per account in
  `accounts.yaml`, `no_signature: boolean` on `create_draft`.

- [ ] **Edit an existing draft** (Cost: M)
  IMAP has no in-place replace: fetch the draft, delete it, APPEND the new
  version. Requires syncing or at least addressing the Drafts folder, which
  the spec currently never touches.

- [ ] **Delete a draft** (Cost: M)
  Same dependency on Drafts-folder access. Conflicts with the spec's "nothing
  is ever deleted" rule unless scoped to drafts the gateway itself created.

- [ ] **Markdown or HTML body** (Cost: M)
  Spark writes markdown and converts to HTML. The spec explicitly excludes
  HTML drafts. Reversing that decision means multipart/alternative composing.

- [ ] **Attachments on drafts** (Cost: M)
  Spark accepts a file path, stdin bytes, or an attachment ID from another
  message. Out of scope in the spec.

- [ ] **Templates with placeholders** (Cost: M)
  Spark has personal and team templates with auto and manual placeholders.
  None exist on this machine.

## 5. Write actions (spec forbids all of these by design)

- [ ] **Archive / move to folder** (Cost: M)
- [ ] **Move to trash** (Cost: M)
- [ ] **Attach / detach Gmail label** (Cost: M)
- [ ] **Mark read / unread** (Cost: M)
- [ ] **Mark spam** (Cost: M)
- [ ] **Star / flag** (Cost: M)
- [ ] **Send a draft, or schedule send** (Cost: L, needs SMTP or provider API)
- [ ] **Unsubscribe** (Cost: L, needs List-Unsubscribe header handling)
  Spark-only concepts with no IMAP equivalent: pin, snooze, reminders, done,
  priority, set aside, smart categories. Not listed as gaps.

## 6. Non-mail features (outside the spec's stated scope)

- [ ] **Calendar: list events, create / update / delete, RSVP, availability** (Cost: L)
- [ ] **Contacts search** (Cost: L)
- [ ] **Meeting transcripts** (Cost: not applicable, Spark-specific)
- [ ] **Teams, shared inboxes, comments, assignments** (Cost: not applicable, Spark-specific)
- [ ] **New-senders view (GateKeeper)** (Cost: M, could be derived from Sent history)

## 7. Operational model

- [ ] **Per-account or per-client access levels** (Cost: M)
  Spark enforces read-only or triage per account, configured in the desktop
  app. The spec grants every valid token all nine operations. Possible shape:
  a scope claim or per-client-id policy mapping to allowed operations and
  accounts.

- [ ] **Agent skill file for REST-only clients** (Cost: S)
  Spark ships a `SKILL.md` via `spark skill`. The spec has MCP tool
  descriptions but nothing equivalent for Codex, scripts, or other REST-only
  clients. Possible shape: `SKILL.md` alongside the README.

- [ ] **Attachment download / streaming** (Cost: M)
  Spark downloads on demand and can stream bytes to stdout for sandboxed
  agents. The spec stores metadata only, by explicit decision.

- [ ] **Preserve links when converting HTML to text** (Cost: S)
  Spark renders bodies as markdown with links kept as `[text](url)`. Confirm
  the `html-to-text` configuration keeps hrefs rather than dropping them.

---

## Where the spec is already ahead of Spark

Listed so the comparison is two-sided. No action needed.

- **Remote and headless.** Spark refuses to run in a sandbox, container, or
  any environment without the desktop session. The gateway is reachable from
  claude.ai, ChatGPT, and scripts over the network with OAuth.
- **CJK search.** Spark's keyword search returned nothing for the
  two-character topic `账单`, while a subject-only filter for the same word
  matched 15 messages and a longer prefix of that subject matched one. Its
  CJK matching looks whole-token or prefix based. The spec's per-character
  tokenization handles this case.
- **Audit log, explicit sync, and `synced_at` freshness.** Spark has none.
- **Guaranteed read-only behavior** with PEEK fetches, versus Spark's access
  levels that depend on desktop settings.

## Notes from the session

- Spark's `folders` output per account and per Gmail label, with counts, is a
  useful reference for what "all folders" would mean for each provider.
- Spark topic search output format: header block per hit (ID, Subject, From,
  To, Date, Flags) followed by the full body. Thread output adds `Type`,
  a `Link:` deep link, and an `Attachments:` table with ID, name, size, MIME
  type, and local cache path.
