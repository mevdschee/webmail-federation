# Webmail Federation — Design

## Context

### webmail (backend)

PHP / MintyPHP. Schema already has a `domain` table (`id`, `name`, `customer_id`) but `EmailClient::getDefaultDomain()` treats it as a singleton (`SELECT name FROM domain WHERE customer_id IS NULL LIMIT 1`).

Mail flow — no SMTP:

- `EmailClient::addThread` → `addMessageToSubscribers` writes `g_thread` / `g_message` once and fans out per-subscriber `thread` / `message` rows.
- Unknown recipients trigger `addCustomer`, which auto-creates `customer` + `user` + `mailbox` + `folder` + `alias` with a random `{7digit}@103mail.com` handle.
- A `sent_email` row logs a metadata-only notification (secure link, no content) — this is the only thing that leaves the server.

REST API lives under `/api/v1/email/...` with Bearer tokens from the `token` table. Compose hits `POST /thread/{folderId}` with `{ subscribers: [emails], subject, body, attachments }`.

### webmail-client (frontend)

React 19 + MUI, currently mock-only. `103mail.com` is hardcoded in `AuthFlow.tsx`, the `/103mail.svg` asset, and all mock data (`@103mail.com` everywhere). No real API client yet — clean slate to introduce federation awareness.

---

## Design: shared client across federated domains

### 0. The four levels: Server, Customer, Domain, User

Before the topology, a word on the entities that structure everything below. The existing webmail schema is multi-tenant (see /home/maurits/projects/webmail/specs/001-internal-messaging/data-model.md), so federation inherits four levels, not three:

```
Server      physical deployment — process + database
  └── Customer    tenant / billing / admin boundary — internal only
        └── Domain      DNS domain — federation participant on the wire
              └── User        mailbox holder — identified by sender_email
```

What each level is:

- **Server** — a running PHP process with a database. An operator may run one or many. Purely an operational concept; never on the wire.
- **Customer** — the `customer` row in the existing schema (`customer(id, name, default_mailbox_id)`). The unit of billing, administration, and data-isolation. `customer_id` scopes almost every per-tenant table (`user`, `mailbox`, `folder`, `alias`, `thread`, `message`, `attachment`, `contact`, `token`, `sent_email`, `domain`, `audit_log`). Admins are admins-of-a-Customer, not admins-of-a-Server. Also internal: the RFC never transmits it.
- **Domain** — the `domain` row (`domain(id, name, customer_id)`). Owned by exactly one Customer. This is the federation participant: it has a DNS name, a Federation hostname (`webmail.{domain}`), a peer descriptor, and its own Ed25519 keypair. Peers see Domains and nothing else.
- **User** — the `user` row (`user(id, customer_id, ...)`). Belongs to exactly one Customer, which means a User's mailbox lives inside that Customer's tenancy. A User's addresses (via `alias`) may be in any Domain owned by their Customer.

**Cardinalities:**

- one Server → N Customers
- one Customer → N Domains (a single Customer may own `acme-eu.com`, `acme-us.com`, `acme.io` together)
- one Customer → N Users (the Users inside that Customer)
- one Domain → N Users indirectly, via the aliases held in that Domain by Users of the Domain's owning Customer
- one User → N Aliases, which may span Domains within the User's Customer

**The wire invariant: Domains look independent; Customer is hidden.** Two Domains owned by the same Customer on the same Server federate as if they were two strangers: separate descriptors, separate keypairs, separate envelopes. A peer cannot tell that `acme-eu.com` and `acme-us.com` are co-owned, and the protocol does not let us tell them. Internally the Server may short-circuit delivery between them (§13.6) and share user accounts across them (§13.4), but none of that is observable from outside.

Why keep Customer rather than collapse it into Domain? Because the jobs are different:

- Customer is the billing/admin/isolation unit. It outlives the specific DNS domains it happens to own; a customer can add `acme.io` next year and keep the same user accounts.
- Domain is the federation identity. It is fixed to one DNS name and one keypair.

Forcing Customer = Domain (i.e. three levels) would mean either "one customer per DNS domain" (breaks the Acme-with-three-domains case and forces duplicate user accounts) or "one DNS domain per customer" (forbids multi-domain tenants). Neither is what the existing schema supports, and neither matches how B2B mail tenancy actually works.

The rest of this document references these four levels by name. Where something is "per-Customer" or "per-Domain", the distinction matters.

### 1. Deployment topology

```
webmail.103mail.com   ◄──►  client bundle + webmail backend (103mail domain)
webmail.example.com   ◄──►  client bundle + webmail backend (example.com domain)
webmail.acme.io       ◄──►  client bundle + webmail backend (acme.io domain)
```

**One Domain, one hostname.** Every Domain serves its entire webmail surface — the static client bundle, the REST API, the federation endpoints, and the `/.well-known/webmail-federation` descriptor — under the single hostname `webmail.{domain}`. No other subdomain (`mail.`, `api.`, `m.`, the apex) is permitted for any webmail-federation endpoint; this is normative in the RFC (§1 Federation hostname, §4.1, §9.1). The client bundle and the API share an origin, which also removes the cross-origin handshake between them.

**Transport.** `https://webmail.{domain}` **MUST** be served over TLS 1.2+ with an X.509 certificate that chains to the Web PKI and is valid for `webmail.{domain}`. Domain-validated (DV) certificates are the minimum acceptable validation level — free issuers like Let's Encrypt satisfy the requirement. OV/EV certificates are permitted but not required. Self-signed certificates, private-CA certificates, and certificates whose SANs do not cover `webmail.{domain}` are not compliant.

Each Domain has its own keypair and branding. A Server **MAY** host a single Domain (the minimal deployment) or several co-hosted Domains sharing a process and DB (§13); peers can't tell the difference. Domains exchange mail via **federation HTTP**, not SMTP.

### 2. Client: drop the domain hardcode

Single bundle, per-domain runtime config resolved from the hostname:

- Add `/config.json` served by each backend at `https://webmail.{domain}/config.json`:
  ```json
  {
    "domain": "103mail.com",
    "apiBase": "https://webmail.103mail.com/api/v1",
    "brand": {
      "name": "103mail",
      "logo": "/103mail.svg",
      "marketingUrl": "https://103mail.com"
    }
  }
  ```
  `apiBase` **MUST** be rooted at `https://webmail.{domain}`; the client rejects any config that points elsewhere, matching the RFC rule that all protocol endpoints live under the single Federation hostname.
- Client bootstraps by fetching `/config.json` before first render; passes the result through a `DomainContext`.
- Replace hardcoded `/103mail.svg`, `https://103mail.com` link, and `@103mail.com` mock strings with `config.brand.*` / `config.domain`.
- Login form: local part only, with `@{config.domain}` shown as an adornment — reinforces that each hostname is a single Domain.
- `AuthContext` stores `{ token, email, domain }`; token scoped per-Domain in `localStorage` keyed by domain.

### 3. Backend: domain config, not implicit singleton

- New `config/domain.php` (or env): `local_domains` — a list of Domains this Server hosts, each with a `domain`, its own `federation_private_key` / `federation_public_key`, and brand metadata. A single-Domain Server is just the one-entry case. Multi-Domain mechanics (Host-header routing, per-Domain well-known, per-Domain keys) are specified in §13.
- Change `EmailClient::getDefaultDomain()` to read domain config rather than the "customer_id IS NULL" row. In multi-Domain Servers, the active Domain is resolved per-request from the `Host` header (§13.2); single-Domain Servers degenerate to the one entry.
- Extend the `domain` table:
  ```
  domain.backend_url      VARCHAR   -- NULL = local
  domain.public_key       TEXT      -- for verifying inbound
  domain.is_local         BOOLEAN   -- this Server hosts the domain
  domain.last_seen_at     DATETIME  -- discovery cache timestamp
  ```
  `domain.customer_id` stays as-is: every Domain — local or remote — has an owning Customer. For locally-hosted Domains that Customer is the tenant whose users actually live there (§0); for remote Domains populated via discovery, the row is synthesized with a `customer_id` that represents "this Server's view of that remote tenant" (a placeholder Customer row is fine) so that the NOT-NULL constraint and the existing `customer_id` scoping continue to hold uniformly.
- Retire the "one row where customer_id IS NULL" convention; every hosted Domain's `domain` row is marked `is_local = true` with the owning Customer's `customer_id`, and other domains are populated lazily via discovery.

### 4. Fan-out split: local vs remote recipients

**Reliability model for everything that follows.** Every cross-Server call in this design — envelope delivery (§4, §6), peer discovery (§5), shadow-offer / claim / backfill (§14.4), registrar lookup (§16.3), pending-shadows query (§16.4), claim-by-real-email (§15.7) — runs over HTTPS to a remote Server we do not control. Treat it as unreliable by default. Concretely:

- **Async at the user boundary.** No compose, no reply, no claim-accept ever blocks on a remote peer's response. The flow is: (1) write local `g_*` and per-user rows, (2) return success to the user, (3) enqueue the outbound call, (4) a background worker actually makes the network attempt. Alice's browser sees `201 Created` from her own Server before any packet goes to a peer. The only thing that can make a user-visible operation fail is a local error (validation, DB write, auth) — never a peer being down.
- **At-least-once delivery + idempotent receivers.** Every outbound mutating call carries a UUIDv7 identifier chosen before the first attempt and reused on retry. Receivers dedupe on `(origin_domain, envelope_id)` for inbound envelopes, `(origin_domain, offer_id)` for shadow-offers, `(origin_domain, claim_id)` for claims. A duplicate produces the original `2xx` response with no re-application of side effects.
- **Queued + retried.** Every outbound call has a row in the single `federation_outbox` table (schema and rules below) with `kind`, `status`, `attempts`, `last_error`, `next_retry_at`. Envelopes, shadow-offers, claims, backfills, and `claim-by-real-email` posts all share the same queue and the same worker — `kind` discriminates the payload, not the delivery machinery. Retry is exponential with full jitter, 30s → 1h, 48h budget (matching RFC §6.4). Workers are idempotent — crashing mid-send and re-running from the queue produces the same outcome, not a duplicate.
- **Bounded timeouts.** Every HTTP call sets a connect timeout (~5s) and a read timeout (~30s for envelopes, ~2s for registrar lookups, ~10s for well-known fetches). "Stuck" never means "waits forever" — it means "waits the budget and then fails transient." This applies especially to registrar `/lookup` (§16.3), which sits in the compose path: a 2s cap with fall-through to local auto-onboarding keeps user-visible latency predictable regardless of peer health.
- **Permanent failure has a user-visible consequence.** Exhausting the retry budget or receiving a terminal 4xx **always** produces a local bounce in the sender's mailbox (§7 notifications), never a silent drop. "Delivered" in the sender's Sent folder means "durably enqueued locally," not "accepted by the remote peer" — but if the remote peer permanently refuses, the sender is told. Alice never has to discover a lost message by asking Bob.
- **Fan-out is per-destination independent.** Alice sending to five external Domains produces five independent `federation_outbox` rows. One peer being slow or down has zero effect on the other four. There is no "all-or-nothing" batch; there is no global lock; there is no cross-peer ordering constraint.
- **Reads are retried too, just more cheaply.** Discovery (`GET /.well-known/...`) and lookup (`GET /api/v1/registrar/lookup`) are naturally idempotent, so retries are free. They still respect the same timeout / fallback discipline: a failed discovery means the envelope sits in `federation_outbox` with `status = pending`, not that the send explodes.

Assume the network is hostile and the peer is often unreachable. Every design choice below — idempotency keys, queue tables, TOFU caching, default-registrar fallback, claim-protocol signatures — is downstream of this assumption.

`EmailClient::addMessageToSubscribers` becomes:

```
for each subscriber email:
    domain = parse(email)
    if domain in local_domains  or  domain_row(domain).is_local:
        existing path: addCustomer-if-missing + insert thread/message rows
    else:
        resolve peer backend for `domain` (discovery, §5)
        enqueue outbound HTTP POST on the federation_outbox table
        background worker retries with exponential backoff
```

Invariant: `g_thread` / `g_message` / `g_attachment` are written once locally so the sender keeps their Sent copy. Only the per-recipient `thread`/`message` insert is replaced by a network hop when the recipient is remote.

**The `federation_outbox` table.** The reliability model above dictates a specific concrete shape: a single, generic table holding every outstanding cross-Server HTTP request this Server still owes to a peer. Envelopes (§6), shadow-offers (§14.4), claims (§14.4), backfill envelopes (§14.4), and `claim-by-real-email` posts (§15.7) all enqueue rows here. A single background worker drains it.

```
federation_outbox (
  id               BIGINT       PK,
  kind             ENUM         -- envelope | offer | claim | backfill
                                -- | claim_by_real_email
  method           VARCHAR      -- always "POST" in v1
  peer_domain      VARCHAR      -- the peer DNS domain (for logging + rate-limit)
  url              TEXT         -- full https://webmail.{peer}/... URL
  headers          JSON         -- X-Federation-Sender / Nonce / Signature, ...
  body             LONGBLOB     -- the exact bytes to send (signature was
                                -- computed over this — do NOT re-serialize
                                -- at send time)
  idempotency_key  VARCHAR      -- envelope_id | offer_id | claim_id
                                -- (whichever UUIDv7 identifies this request)
  origin_domain    VARCHAR      -- which of our local_domains owns this send
                                -- (picks the signing key)
  related_id       BIGINT       -- FK into g_message / alias / offer / etc.
                                -- (polymorphic via `kind`; used for
                                -- bounce routing, UI, audit)
  status           ENUM         -- pending | in_flight | delivered
                                -- | permanent_failure
  attempts         INT          -- incremented per send try
  last_error       TEXT         -- transport / status / body of last fail
  last_http_status INT          -- NULL until we've seen any HTTP response
  next_retry_at    DATETIME     -- the worker wakes up on this
  created_at       DATETIME,
  delivered_at     DATETIME     NULLABLE,
  UNIQUE(origin_domain, idempotency_key)
)
```

Rules of this table:

- **Write-before-send.** The row is inserted in the same DB transaction that writes the user-facing `g_*` rows or §14 / §15 state change. No row, no send attempt. This is what makes the write-locally-then-return-success flow durable: a worker crash after commit still re-sends the request on restart.
- **Body is bytes, not a recipe.** The HTTP body is stored exactly as signed. On retry the worker re-uses the stored `body` and `headers` verbatim — never re-serialized, never re-signed. Anything else races the signature against JSON key ordering and clock-skew in nonce reuse. Corollary: the stored `body` is immutable; schema migrations to envelope shape do not rewrite old rows, they only affect new sends.
- **Single worker, but parallel per-peer.** One worker process (or pool) claims rows where `status = 'pending' AND next_retry_at <= NOW()`, flips them to `in_flight`, does the `curl`, writes back the result. The claim query partitions by `peer_domain` so one slow peer cannot starve sends to other peers (§4 preamble: no synchronous fan-out across peers). A per-peer concurrency cap plus the `peer_domain` index keeps one misbehaving peer from eating the worker pool.
- **Retry schedule.** `next_retry_at = now + expbackoff(attempts, base=30s, cap=1h, jitter=full)`, ceiling 48h total budget (matching RFC §6.4). On `202` or `200`: `status = 'delivered'`, `delivered_at = NOW()`. On transient (5xx / 408 / 425 / 429 / connect timeout / read timeout / TLS error): `attempts++`, reschedule. On permanent 4xx or budget-exhausted: `status = 'permanent_failure'`, and the consequence is routed by `kind` — an envelope failure generates a local bounce in the sender's mailbox, an offer failure is silently retired, a claim failure is surfaced in the claiming user's UI.
- **Idempotency key is the protocol identifier.** `envelope_id` for envelopes, `offer_id` for shadow-offers, `claim_id` for claims, and so on — all UUIDv7 generated at enqueue time, never at send time. The `UNIQUE(origin_domain, idempotency_key)` constraint makes double-enqueue a database error, not a duplicate send.
- **Honors `Retry-After`.** If the peer returns `429` with a `retry_after_seconds` body field or a `Retry-After` header, `next_retry_at` is set no earlier than that time for that `peer_domain`. Applies to the whole peer, not just the row, so follow-up sends to that peer also back off.
- **Observable as-is.** The table doubles as the operational queue and the forensic log. "What is this Server currently owing to that Server?" is a single `SELECT ... WHERE peer_domain = ? AND status IN ('pending', 'in_flight')`. Delivered rows may be pruned after N days; permanent_failure rows are kept for audit and are what `sent_email`-bounce templates reference.
- **Nothing in the user-facing code path writes HTTP directly.** Callers stage `federation_outbox` rows and return; the worker owns every outbound `curl`. This is the concrete mechanism for "async at the user boundary" — there is no branch in request-handling code that waits on a peer.

Inbound requests have no analogous table. A peer's envelope hits `/api/v1/federation/inbound`, is verified and written into `g_thread` / `g_message` / per-user rows in one transaction, and the peer gets `202` before the response returns. The idempotency key on the receiver side is the incoming `envelope_id` (or `offer_id` / `claim_id`), stored in a small `federation_inbox_seen(origin_domain, kind, idempotency_key, first_seen_at)` dedupe table so replays produce the original response without re-applying side effects. That table is tiny, TTL'd per RFC §9.3 nonce-window rules, and does not need the retry machinery above.

### 5. Peer discovery + trust

- Each backend exposes its peer descriptor at the fixed Federation hostname `https://webmail.{domain}/.well-known/webmail-federation`:
  ```json
  {
    "domain": "example.com",
    "backend": "https://webmail.example.com",
    "inbound": "https://webmail.example.com/api/v1/federation/inbound",
    "public_key": "ed25519:..."
  }
  ```
  The descriptor's `backend` and `inbound` **MUST** be rooted at `https://webmail.{domain}`. A descriptor advertising any other hostname is rejected.
- First time 103mail needs to talk to `example.com`: HTTP `GET https://webmail.example.com/.well-known/webmail-federation`, pin the pubkey into the `domain` row, cache for 24h. A 103mail peer **MUST NOT** probe the apex (`https://example.com/...`) or any alternative subdomain; discovery is a fixed-URL fetch, not a search.
- The fetch is HTTPS-only and the certificate **MUST** be a Web-PKI certificate valid for `webmail.example.com` (DV minimum, §1). Self-signed, private-CA, and bare-hostname certificates are refused at the TLS layer before any descriptor bytes are read.
- Web-PKI CA chain + TOFU pubkey pin: the CA check authenticates the hostname at first contact; the pinned Ed25519 key authenticates every subsequent envelope regardless of CA changes.

### 6. Inbound federation endpoint

New `FederationApi` (pair of `EmailApi`):

```
POST /api/v1/federation/inbound
Headers: X-Federation-Sender:    example.com
         X-Federation-Signature: ed25519(body)
Body: {
  thread:      { subject, created_at, federation_id, ... },
  message:     { sender_email, sender_name, body, body_html, sent_at, ... },
  attachments: [ ... ],
  recipients:  [ local emails this payload is for ]
}
```

On receipt:

1. Verify signature against cached pubkey for `example.com` (re-discover if stale).
2. Reject if `sender_email` domain ≠ `X-Federation-Sender` — prevents one peer spoofing as another.
3. Reuse the local branch of `addMessageToSubscribers`: auto-create recipient customers on THIS backend (they're local users here), insert rows, trigger a `sent_email` notification with a link back to *this* backend.

### 7. Notifications follow the recipient

Currently `sent_email` links back to the backend that ran the code. That stays correct post-federation because notifications are generated by the **recipient's** backend after inbound delivery, not the sender's. Only change: the notification template's base URL pulls from domain config rather than a constant.

### 8. Auto-created aliases stay local

When `addCustomer` runs for a remote recipient today, it creates a local mailbox — that was the single-domain shortcut and is wrong under federation. Under the new design, remote recipients are pushed over federation; the local DB only gains a `contact` row so the sender sees them in autocomplete.

### 9. Compose UI change (client)

The `POST /thread/{folderId}` payload is unchanged — `subscribers` is still a list of emails. Federation is invisible to the UI. The only client change is the domain-config extraction in §2.

### 10. Walkthrough: sending a cross-domain message

Alice (`1234567@103mail.com`) composes a message to Bob (`bob@example.com`). Both domains are served by separate Webmail backends. The sequence:

**Step A — client → origin backend.** Alice's browser posts the usual compose payload to her own backend:

```
POST https://webmail.103mail.com/api/v1/email/thread/{folderId}
Authorization: Bearer {alice's token}

{
  "subscribers": ["bob@example.com"],
  "subject":     "Hello from 103mail",
  "body":        "Hi Bob — ...",
  "attachments": []
}
```

The client has no knowledge that `example.com` is remote. Federation is entirely a backend concern.

**Step B — origin backend writes locally, once.** `EmailClient::addThread` on the 103mail backend writes (exactly as today):

- `g_thread` — with a new `federation_id = uuidv7()` added by this design.
- `g_message` — with its own `federation_id = uuidv7()`, `sender_email = 1234567@103mail.com`.
- Alice's per-user rows: `thread` (in her Sent folder) and `message` (marked read).

This guarantees Alice sees her Sent copy even if example.com is unreachable.

**Step C — origin backend splits fan-out.** `addMessageToSubscribers` iterates `["bob@example.com"]`:

- parses domain → `example.com`.
- looks up the `domain` row: not marked `is_local`.
- delegates to the outbound federation branch.

No local mailbox is created for Bob (§8). A `contact` row for Bob is upserted for autocomplete only.

**Step D — origin backend discovers example.com (first time only).** If `domain.last_seen_at` is stale or missing:

```
GET https://webmail.example.com/.well-known/webmail-federation
```

The fetch enforces TLS with a Web-PKI DV-or-better certificate valid for `webmail.example.com`; any cert failure aborts discovery (no fallback to the apex, no plaintext, no private-CA override). The returned `public_key` is pinned into the `domain` row on TOFU (§5). Cached for 24h.

**Step E — origin backend enqueues the envelope.** A row is inserted into `federation_outbox`:

```
(id, kind = "envelope", peer_domain = "example.com",
 url = "https://webmail.example.com/api/v1/federation/inbound",
 headers = {...signed headers...},
 body = {...signed envelope bytes...},
 idempotency_key = envelope_id,
 origin_domain = "103mail.com",
 related_id = g_message.id,
 status = "pending", attempts = 0, next_retry_at = NOW())
```

The actual HTTP call is done by a background worker, so Alice's compose HTTP request returns immediately (the message is already in her Sent folder; federation is async).

**Step F — worker signs and posts the envelope.** The worker picks up the pending row and sends:

```
POST https://webmail.example.com/api/v1/federation/inbound
X-Federation-Version:   0.1
X-Federation-Sender:    103mail.com
X-Federation-Nonce:     {uuidv7}
X-Federation-Signature: ed25519(body || "\n" || nonce)

{
  "version":       "0.1",
  "envelope_id":   "{uuidv7}",
  "origin_domain": "103mail.com",
  "origin_sent_at":"2026-04-18T10:15:00Z",
  "thread":  { "federation_id": "{g_thread.federation_id}",
               "subject":       "Hello from 103mail",
               "in_reply_to":   null },
  "message": { "federation_id": "{g_message.federation_id}",
               "sender_email":  "1234567@103mail.com",
               "sender_name":   "Alice",
               "body":          "Hi Bob — ...",
               "body_html":     "<p>Hi Bob — ...</p>",
               "sent_at":       "2026-04-18T10:15:00Z" },
  "attachments": [],
  "recipients":  ["bob@example.com"]
}
```

On `202 Accepted`, `federation_outbox.status` flips to `delivered`. On transient failure (5xx, 429), the retry clock starts (30s → 1h with jitter, 48h budget). On permanent failure (4xx), a local bounce message is inserted into Alice's inbox.

**Step G — destination backend verifies and delivers.** The example.com backend:

1. Verifies the Ed25519 signature against its cached pubkey for `103mail.com`.
2. Checks `envelope.origin_domain == X-Federation-Sender == domain-of(sender_email)`.
3. Dedupes on `(origin_domain, envelope_id)`.
4. Writes local `g_thread` / `g_message` rows, carrying over the `federation_id` values so replies can correlate.
5. Calls the **local** branch of `addMessageToSubscribers` for `bob@example.com`: if Bob has no account, the existing auto-onboarding creates `customer` + `user` + `mailbox` + `folder` + `alias` — but now with an `@example.com` alias, because this backend's domain config says its local domain is `example.com`.
6. Inserts Bob's per-user `thread` / `message`, unread.
7. Enqueues a `sent_email` notification linking to `https://webmail.example.com/...` — **Bob's home backend**, not Alice's.

Bob receives the notification email, clicks the link, logs into his example.com mailbox, and reads the message. 103mail.com never touches Bob's credentials or mailbox contents.

### 11. Walkthrough: replying across domains

Bob opens the thread and hits Reply. The reply travels back along the same rails, mirrored.

**Step A — Bob's client → example.com backend.** The client posts a reply to example.com's backend:

```
POST https://webmail.example.com/api/v1/email/message/{threadId}
Authorization: Bearer {bob's token}

{ "body": "Thanks Alice — ...", "attachments": [] }
```

`{threadId}` here is Bob's **local** per-user `thread.id` on example.com (an autoincrement). The frontend doesn't see federation ids.

**Step B — example.com backend writes locally.** `EmailClient::addMessage` appends a new `g_message` to the **same** `g_thread` Bob received in §10 Step G. Because that `g_thread` has a `federation_id` stored from inbound delivery, example.com knows this thread is federated. The new `g_message` gets its own fresh `federation_id`.

Bob's per-user Sent copy is written, unread-for-others.

**Step C — example.com backend walks the subscriber list.** The `g_thread` has subscribers `{Alice, Bob}`. Iterating:

- `bob@example.com` → local, no network.
- `1234567@103mail.com` → domain `103mail.com`, not `is_local` on this backend → outbound federation branch.

**Step D — example.com enqueues and posts to 103mail.** A `federation_outbox` row is inserted on the example.com side (the table lives on whichever server is the origin of a given envelope). The worker posts:

```
POST https://webmail.103mail.com/api/v1/federation/inbound
X-Federation-Sender: example.com
X-Federation-Signature: ...

{
  "version":       "0.1",
  "envelope_id":   "{uuidv7}",
  "origin_domain": "example.com",
  "origin_sent_at":"2026-04-18T10:32:00Z",
  "thread":  { "federation_id": "{same g_thread.federation_id as in §10}",
               "subject":       "Hello from 103mail",
               "in_reply_to":   "{§10 message.federation_id}" },
  "message": { "federation_id": "{new uuidv7}",
               "sender_email":  "bob@example.com",
               "sender_name":   "Bob",
               "body":          "Thanks Alice — ...",
               "body_html":     "<p>Thanks Alice — ...</p>",
               "sent_at":       "2026-04-18T10:32:00Z" },
  "attachments": [],
  "recipients":  ["1234567@103mail.com"]
}
```

Two things to note:

- `thread.federation_id` is **the same value** that 103mail generated when Alice first composed in §10 Step B. It was carried in the original envelope, stored on example.com's `g_thread`, and is now echoed back. This is how the reply correlates to the existing thread on both sides.
- `thread.in_reply_to` references the specific message being replied to (§10's `g_message.federation_id`) — optional for threading but useful for quoting / UI.

**Step E — 103mail backend verifies and appends.** The 103mail backend:

1. Verifies signature with its cached pubkey for `example.com`.
2. Dedupes on `(origin_domain = "example.com", envelope_id)`.
3. Looks up `g_thread` by `federation_id`. **Found** — this is a reply to an existing thread, not a new thread. No new `g_thread` row is created.
4. Appends a new `g_message` (with Bob's federation id) under that existing `g_thread`.
5. Walks the thread's **local** subscribers and inserts a per-user `message` row for Alice, unread, into her existing `thread`.
6. Enqueues a `sent_email` notification to Alice linking to `https://webmail.103mail.com/...`.

Alice gets notified, opens her thread, and sees Bob's reply appended under the same subject. From her UI the conversation looks local — threading, unread counts, and quoted replies all work as if example.com were just another 103mail user.

### 12. What each server persists

After the exchange in §10–§11, the state on each side is:

| Table            | 103mail row                                  | example.com row                              |
|------------------|----------------------------------------------|----------------------------------------------|
| `g_thread`       | federation_id = T, subject = "Hello ..."     | federation_id = **T**, subject = "Hello ..." |
| `g_message` (1)  | federation_id = M1, sender = Alice           | federation_id = **M1**, sender = Alice       |
| `g_message` (2)  | federation_id = M2, sender = Bob             | federation_id = **M2**, sender = Bob         |
| `thread`         | one row — Alice's view                       | one row — Bob's view                         |
| `message` × 2    | Alice's Sent (M1) + Alice's Inbox (M2)       | Bob's Inbox (M1) + Bob's Sent (M2)           |
| `federation_outbox` | one row (outbound M1 to example.com)    | one row (outbound M2 to 103mail)             |

The federated identifiers `T`, `M1`, `M2` are identical across both databases; the autoincrement local ids are not. This is what makes reply correlation and idempotent re-delivery possible.

### 13. Multi-Domain Servers

Nothing in the wire protocol requires that one DNS domain equal one Server. A single Server **MAY** host several Domains at once, either because one Customer owns several DNS domains (e.g. `Acme Corp` owning `acme-eu.com`, `acme-us.com`, `acme.io`) or because the Server is multi-Customer (e.g. one operator runs `103mail.com` for Customer A and `example.com` for Customer B on the same process and database). The four-level hierarchy from §0 cleanly covers both cases: `Customer → Domain` is a one-to-many relation, and a Server may host any number of Customers.

Each Domain remains an independent federation participant on the wire — the RFC doesn't know, and its peers can't tell, that two Domains share a process or that they share a Customer. A single-Customer/single-Domain Server is this same setup with one `customer` row and `local_domains` holding one entry, so the multi-Domain machinery below is not a separate mode but the general case.

#### 13.1 What changes in Domain config

`config/domain.php` (or the equivalent env-backed loader) returns a list of Domains this Server hosts. Each entry names the DNS domain and the owning Customer:

```
local_domains: [
  {
    "domain":          "acme-eu.com",
    "customer_id":     42,                              # Acme Corp
    "brand":           { "name": "Acme (EU)",  "logo": "/acme-eu.svg",
                         "marketing_url": "https://acme.com" },
    "federation_private_key": "ed25519:...",
    "federation_public_key":  "ed25519:...",
  },
  {
    "domain":          "acme-us.com",
    "customer_id":     42,                              # same Customer
    "brand":           { "name": "Acme (US)",  "logo": "/acme-us.svg",
                         "marketing_url": "https://acme.com" },
    "federation_private_key": "ed25519:...",
    "federation_public_key":  "ed25519:...",
  },
  {
    "domain":          "example.com",
    "customer_id":     99,                              # different Customer
    "brand":           { "name": "Example Mail", "logo": "/example.svg",
                         "marketing_url": "https://example.com" },
    "federation_private_key": "ed25519:...",
    "federation_public_key":  "ed25519:...",
  }
]
```

`customer_id` ties the Domain to its owning Customer (matching `domain.customer_id` in the DB). The first two entries share a Customer and therefore share users, passwords, mailboxes, and admin scope (§13.4, §13.9). The third is a different Customer on the same Server, fully isolated at the `customer_id` boundary (§13.8).

The Federation hostname `webmail.{domain}` is **derived**, not configured, from the `domain` field — `webmail.acme-eu.com`, `webmail.example.com`. The config loader rejects any attempt to override the hostname (e.g. a legacy `hostname: "mail.acme-eu.com"` field) so that a deploy cannot accidentally drift away from the RFC's single-hostname rule.

Each Domain has its **own** federation keypair — never shared, because remote peers pin keys per DNS domain (§5). Compromise of one Domain's key must not let an attacker impersonate the others co-hosted on the same Server. Each Domain also **MUST** present an X.509 certificate valid for its own `webmail.{domain}` (DV minimum); a single wildcard certificate covering `webmail.*.{parent-zone}` is acceptable only if it is actually valid for every hosted Domain's Federation hostname.

#### 13.2 Request routing: the Host header selects the domain

Because the browser always knows the user as "the person at `webmail.example.com`", the backend picks the active domain from the incoming HTTP `Host` header. Every webmail and federation endpoint lives at a single hostname per Domain — `webmail.{domain}` — so the mapping `Host → active domain` is one lookup:

- `GET https://webmail.103mail.com/config.json` → 103mail's brand block.
- `GET https://webmail.example.com/config.json` → example.com's brand block.
- `POST https://webmail.103mail.com/api/v1/email/thread/...` → operates inside the 103mail domain.
- `POST https://webmail.example.com/api/v1/email/thread/...` → operates inside the example.com domain.
- `POST https://webmail.103mail.com/api/v1/federation/inbound` → inbound federation, 103mail.

A lightweight middleware at the top of request handling resolves `Host` → one entry in `local_domains` by stripping the leading `webmail.` label, stores it in request state as `$activeDomain`, and rejects (404) any hostname whose form is not `webmail.{d}` for some `d` in the config. Requests to the apex (`Host: example.com`), to legacy subdomains (`Host: mail.example.com`, `Host: api.example.com`), or to any other hostname are rejected at the same layer — not silently accepted and routed. Every downstream call that used to reach for a singleton default domain now takes `$activeDomain` instead.

#### 13.3 Each Domain gets its own well-known endpoint

The same Server serves:

- `https://webmail.103mail.com/.well-known/webmail-federation` → `{domain: "103mail.com", public_key: "ed25519:K_103", ...}`
- `https://webmail.example.com/.well-known/webmail-federation` → `{domain: "example.com", public_key: "ed25519:K_ex", ...}`

Selection is again `Host`-based on the `webmail.{d}` form. The apex (`https://103mail.com/...`) is not a protocol endpoint and **MUST NOT** serve the descriptor; operators may use it for a marketing page or may leave it without a webmail-federation route entirely. Peers never learn that these two Domains share a Server.

#### 13.4 Customer, User, and Domain ownership

The four-level hierarchy (§0) drives the rules here — specifically, that Customer sits between Server and Domain, and that Users belong to Customers, not to Domains directly.

- **Customer ownership of Domains.** `domain.customer_id` is already in the schema. A Customer may own one Domain (`Beta Inc` owning just `beta.io`) or several (`Acme Corp` owning `acme-eu.com`, `acme-us.com`, `acme.io`). All Domains a Customer owns **MUST** be hosted on the same Server — ownership is resolved through one `customer` row and therefore one database. Co-hosted Domains with different owners are a separate case (§13.9: one Server, N Customers).
- **User belongs to Customer, not to Domain.** `user.customer_id` is existing; there is no `user.domain_id` column and this design does not add one. A User's reachable identities are the `alias` rows attached to their mailbox, each of which carries an `email` whose domain part is one of the Customer's Domains. Alice at Acme may have `alice@acme-eu.com` and `alice@acme-us.com` as aliases on the same mailbox; both are valid, both deliver to the same inbox.
- **Cross-Domain single sign-on is allowed within a Customer.** Since a User has one account and one password inside their Customer, they can log in at any Federation hostname whose Domain their Customer owns. `alice` authenticates at `webmail.acme-eu.com` and at `webmail.acme-us.com` and reaches the same mailbox. No cross-Customer SSO: if `alice@acme-eu.com` and `alice@beta.io` are different Customers, they are different accounts and different passwords even if the same person holds both.
- **Aliases are keyed by unique `email`.** `alice@103mail.com` and `alice@example.com` coexist without schema change. Two different Customers cannot both own an alias for the same `email` (the uniqueness constraint prevents it); this is how address ownership stays single-sourced even on a shared Server.
- **Tokens are Domain-scoped on the wire, Customer-scoped internally.** Bearer tokens are issued by the backend for a specific (Customer, User) pair, and carry the DNS domain the token was minted for in their claims. A token minted at `webmail.acme-eu.com` is accepted at `webmail.acme-us.com` **only if** both Domains share a Customer — the backend checks `domain.customer_id` equality before honoring cross-Domain use. A token **MUST NEVER** be honored across Customer boundaries.
- **Federation identity is per-Domain, always.** On the wire, Alice's mail from `alice@acme-eu.com` is signed by the `acme-eu.com` keypair, and mail from `alice@acme-us.com` is signed by the `acme-us.com` keypair. Peers see two independent Domains. The Customer linking them is invisible — peers cannot tell that these two `alice`s are the same person, nor that the Domains are co-owned.

#### 13.5 Auto-onboarding uses the recipient's domain, not a global default

Today `addCustomer` creates a random `{7digit}@{default_domain}` alias. On a multi-Domain Server, it derives the alias domain from the recipient's email:

```
email       = "foo@example.com"
domain      = "example.com"
# Is this domain hosted here?
if domain in local_domains:
    # Resolve the owning Customer of this Domain; auto-onboarding
    # creates a user inside that Customer (user.customer_id = owning customer).
    owning_customer = local_domains[domain].customer_id
    alias_local_part = random_int(1000000, 9999999)
    alias            = f"{alias_local_part}@{domain}"
    create user(customer_id = owning_customer)
         + mailbox(customer_id = owning_customer)
         + folder(customer_id = owning_customer)
         + alias(customer_id = owning_customer, email = alias)
    # If the Customer does not yet exist (brand-new tenant), create it first.
else:
    # Not local — federation branch (§4), no onboarding.
```

The random-local-part trick for privacy is preserved; only the domain suffix is now dynamic.

#### 13.6 Delivery between co-hosted Domains short-circuits federation

When Alice at `103mail.com` sends to Bob at `example.com` **and both Domains are co-hosted on the same Server**, the fan-out loop in §4 takes the local branch for both recipients. No envelope is built, no HTTP is emitted, no signature is computed, no `federation_outbox` row is written. The exchange is a pure in-process database operation — exactly as it would be for two users on the same DNS domain.

This short-circuit is based on Server co-location (membership in `local_domains`), not Customer co-ownership. Two Domains owned by the **same Customer** short-circuit trivially — they share `customer_id` and Alice's and Bob's mailboxes sit in the same tenant rows. Two Domains owned by **different Customers** on the same Server also short-circuit, but the write crosses `customer_id`: Alice is in Customer A's rows, Bob is in Customer B's rows, and the per-recipient insert into Bob's `thread` / `message` has to happen under B's tenancy. That is permitted — it is the same operation that happens today when Acme's user messages Beta's user on a shared backend — but it is the one place where the `customer_id` boundary is crossed by a write. Customer-level data isolation (§0) still holds for reads: Alice cannot see Bob's folder tree, Bob cannot see Alice's Sent.

That property follows from the fan-out rule in §4 unchanged: membership in `local_domains` is the only test. A future operator who splits `example.com` onto its own separate Server flips one row (`domain.is_local = false`, `domain.backend_url = "https://webmail.example.com"`), and from that point the same send automatically uses federation instead. The sending code path is identical either way; only the branch taken at dispatch time differs — and the Customer layer is invisible to peers regardless of which side of the branch is taken.

#### 13.7 Notifications pick the right branded host

The `sent_email` template's base URL now resolves from the **recipient's** Domain config, not from a single default. A notification destined for `bob@example.com` points at `https://webmail.example.com/...`, and one destined for `alice@103mail.com` points at `https://webmail.103mail.com/...` — even when both are generated by the same Server in the same request.

#### 13.8 What is not shared across co-hosted Domains

Regardless of whether the Domains share a Customer:

- **Keys** — separate Ed25519 keypair per Domain (§13.1).
- **Brand assets** — separate logo, marketing URL, display name (§2, §13.1).
- **Federation identity on the wire** — separate peer descriptor, separate `origin_domain`, separate signatures (§5).

Across Domains of **different Customers**, additionally:

- **User accounts** — a User of Customer A cannot log into any Domain of Customer B; `customer_id` gates authentication.
- **Tokens** — a token minted for Customer A is rejected against any Domain whose `customer_id` is not A.
- **Mailboxes, folders, aliases, contacts** — every per-tenant table is scoped by `customer_id`; cross-Customer reads are forbidden.

#### 13.9 What is shared across co-hosted Domains

Regardless of Customer ownership:

- **The codebase and the running process** — one PHP Server handles N Customers × M Domains.
- **The database schema and, optionally, the physical database** — `customer`, `domain`, `user`, `alias`, etc. are reused; rows are scoped by `customer_id` / `domain_id`. Operators who want stricter blast-radius isolation can still deploy one database per Customer (or per Domain) without any code change; the domain loader just points each `Host` at a different DSN.
- **The outbound federation worker and the inbound federation endpoint** — they loop over `local_domains` when signing outbound envelopes (using each envelope's `origin_domain` to pick the right key) and accept inbound traffic for any hosted Domain.

Across Domains of **the same Customer**, additionally:

- **User accounts and passwords** — one `user` row serves all of the Customer's Domains; a single login works at every `webmail.{domain}` whose Domain the Customer owns (§13.4).
- **Mailboxes** — a User's single mailbox receives mail addressed to any of the User's aliases, across Domains of the User's Customer.
- **Tokens within the Customer** — a token can be accepted at any Domain owned by its Customer (§13.4); `customer_id` equality is the check.
- **Admin scope** — a Customer admin administers all of that Customer's Domains at once.

The federation protocol does not change. The operator gains two orthogonal deployment choices: (1) how many Customers per Server, and (2) how many Domains per Customer. A single-Customer/single-Domain Server is the minimal case; all four combinations are supported with the same code, same schema, and the same wire format — and peers cannot tell them apart.

### 14. Shadow mailboxes for unreachable domains

**Scope — this is only for unknown *domains*, not unknown *users*.** When Alice composes to `X@Y`:

- `Y` federates and `X` exists there → normal §6 delivery.
- `Y` federates but `X` does not exist → the destination returns `404 no_recipients` per RFC §6.3, the Origin generates a local bounce per §6.4, and §14 does **not** apply. Unknown users on federated domains bounce; they are never shadowed.
- `Y` does not federate (permanent — e.g. `gmail.com`, or terminal discovery failure on a DNS domain that may federate later) → this section applies.

When Alice sends to `bob@unknown.com` and `unknown.com` has no federation endpoint, the message cannot be pushed. Three behaviors are possible: bounce immediately (SMTP-style), bounce after a retry budget, or **hold the message in a shadow mailbox until the domain is reachable**. We take the third — it gives the sender a "delivered" experience, preserves history across the gap, and later produces a clean hand-off when the remote domain joins the federation.

**Where does the shadow live? Always at a registrar — never at the sender.** A non-registrar sender does not hold shadows locally. Instead, per §15.4, every non-registrar Domain has a single **Default Registrar** configured, and the sender's fan-out federates the undeliverable send to that registrar as an ordinary §6 envelope. The registrar runs §15.5, which includes the shadow-creation step. This is the same mechanism §15 uses for external-address recipients; §14 is the life-cycle view (hold → discover → claim → backfill) of what §15.5 step 3 creates.

A registrar that sends to an unknown domain creates the shadow in its own tenancy directly (no self-federation step).

The cost is identity management: a shadow is an unclaimed placeholder mailbox for a person who does not (yet) control any account on any Server. That cost is paid by the **claim protocol** (§14.4), which transfers shadow history to the real mailbox over federation once the domain becomes reachable, or by the real-email claim paths in §15.6.

#### 14.1 The shadow record

A shadow is an entry in the registrar's extended `alias` table (§15.2) with `state = 'tentative'`. There is **no** separate `shadow_alias` table; unifying on the §15.2 alias shape means a shadow, a forwarder, and an active canonical are all rows in one table distinguished only by `state`. The `alias.real_email` column holds the external address (e.g. `bob@unknown.com`); `alias.email` is the registrar-side 7-digit canonical (`{7digit}@{registrar_domain}`, §13.5) that backs the placeholder mailbox. Claim bookkeeping (`issued_at`, `expires_at`, transition to `forwarder` / tombstone) comes from §15.2 as well.

The backing `user` row is a normal row in the registrar's tenant — a `customer` whose mailboxes happen to hold tentative shadows alongside real users. It has no password, no login history, and is never login-enabled while the alias is `tentative`. The mailbox accumulates messages; nothing authenticates into it.

**Which Domain hosts the shadow?** Always the registrar, never the sender.

- If the sender is a non-registrar Domain, its Default Registrar (§15.4) receives the federation envelope and creates the shadow under its own active Domain. Alice on `example.com` (non-registrar) sending to `bob@unknown.com` produces a shadow at, say, `103mail.com` — because `103mail.com` is `example.com`'s Default Registrar, not because it's Alice's home.
- If the sender is itself a registrar, the shadow is created locally under the sender's active Domain (no self-federation step).

This is one change from the earlier draft, which let non-registrars hold shadows at the sender's own Domain. Routing all shadows through the Default Registrar keeps the shadow population concentrated on a small number of Domains that actually offer the `/registrar/*` endpoints, avoids split-brain shadows across all senders that ever tried to reach a given external email, and matches §15's real-email model.

Two shadows for the same `bob@unknown.com` can still exist if two different registrars each received an envelope from their own sender-base. That case is handled by §16.4 consolidation, not by §14 alone — when any one registrar transitions the address to claimed, peer registrars' pending shadows are pulled in.

#### 14.2 Onboarding trigger: fan-out fall-through on the registrar

The §4 fan-out loop has three relevant branches:

```
# On the SENDER (both registrar and non-registrar):
for each subscriber email:
    domain = parse(email)
    if domain in local_domains:
        local branch (unchanged)
    else if domain.is_local == false and domain.backend_url is set:
        remote branch (unchanged — federation push)
    else:
        # domain is unknown OR discovery has terminally failed
        if this Domain is a registrar:
            run registrar branch locally (§15.5)   # shadow lives here
        else:
            federate envelope to our Default Registrar (§15.4)
            # shadow lives on the Default Registrar, not here
```

"Discovery has terminally failed" means: a `/.well-known/webmail-federation` probe has been attempted and returned 404 / connection refused / TLS failure / invalid JSON. Transient failures (timeouts, 5xx) do **not** trigger this branch; they keep the envelope in `federation_outbox` with `status = pending` and retry until the budget is exhausted. The fall-through branch is only for the terminal "this domain does not participate" signal.

On the registrar, §15.5 step 3 creates the tentative alias, inserts the message into the backing mailbox under the registrar's tenant, and enqueues the SMTP notification (§14.2.1).

Alice's Sent folder still shows `bob@unknown.com` as the recipient. The shadow is never surfaced in her UI; she cannot tell from the compose experience whether the message ended up at her Default Registrar or a local shadow on her own Domain.

#### 14.2.1 SMTP notification to the external address

Whenever a registrar creates a tentative shadow, it **MUST** emit a metadata-only SMTP notification to the external address, telling the recipient they have mail waiting and how to claim it. The notification is sent the first time the shadow is created; subsequent messages into the same shadow **SHOULD NOT** re-notify — this is a one-shot "you have a new mailbox" signal, not a per-message alert.

The notification reuses the existing `sent_email` table and `email_template` machinery (see /home/maurits/projects/webmail/specs/001-internal-messaging/data-model.md). A new template is added:

```
email_template(
  name:           "shadow_onboarding_invite",
  language:       "en",
  sender_name:    "{registrar_brand.name}",
  sender_email:   "noreply@{registrar_domain}",
  subject:        "You have messages waiting at {registrar_brand.name}",
  body:           <metadata-only; see below>,
  parameters:     [ "external_email", "sender_display",
                    "message_count_hint", "claim_url" ]
)
```

Body outline (strictly metadata — no message body content, per the project's notifications-only invariant, /home/maurits/projects/webmail/specs/001-internal-messaging/data-model.md:95):

> Hi,
>
> Someone sent a message to **{external_email}**, but that address isn't on a federated mail service yet. {registrar_brand.name} is holding the message for you.
>
> You can claim this mailbox by clicking the link below. It's free, and you can either sign up here or link an existing federated account to collect the mail:
>
> → {claim_url}
>
> If you don't recognize this, you can ignore this email — the link expires in 30 days, and we'll stop notifying you.

`claim_url` points at `https://webmail.{registrar_domain}/claim?token=...` per §15.6. The `sent_email` row records the invite as an audit trail, the same as every other notification emitted by the system. If SMTP delivery of the invite itself fails terminally (e.g. the external address bounces), the shadow stays but the invite flag is marked undeliverable; subsequent sends to the same external email do not re-attempt notification.

The notification is distinct from — and must not be confused with — the per-message notifications a user receives for mail in a mailbox they already own. A shadow's backing mailbox never emits per-message notifications because nobody owns it; only the single onboarding invite is sent.

#### 14.3 Discovery promotion

A background worker on every registrar re-probes every `domain` row with `domain.backend_url IS NULL` for which the registrar holds at least one `alias` with `state = 'tentative'`. Frequency: low (hourly is fine). When a probe succeeds:

1. The `domain` row is promoted: `backend_url`, `public_key`, `last_seen_at` populated.
2. An **offer** envelope is sent to the newly-federated peer (§14.4), one per external email that has a shadow on our side.
3. Until the peer responds, new sends to that address use the normal remote branch (the domain is reachable now). Old messages stay in the shadow; they are only moved by a successful claim.

Promotion can also happen reactively: if we receive a well-formed inbound envelope from `unknown.com`, that is proof the domain is federated. We promote and then emit the offer envelopes.

#### 14.4 The claim protocol

Claim is a **two-party, federation-signed** handshake. There is no email verification, no out-of-band code — control of `bob@unknown.com` is proven by the fact that `unknown.com`'s authoritative federation backend signs the claim.

##### Offer (old-host → new-host)

When 103mail discovers that `unknown.com` is now federated and has shadow(s), it emits one envelope per shadow to the new peer:

```
POST https://webmail.unknown.com/api/v1/federation/shadow-offer
X-Federation-Sender:    103mail.com
X-Federation-Signature: ed25519(body)

{
  "version":        "0.1",
  "offer_id":       "{uuidv7}",
  "origin_domain":  "103mail.com",
  "external_email": "bob@unknown.com",
  "message_count":  7,
  "oldest_sent_at": "2026-02-01T...",
  "newest_sent_at": "2026-04-10T...",
  "summary": [
    { "federation_id": "{M1}", "sender_email": "alice@103mail.com",
      "subject": "Hello",    "sent_at": "..." },
    ...
  ]
}
```

The offer is metadata only — no bodies, no attachments. unknown.com stores it and, on Bob's next login, surfaces a UI prompt: *"103mail.com is holding 7 messages addressed to you from before this domain joined the federation. Claim them?"*

##### Claim (new-host → old-host)

If Bob accepts, his backend signs and posts:

```
POST https://webmail.103mail.com/api/v1/federation/claim
X-Federation-Sender:    unknown.com
X-Federation-Signature: ed25519(body)

{
  "version":        "0.1",
  "claim_id":       "{uuidv7}",
  "offer_id":       "{the offer_id from §14.4 Offer}",
  "external_email": "bob@unknown.com",
  "accepted_by":    "bob@unknown.com"
}
```

The signature over this body by `unknown.com`'s pinned key is the sole proof of control. The holding registrar (103mail in this example) verifies, looks up the tentative `alias` row for `bob@unknown.com`, and proceeds to backfill.

If Bob rejects (or the offer expires after, say, 30 days), 103mail may either re-offer on the next send to `bob@unknown.com` or delete the shadow and bounce future sends. Suggest re-offer forever — deletion is destructive and there's no strong reason to forget.

##### Backfill (old-host → new-host)

For each `g_message` in the shadow mailbox, 103mail sends a normal §6 inbound envelope to unknown.com, with one additional field:

```
{
  ...standard inbound fields...,
  "backfill_of_offer_id": "{offer_id}",
  "origin_sent_at":       "{original sent_at, NOT now}"
}
```

unknown.com treats these envelopes like any other inbound, except:

- `origin_sent_at` is preserved on the new local rows (message history stays chronological).
- The `thread.federation_id` from the original envelope — which was stored on the shadow's `g_thread` — is reused, so any other federated participant (e.g. Alice's 103mail thread) stays correlated across the claim. Future replies from Bob to that thread will match Alice's `g_thread` by federation_id on her side, exactly as in §11.

After all backfill envelopes are acknowledged:

1. The holding registrar transitions the tentative alias: `alias.state = 'tombstone'`, records the claiming domain, and timestamps the transition. The §15.2 `state` enum already has `tombstone`; no separate `shadow_alias.claimed_at` column is needed.
2. The backing user / mailbox / folder rows are **soft-deleted** (marked deleted, not dropped — audit trail).
3. The `alias` row is retained as a historical pointer: future lookups for `bob@unknown.com` see a tombstoned alias and fall through to the normal remote branch (the domain is now federated).

#### 14.5 State machine

```
            send to unknown.com
                    │
                    ▼
        ┌───────────────────────┐
        │  discovery attempted  │
        └───────────┬───────────┘
           success  │  terminal failure
         ┌──────────┴──────────┐
         ▼                     ▼
   federated remote        shadow (unclaimed)
                                │
                                │  discovery succeeds later
                                ▼
                          shadow + offer sent
                                │
                   ┌────────────┼────────────┐
            accept │            │ reject     │ expire / no response
                   ▼            ▼            ▼
                 backfill     stays        stays
                   │      (re-offer on    (re-offer on
                   ▼       next send)      next send)
             claimed + tombstone
```

Terminal states: `federated remote` and `claimed + tombstone`. Everything else is transient.

#### 14.6 Sender-visible semantics

- **During shadow hold (sender side):** Alice sees her message in Sent as usual, recipient `bob@unknown.com`. No bounce, no "pending" indicator; the envelope hit Alice's Default Registrar (if Alice is on a non-registrar Domain) with a `202 Accepted` and her view doesn't know or care what happened next. If she sends again, the registrar's §15.5 step 1 matches the tentative alias and the new message joins the same shadow mailbox.
- **At claim time:** Alice sees nothing. The backfill is entirely a registrar → claimed-domain internal matter.
- **After claim, on reply:** Bob replies from `bob@unknown.com`. The reply arrives at Alice via the normal §11 path; the `thread.federation_id` matches Alice's existing `g_thread`, so the reply appears threaded under her original message. From her perspective, Bob "finally got back to her" — the claim is invisible.
- **Unknown *user* on a known federated domain is NOT this flow.** If Alice sends to `nosuchbob@example.com` and `example.com` federates but has no such recipient, the destination returns `404 no_recipients` (RFC §6.3); Alice gets a local bounce (RFC §6.4). No shadow is ever created.

#### 14.7 Edge cases

- **Two registrars hold shadows for the same external email.** Each received an envelope from its own sender-base and created a local shadow. Neither is wrong. When one transitions to `active` or `forwarder`, §16.4 pulls in the peers' pending shadows via the consolidation path; overlap between backfills is deduped by `(origin_domain, envelope_id)` (§5.4) and by thread correlation on `federation_id`.
- **Sender spoofing of offers.** The offer (§14.4) is signed by the purported old-host registrar's domain key, pinned via TOFU (§5). A third party cannot inject fake offers.
- **Bob does not exist on unknown.com yet at the moment of claim.** Offers queue on unknown.com keyed by `external_email`; when the local account is first created there via normal onboarding, pending offers are surfaced at login.
- **Key rotation of the registrar between shadow creation and claim.** Not a problem — the shadow's creation does not bind to the registrar's key at creation time; the offer is signed fresh at offer time with whatever key is current, and the claim signature from the new host is what the registrar verifies against the new host's pinned key.
- **Replay of a claim.** The registrar stores `(offer_id, claim_id)` and refuses duplicates.
- **SMTP invite to the external address bounces.** The shadow stays; the invite is flagged undeliverable and not re-sent on subsequent messages. A domain that later federates can still trigger the §14.4 claim flow — the invite was a convenience for the recipient, not a prerequisite for claim.
- **Shadow creation on the registrar when no Default Registrar is reachable.** A non-registrar sender whose Default Registrar is down for longer than the retry budget (§10) generates a local bounce to Alice; it does **not** fall back to holding the shadow locally. Centralizing shadows on registrars is a hard rule, not a best-effort one.

#### 14.8 What this adds to the schema

- No new `shadow_alias` table — shadows are rows in the §15.2-extended `alias` table with `state = 'tentative'`.
- One new inbound endpoint on every registrar: `POST /api/v1/federation/shadow-offer` (§14.4).
- One new inbound endpoint on every registrar: `POST /api/v1/federation/claim` (§14.4).
- One new `email_template` row: `shadow_onboarding_invite` (§14.2.1).
- No changes to `g_thread` / `g_message` — backfill uses the existing inbound envelope with two optional fields.

The federation protocol remains additive: a peer that does not implement §14 (i.e. is not a registrar) still federates normally for reachable domains and bounces for unreachable ones. Shadow-holding is strictly a registrar capability, composed on top of §15 rather than layered underneath it.

### 15. Real-email addressing

Every user account — on any domain — has a verified external `real_email` (a regular mailbox like `alice@gmail.com`). The field serves two operational jobs on every domain — receiving notifications and logging in — and one addressing job — being the lookup key senders can use instead of the canonical — that is handled only by domains that opt in as a **registrar**. §15 covers the single-registrar mechanics; §16 covers multiple registrars.

#### 15.1 The registrar flag

A federated domain is a registrar or it is not. The capability is declared as a boolean in `config/domain.php`:

```
'registrar' => true,   // default: false
```

A non-registrar domain (example.com, acme.io): users have canonical addresses like `bob@example.com`. Their `real_email` is operational only — they log in with it and receive notifications at it, but senders cannot address them by it.

A registrar (103mail.com): all of the above, plus it accepts real_emails as recipients. A sender addressing `bob_real@gmail.com` hits a lookup (§15.5) and is either resolved to an existing canonical, forwarded to a federated home (§15.8), or auto-onboarded into a new local mailbox (§15.5 step 3).

A registrar is a full federated domain; `registrar: true` is an added capability, not a replacement. The flag does not change at runtime; promoting a domain to registrar later requires a real_email backfill.

#### 15.2 Schema

All domains extend `user`:

```
user.real_email              VARCHAR NULLABLE
user.real_email_verified_at  DATETIME NULLABLE
UNIQUE(domain_id, real_email)
```

Uniqueness is per-domain. Two domains may each hold a `real_email` record for the same external address, for the same person or different people — cross-domain reconciliation happens at claim time (§15.7, §16.4).

Registrars additionally extend `alias`:

```
alias.real_email         VARCHAR NULLABLE
alias.forward_canonical  VARCHAR NULLABLE       -- e.g. "bob@example.com"
alias.state              ENUM('active','tentative','forwarder','tombstone')
alias.issued_at          DATETIME NULLABLE
alias.expires_at         DATETIME NULLABLE      -- TTL for registrar records
UNIQUE(domain_id, real_email)
```

An `alias` on a registrar is one of:

- **Active** — a normal user's canonical, with `real_email` set for lookup.
- **Tentative** — an auto-onboarded shadow awaiting claim (§15.5 step 3).
- **Forwarder** — a real_email that a federated-domain user has claimed; the registrar routes sends to their canonical (§15.8).
- **Tombstone** — a historical pointer (no mailbox, no routing).

#### 15.3 Registration (all domains)

Signup is gated on real_email verification (double opt-in):

1. User enters local-part + real_email at `mail.<domain>/signup`.
2. Backend creates `user` with `real_email_verified_at = NULL`. On a registrar the canonical is the random 7-digit handle (§13.5); on a non-registrar it's whatever local-part the user chose.
3. SMTP verification link to real_email.
4. On click: `real_email_verified_at = NOW()`. Account becomes usable.

Unverified accounts cannot send or receive. Users may log in with either their canonical or their real_email as username; both are domain-scoped.

#### 15.4 Compose — non-registrar (the Default Registrar)

Non-registrars do no real-email lookup themselves, and they do not hold shadows locally (§14.1). Instead, every non-registrar Domain has exactly one **Default Registrar**: a single federated Domain — chosen at config time by the non-registrar's operator — that receives every send the non-registrar cannot deliver itself, and that holds shadows and notifies external recipients on the non-registrar's behalf.

**The Default Registrar invariant.** Every non-registrar Domain **MUST** have a Default Registrar configured. Boot fails fast if it is missing. This is what makes the fan-out below total — there is no "no registrar available" branch to reason about, at runtime or in code.

**Config shape:**

```
'default_registrar' => [
    'domain'     => '103mail.com',
    'pinned_key' => 'ed25519:...',
],
'fallback_registrars' => [                 # optional; empty by default
    ['domain' => 'mailhub.org',     'pinned_key' => 'ed25519:...'],
],
```

`default_registrar` is a single entry, not a list, because the designation is singular: it's the one Domain that becomes the home for this non-registrar's tentative shadows (§14.1). `fallback_registrars` exists only as a federation-retry safety net if the Default is unreachable beyond the normal retry window (§10); it never sees traffic when the Default is healthy, which prevents split-brain shadows for the same external address across multiple registrars for the same sender Domain.

A registrar Domain (`registrar: true`) satisfies the invariant on its own — it runs §15.5 locally and does not configure a Default Registrar.

**Fan-out for non-registrars:**

```
for each subscriber "X@Y":
    if Y ∈ local_domains:      local branch (unchanged)
    else if Y is federated:    federation branch (§4)
                               # includes the "unknown user on known domain"
                               # case, which 404s per RFC §6.3 and becomes
                               # a local bounce — no shadow, no registrar.
    else:                      registrar branch — federate to the
                               Default Registrar with recipient = "X@Y"
                               unchanged. The registrar runs §15.5,
                               which either resolves or creates a shadow
                               and emits the §14.2.1 SMTP invite.
```

Outbound federation to the Default Registrar reuses the §6 inbound envelope unmodified; the registrar is responsible for resolving or onboarding the non-federated recipient (§15.5). The sending Domain never fans out to more than one registrar for a given external address and **MUST** keep that selection stable across retries (RFC §8.2).

#### 15.5 Compose — registrar

When a registrar is asked to deliver to `X@Y` where `Y` is not a federated domain:

```
1. Local match:
     SELECT * FROM alias
      WHERE domain_id = active_domain AND real_email = 'X@Y'
        AND state IN ('active','forwarder') AND expires_at > NOW
     hit → §15.8 routing
2. Peer match:
     parallel GET /api/v1/registrar/lookup?hash=sha256(X@Y)
     on each registrar_peer (§16.3)
     hit → cache locally with TTL, §15.8 routing
3. Auto-onboard:
     alias(state='tentative', real_email='X@Y',
           issued_at=NOW, expires_at=NOW+90d)
     plus 7-digit canonical on this registrar
     deliver the message locally (the shadow mailbox, §14)
     emit the §14.2.1 `shadow_onboarding_invite` SMTP notification to X@Y
       (only on first creation — subsequent messages to the same
        tentative alias do not re-notify).
```

Tentative shadows do not answer `/lookup` as authoritative and do not announce to peers. They are registrar-local until claim.

#### 15.6 Claim

The SMTP invite links to `https://webmail.<registrar>/claim?token=...`. The token is single-use, signed, 30-day, bound to a specific tentative shadow.

The claim page offers two options:

**(a) Sign up here.** User sets a password. The tentative shadow is promoted in place: `user.real_email_verified_at = NOW()`, `alias.state = 'active'`. Canonical remains the 7-digit handle. §16.4 consolidation follows.

**(b) I have a federated account.** User enters a canonical, e.g. `bob@example.com`. Registrar redirects to `https://webmail.example.com/federated-claim?token=...&registrar=103mail.com`. example.com authenticates Bob, confirms his `user.real_email` matches the shadow's real_email, and posts a claim back to the registrar (§15.7). §16.4 consolidation follows.

#### 15.7 claim-by-real-email endpoint

Non-registrar → registrar, closing path (b):

```
POST https://webmail.103mail.com/api/v1/federation/claim-by-real-email
X-Federation-Sender:    example.com
X-Federation-Signature: ed25519(body)

{
  "version":    "0.1",
  "claim_id":   "{uuidv7}",
  "token":      "{shadow token from the invite}",
  "real_email": "bob@gmail.com",
  "canonical":  "bob@example.com"
}
```

Registrar verifies the signature, then:

1. Resolves `token` to a tentative shadow; aborts if already claimed or expired.
2. Confirms `real_email` matches the shadow.
3. Accepts example.com's assertion that `canonical` is a verified-real_email user on its domain. The cryptographic proof is the federation signature; SMTP ownership is the claiming domain's job to verify on its side, not the registrar's.
4. Updates the `alias`: `state = 'forwarder'`, `forward_canonical = 'bob@example.com'`, `issued_at = NOW()`.
5. Backfills the shadow's messages to example.com via §14.4 backfill envelopes, preserving `origin_sent_at` and `thread.federation_id`.
6. Tombstones the shadow's customer / user / mailbox / folder rows; keeps the `alias` as the forwarder.
7. Drives §16.4 consolidation with peer registrars.

Replay protection: `(token, claim_id)` stored, duplicates rejected.

#### 15.8 Routing after resolution

When §15.5 step 1 or 2 returns a hit:

- `state = 'active'` → deliver locally into the canonical's mailbox (existing local branch).
- `state = 'forwarder'` → build a §6 federation envelope to `forward_canonical`, with recipient rewritten from the real_email to the canonical. The sending user's view is unchanged; their Sent copy still records the original `X@Y`.

No shadow is recreated on subsequent sends. The forwarder record persists until expiry (re-issued on activity, §16.6) or re-claim.

#### 15.9 Notifications

`sent_email` sets SMTP `To:` to `user.real_email`. Under §15, every user has this field populated (gated on verification). No template change beyond the field reference; the notification remains metadata-only with a domain-hosted link (§7).

### 16. Multi-registrar coordination

When more than one registrar exists, users must see a consistent world — a lookup for `bob@gmail.com` reaches the same canonical regardless of which registrar answers, and a claim on one registrar does not strand orphan shadows on others. The protocol is minimal: registrars are ordinary federated peers that expose two extra endpoints and know each other via per-registrar config. No manifest, no quorum, no gossip.

#### 16.1 Registrars as federated peers

A registrar's `/.well-known/webmail-federation` adds one field:

```
{ ...§5 fields..., "registrar": true }
```

All other trust mechanics — key pinning, TOFU, signature verification — reuse §5/§6 unchanged. A registrar is discoverable to anyone; it becomes relevant to another registrar's `/lookup` logic only if it appears in that registrar's `registrar_peers` config:

```
'registrar_peers' => [
    ['domain' => 'mailhub.org',     'pinned_key' => 'ed25519:...'],
    ['domain' => 'example-mail.net','pinned_key' => 'ed25519:...'],
],
```

Adding or removing a peer is an ops change on each running registrar. Trust is bilateral and local, matching how federated domains already treat each other. A registrar not in your `registrar_peers` is not consulted on lookups and has no standing to drive shadow consolidation on your side.

#### 16.2 Two new endpoints

On registrars only:

```
GET /api/v1/registrar/lookup?hash={sha256_hex(lowercase(real_email))}
  → 200 { canonical, forward_canonical?, issued_at, expires_at }
  → 404 on no active/forwarder match

GET /api/v1/registrar/pending-shadows?hash={sha256_hex(lowercase(real_email))}
  X-Federation-Sender:    {peer_registrar_domain}
  X-Federation-Signature: ed25519(canonical_request)
  → 200 [ { alias_id, message_count, oldest_sent_at, newest_sent_at, summary: [...] } ]
  → 404 on no tentative shadows
```

`/lookup` is unauthenticated — the hash is already known to whoever asked. SHA-256 of the lowercased real_email, no salt in v1; a targeted lookup carries no less entropy than the address itself. Deployments wanting stricter privacy can run snapshot imports against trusted peers instead of live queries (out of scope for v1).

`/pending-shadows` is peer-authenticated — it surfaces the existence of unclaimed shadows, useful for claim consolidation but sensitive otherwise.

A registrar answers `/lookup` only from its own `alias` rows, never from its peer cache. Authority is single-sourced per record.

#### 16.3 Cross-registrar fallback at lookup

When §15.5 step 1 misses, the registrar queries every entry in `registrar_peers` in parallel (default timeout 2s):

```
for peer in registrar_peers:
    spawn: GET https://webmail.{peer.domain}/api/v1/registrar/lookup?hash=...
merge:
    hit → cache locally keyed by (hash, peer.domain)
           TTL = min(response.expires_at - NOW, 24h)
           return canonical / forward_canonical
    all 404 / timeout → miss, proceed to §15.5 step 3
```

Cache is per-querier and consulted only for outbound routing — a registrar's own `/lookup` is always answered from local `alias` rows, so authority does not leak across registrars.

#### 16.4 Claim consolidation

When a tentative shadow transitions to `active` or `forwarder` on registrar W (either via §15.6 path a or path b), W immediately drives consolidation:

```
for peer in registrar_peers:
    GET https://webmail.{peer.domain}/api/v1/registrar/pending-shadows?hash=... → peer_shadows
```

If any peer returns shadows, the claiming user sees a consolidated UI:

> Claimed `bob@gmail.com` on 103mail (7 messages).
> We also found pending messages addressed to you:
>   — mailhub.org: 3 messages, oldest 2026-03-02
>   — example-mail.net: 1 message, 2026-04-11
> [ Import all ]  [ Skip ]

On **Import all**, W posts to each peer's `POST /api/v1/federation/backfill` (the §14.4 endpoint, reused), signed by W, instructing the peer to backfill its tentative shadow's messages to the now-authoritative destination (either W's local canonical, or via forwarder to `forward_canonical`). Each peer tombstones after ack.

On **Skip**, peer shadows expire at 90 days and become eligible for fresh onboarding after.

Symmetry: whether W became authoritative via signup (§15.6 a) or federated claim (§15.6 b), consolidation is identical.

#### 16.5 Races and conflicts

**First-touch race.** Alice via registrar A and Carol via registrar B send to an unseen `bob@gmail.com` within seconds. Both §15.5 step 2 miss (no authoritative record yet), both fall through to step 3, both create tentative shadows and SMTP Bob. Bob's inbox receives two invites. He clicks one, say A's. A becomes authoritative; §16.4 consolidation pulls from B; B tombstones. No coordination was required during the race.

**Dual-authoritative.** Bob clicks both invites and signs up on both A and B. Both hold authoritative records. On lookup, each responder returns its own; the querier's cache merge uses latest `issued_at`, tiebroken by lexicographic responder_domain. Sends may land on either of Bob's canonicals — both are legitimately his. Rare and voluntary; accepted for v1.

**Rogue registrar.** A peer in your `registrar_peers` fabricates a `/lookup` response. Sends for the targeted real_email may be routed to a canonical the rogue controls, for domains that trust it. Defenses:

- Curated `registrar_peers` — small operator-vetted list; misbehavior surfaces as misrouted mail and the peer is removed in config.
- Federation envelope signatures — the rogue can receive mail at a canonical it controls but cannot impersonate the sender; reply paths go to the real sender's domain.
- Optional v2 hardening: record-level signatures on `/lookup` responses and signed revocation tombstones. Not required for correctness in v1 given the curated trust model.

DNS-resolver posture: each operator trusts the resolvers it configures; misbehavior is a configuration issue, not a protocol failure.

#### 16.6 Freshness and expiry

Authoritative records carry `expires_at`, default 90 days from `issued_at`. Past-expiry records stop being served, and the real_email becomes eligible for fresh onboarding on the next send. Re-issue happens implicitly whenever the user is active — any login or own-mail activity bumps `issued_at` on the canonical's alias. Deleted or abandoned accounts age out within the same window; sends inside that window may still reach a now-missing canonical, handled by the recipient's existing missing-user path.

No explicit revocation protocol. Re-issue-or-expire covers rotation, deletion, and migration. The trade is slightly longer tail-staleness in exchange for a flatter protocol surface.

#### 16.7 N=1 degenerate case

A single-registrar deployment (only 103mail) is this protocol with an empty `registrar_peers` list. `/lookup` is local-only, cross-registrar fallback finds no peers, consolidation queries zero endpoints. Every code path behaves correctly. Scaling to N=2 is purely operational — each side adds the other's domain + key to config.

---

## Migration steps, in order

1. **Client**: introduce `/config.json` + `DomainContext`, strip literal `103mail.com` from source.
2. **Backend**: move default domain to domain config; add `domain.backend_url` / `public_key` / `is_local`.
3. **Backend**: add well-known discovery route + keypair generation on first boot.
4. **Backend**: split `addMessageToSubscribers` into local/remote branches; add outbound `FederationClient` + `federation_outbox` table + retry worker.
5. **Backend**: add inbound `FederationApi` with signature verification.
6. **Spin up** a second domain (e.g. `example.com`) in staging, verify round-trip: compose from 103mail → inbox on example.com → reply federates back.
7. **Harden**: rate-limit inbound per peer, cap attachment size, bounce semantics (`federation_outbox.status = permanent_failure` generates a local bounce message to sender).
8. **Schema (all domains)**: `user.real_email` + `real_email_verified_at` + unique `(domain_id, real_email)`.
9. **Registration (all domains)**: require real_email at signup; SMTP double-opt-in; gate account activation on verification.
10. **Notifications (all domains)**: wire `sent_email` `To:` to `user.real_email`.
11. **Schema (registrars)**: `alias.real_email`, `alias.forward_canonical`, `alias.state`, `alias.issued_at`, `alias.expires_at`.
12. **Compose (non-registrars)**: add `default_registrar` (required) and optional `fallback_registrars` config; extend §4 fan-out with the registrar branch — federate non-federated recipients to the Default Registrar (§15.4). Boot fails if `default_registrar` is missing on a non-registrar Domain.
13. **Compose (registrars)**: implement §15.5 — local match, peer match, auto-onboarding with tentative shadow + SMTP invite + claim page (both signup and federated-claim paths).
14. **Federation (registrars)**: `POST /api/v1/federation/claim-by-real-email`; reuse §14.4 backfill for messages moving out of a shadow.
15. **Multi-registrar (registrars)**: expose `/api/v1/registrar/lookup` and `/api/v1/registrar/pending-shadows`; add `registrar_peers` config; cross-registrar query on lookup miss; consolidation on claim.
16. **Staging test**: two-registrar + one-non-registrar deployment. Verify (a) non-registrar's send to unseen real_email resolves via registrar and auto-onboards; (b) race between two registrars converges on a single mailbox after claim via consolidation; (c) federated-account claim produces a stable forwarder, and subsequent sends route directly without creating new shadows.

---

## Open questions

- **Reply threading across domains**: `g_thread.id` is a local autoincrement. Need a federated stable ID — propose `g_thread.federation_id` (UUID, generated on origin, carried in envelope, matched on inbound replies).
- **Account portability**: if `user@example.com` wants to move to `acme.io`, do we keep the old address forwarding? Out of scope for v1 — document as "no".
- **Spam / abuse**: TOFU pubkey pinning trusts any domain that has DNS. Add a domain-level knob (`federation.accept_from: any | allowlist`) for private deployments.
- **Real-email rotation latency**: §16.6 relies on re-issue-or-expire (TTL 90d). A user changing `real_email` on their non-registrar home is reflected at registrars only after the next claim; old forwarders may route stale for up to the TTL. v2 could add a proactive `POST /federation/real-email-change` notification to shorten the window. Out of scope v1.
- **Record-level signatures on `/registrar/lookup`**: v1 trusts the HTTPS response from a `registrar_peers`-listed peer. v2 hardening would sign each record so cached entries survive peer removal with attributable provenance. Not required for correctness given the curated-peer model.
- **Registrar admission**: `registrar_peers` is curated per-operator; there is no global registrar directory. Adding a new registrar requires each existing registrar's operator to add its domain + key to their own config. This is intentionally a governance-free, config-level choice — the tradeoff is that a new registrar is not immediately visible network-wide. Acceptable while N stays small.
