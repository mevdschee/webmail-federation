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

### 1. Deployment topology

```
mail.103mail.com    ──┐
mail.example.com    ──┼──►  static bundle of webmail-client (CDN)
mail.acme.io        ──┘

api.103mail.com     ◄──►  webmail backend (103mail tenant)
api.example.com     ◄──►  webmail backend (example.com tenant)
api.acme.io         ◄──►  webmail backend (acme.io tenant)
```

One client bundle, N backends. Each domain gets its own backend deployment + DB. They exchange mail via **federation HTTP**, not SMTP.

### 2. Client: drop the domain hardcode

Single bundle, per-domain runtime config resolved from the hostname:

- Add `/config.json` served by each backend (or inlined at build of a per-tenant `index.html`):
  ```json
  {
    "domain": "103mail.com",
    "apiBase": "https://api.103mail.com/api/v1",
    "brand": {
      "name": "103mail",
      "logo": "/103mail.svg",
      "marketingUrl": "https://103mail.com"
    }
  }
  ```
- Client bootstraps by fetching `/config.json` before first render; passes the result through a `TenantContext`.
- Replace hardcoded `/103mail.svg`, `https://103mail.com` link, and `@103mail.com` mock strings with `tenant.brand.*` / `tenant.domain`.
- Login form: local part only, with `@{tenant.domain}` shown as an adornment — reinforces that each hostname is a single tenant.
- `AuthContext` stores `{ token, email, domain }`; token scoped per-tenant in `localStorage` keyed by domain.

### 3. Backend: tenant config, not implicit singleton

- New `config/tenant.php` (or env): `local_domain = "103mail.com"`, `federation_private_key`, `federation_public_key`.
- Change `EmailClient::getDefaultDomain()` to read tenant config rather than the "customer_id IS NULL" row.
- Extend the `domain` table:
  ```
  domain.backend_url      VARCHAR   -- NULL = local
  domain.public_key       TEXT      -- for verifying inbound
  domain.is_local         BOOLEAN   -- this server hosts the domain
  domain.last_seen_at     DATETIME  -- discovery cache timestamp
  ```
- Retire the "one row where customer_id IS NULL" convention; the single self-row is marked `is_local = true`, other domains are populated lazily via discovery.

### 4. Fan-out split: local vs remote recipients

`EmailClient::addMessageToSubscribers` becomes:

```
for each subscriber email:
    domain = parse(email)
    if domain == tenant.local_domain  or  domain.is_local == true:
        existing path: addCustomer-if-missing + insert thread/message rows
    else:
        resolve peer backend for `domain` (discovery, §5)
        POST signed envelope to peer /api/v1/federation/inbound
        record outbound attempt in a new `federation_delivery` table
            (id, g_message_id, peer_domain, status, attempts,
             last_error, next_retry_at)
        background worker retries with exponential backoff
```

Invariant: `g_thread` / `g_message` / `g_attachment` are written once locally so the sender keeps their Sent copy. Only the per-recipient `thread`/`message` insert is replaced by a network hop when the recipient is remote.

### 5. Peer discovery + trust

- Each backend exposes `/.well-known/webmail-federation`:
  ```json
  {
    "domain": "example.com",
    "backend": "https://api.example.com",
    "inbound": "https://api.example.com/api/v1/federation/inbound",
    "public_key": "ed25519:..."
  }
  ```
- First time 103mail needs to talk to `example.com`: HTTP `GET https://example.com/.well-known/webmail-federation`, pin the pubkey into the `domain` row, cache for 24h.
- No CA needed — TOFU + well-known, same shape as Matrix server-to-server.

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

Currently `sent_email` links back to the backend that ran the code. That stays correct post-federation because notifications are generated by the **recipient's** backend after inbound delivery, not the sender's. Only change: the notification template's base URL pulls from tenant config rather than a constant.

### 8. Auto-created aliases stay local

When `addCustomer` runs for a remote recipient today, it creates a local mailbox — that was the single-domain shortcut and is wrong under federation. Under the new design, remote recipients are pushed over federation; the local DB only gains a `contact` row so the sender sees them in autocomplete.

### 9. Compose UI change (client)

The `POST /thread/{folderId}` payload is unchanged — `subscribers` is still a list of emails. Federation is invisible to the UI. The only client change is the tenant-config extraction in §2.

### 10. Walkthrough: sending a cross-domain message

Alice (`1234567@103mail.com`) composes a message to Bob (`bob@example.com`). Both domains are served by separate Webmail backends. The sequence:

**Step A — client → origin backend.** Alice's browser posts the usual compose payload to her own backend:

```
POST https://api.103mail.com/api/v1/email/thread/{folderId}
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
GET https://example.com/.well-known/webmail-federation
```

The returned `public_key` is pinned into the `domain` row on TOFU (§5). Cached for 24h.

**Step E — origin backend enqueues the envelope.** A row is inserted into `federation_delivery`:

```
(id, g_message_id, peer_domain = "example.com",
 status = "pending", attempts = 0, next_retry_at = NOW())
```

The actual HTTP call is done by a background worker, so Alice's compose HTTP request returns immediately (the message is already in her Sent folder; federation is async).

**Step F — worker signs and posts the envelope.** The worker picks up the pending row and sends:

```
POST https://api.example.com/api/v1/federation/inbound
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

On `202 Accepted`, `federation_delivery.status` flips to `delivered`. On transient failure (5xx, 429), the retry clock starts (30s → 1h with jitter, 48h budget). On permanent failure (4xx), a local bounce message is inserted into Alice's inbox.

**Step G — destination backend verifies and delivers.** The example.com backend:

1. Verifies the Ed25519 signature against its cached pubkey for `103mail.com`.
2. Checks `envelope.origin_domain == X-Federation-Sender == domain-of(sender_email)`.
3. Dedupes on `(origin_domain, envelope_id)`.
4. Writes local `g_thread` / `g_message` rows, carrying over the `federation_id` values so replies can correlate.
5. Calls the **local** branch of `addMessageToSubscribers` for `bob@example.com`: if Bob has no account, the existing auto-onboarding creates `customer` + `user` + `mailbox` + `folder` + `alias` — but now with an `@example.com` alias, because this backend's tenant config says its local domain is `example.com`.
6. Inserts Bob's per-user `thread` / `message`, unread.
7. Enqueues a `sent_email` notification linking to `https://mail.example.com/...` — **Bob's home backend**, not Alice's.

Bob receives the notification email, clicks the link, logs into his example.com mailbox, and reads the message. 103mail.com never touches Bob's credentials or mailbox contents.

### 11. Walkthrough: replying across domains

Bob opens the thread and hits Reply. The reply travels back along the same rails, mirrored.

**Step A — Bob's client → example.com backend.** The client posts a reply to example.com's backend:

```
POST https://api.example.com/api/v1/email/message/{threadId}
Authorization: Bearer {bob's token}

{ "body": "Thanks Alice — ...", "attachments": [] }
```

`{threadId}` here is Bob's **local** per-user `thread.id` on example.com (an autoincrement). The frontend doesn't see federation ids.

**Step B — example.com backend writes locally.** `EmailClient::addMessage` appends a new `g_message` to the **same** `g_thread` Bob received in §10 Step G. Because that `g_thread` has a `federation_id` stored from inbound delivery, example.com knows this thread is federated. The new `g_message` gets its own fresh `federation_id`.

Bob's per-user Sent copy is written, unread-for-others.

**Step C — example.com backend walks the subscriber list.** The `g_thread` has subscribers `{Alice, Bob}`. Iterating:

- `bob@example.com` → local, no network.
- `1234567@103mail.com` → domain `103mail.com`, not `is_local` on this backend → outbound federation branch.

**Step D — example.com enqueues and posts to 103mail.** A `federation_delivery` row is inserted on the example.com side (the table lives on whichever server is the origin of a given envelope). The worker posts:

```
POST https://api.103mail.com/api/v1/federation/inbound
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
6. Enqueues a `sent_email` notification to Alice linking to `https://mail.103mail.com/...`.

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
| `federation_delivery` | one row (outbound M1 to example.com)    | one row (outbound M2 to 103mail)             |

The federated identifiers `T`, `M1`, `M2` are identical across both databases; the autoincrement local ids are not. This is what makes reply correlation and idempotent re-delivery possible.

### 13. Hosting multiple domains on one server

Nothing in the wire protocol requires that one domain equal one server. A single Webmail backend deployment **MAY** be authoritative for several domains at once (e.g. one operator runs `103mail.com`, `example.com`, and `acme.io` on the same host and database). The federation RFC still sees each domain as a distinct logical Server; the physical process just serves several of them.

#### 13.1 What changes in tenant config

The single-valued `tenant.local_domain` from §3 becomes a set. `config/tenant.php` (or the equivalent env-backed loader) now returns:

```
local_domains: [
  {
    "domain":          "103mail.com",
    "hostname":        "mail.103mail.com",
    "api_hostname":    "api.103mail.com",
    "brand":           { "name": "103mail",  "logo": "/103mail.svg",
                         "marketing_url": "https://103mail.com" },
    "federation_private_key": "ed25519:...",
    "federation_public_key":  "ed25519:...",
  },
  {
    "domain":          "example.com",
    "hostname":        "mail.example.com",
    "api_hostname":    "api.example.com",
    "brand":           { "name": "Example Mail", "logo": "/example.svg",
                         "marketing_url": "https://example.com" },
    "federation_private_key": "ed25519:...",
    "federation_public_key":  "ed25519:...",
  }
]
```

Each domain has its **own** federation keypair — never shared, because remote peers pin keys per domain (§5). Compromise of one hosted domain's key must not let an attacker impersonate the others.

#### 13.2 Request routing: the Host header selects the tenant

Because the browser always knows the user as "the person at `mail.example.com`", the backend picks the active domain from the incoming HTTP `Host` header:

- `GET https://mail.103mail.com/config.json` → 103mail's brand block.
- `GET https://mail.example.com/config.json` → example.com's brand block.
- `POST https://api.103mail.com/api/v1/email/thread/...` → operates inside the 103mail tenant.
- `POST https://api.example.com/api/v1/email/thread/...` → operates inside the example.com tenant.

A lightweight middleware at the top of request handling resolves `Host` → one entry in `local_domains`, stores it in request state as `$activeTenant`, and rejects (404) any hostname not in the config. Every downstream call that used to reach for a singleton default domain now takes `$activeTenant` instead.

#### 13.3 Each hosted domain gets its own well-known endpoint

The same backend process serves:

- `https://103mail.com/.well-known/webmail-federation` → `{domain: "103mail.com", public_key: "ed25519:K_103", ...}`
- `https://example.com/.well-known/webmail-federation` → `{domain: "example.com", public_key: "ed25519:K_ex", ...}`

Selection is again `Host`-based. Peer servers never learn that these two domains share a process.

#### 13.4 User and mailbox identity stays domain-local

- A `customer` and a `user` row belong to exactly one hosted domain. The association is recorded as `user.domain_id` (new column) — there is no "customer that spans domains". A person who wants accounts on both `103mail.com` and `example.com` has two `user` rows and two `customer` rows.
- Aliases are already keyed by unique `email`; `alice@103mail.com` and `alice@example.com` coexist without schema change.
- Login is scoped by the active tenant: the same local-part (`alice`) can log into `mail.103mail.com` and `mail.example.com` and reach different mailboxes. Bearer tokens are issued with the domain embedded, and rejected if used against a different tenant.

#### 13.5 Auto-onboarding uses the recipient's domain, not a global default

Today `addCustomer` creates a random `{7digit}@{default_domain}` alias. Under multi-domain hosting, it derives the alias domain from the recipient's email:

```
email       = "foo@example.com"
domain      = "example.com"
# Is this domain hosted here?
if domain in local_domains:
    alias_local_part = random_int(1000000, 9999999)
    alias            = f"{alias_local_part}@{domain}"
    create customer + user + mailbox + folder + alias
    # user.domain_id = id of example.com row
else:
    # Not local — federation branch (§4), no onboarding.
```

The random-local-part trick for privacy is preserved; only the domain suffix is now dynamic.

#### 13.6 Delivery between two locally hosted domains short-circuits federation

When Alice at `103mail.com` sends to Bob at `example.com` **and both are hosted by the same backend**, the fan-out loop in §4 takes the local branch for both recipients. No envelope is built, no HTTP is emitted, no signature is computed, no `federation_delivery` row is written. The exchange is a pure in-process database operation — exactly as it would be for two users on the same domain.

That property follows from the fan-out rule in §4 unchanged: membership in `local_domains` is the only test. A future operator who splits `example.com` onto its own separate backend flips one row (`domain.is_local = false`, `domain.backend_url = "https://api.example.com"`), and from that point the same send automatically uses federation instead. The sending code path is identical either way; only the branch taken at dispatch time differs.

#### 13.7 Notifications pick the right branded host

The `sent_email` template's base URL now resolves from the **recipient's** domain config, not from a single tenant default. A notification destined for `bob@example.com` points at `https://mail.example.com/...`, and one destined for `alice@103mail.com` points at `https://mail.103mail.com/...` — even when both are generated by the same backend in the same request.

#### 13.8 What is not shared across hosted domains

- **Keys** — separate Ed25519 keypair per domain (§13.1).
- **Brand assets** — separate logo, marketing URL, display name (§2, §13.1).
- **User accounts** — no cross-domain single sign-on (§13.4).
- **Tokens** — issued against one tenant, rejected against others.

#### 13.9 What is shared across hosted domains

- **The codebase and the running process** — one PHP deployment handles N domains.
- **The database schema and, optionally, the physical database** — `domain`, `customer`, `user`, `alias`, etc. are reused; rows are scoped by `domain_id` / `customer_id`. Operators who want stricter blast-radius isolation can still deploy one database per hosted domain without any code change; the tenant loader just points each `Host` at a different DSN.
- **The outbound federation worker and the inbound federation endpoint** — they simply loop over `local_domains` when signing outbound envelopes (using each envelope's `origin_domain` to pick the right key) and accept inbound traffic for any hosted domain.

The federation protocol does not change. The operator gains a deployment choice: N domains on one process for low-volume / small tenants, or one domain per process for isolation, performance, or compliance — with the same code, same schema, same wire format.

---

## Migration steps, in order

1. **Client**: introduce `/config.json` + `TenantContext`, strip literal `103mail.com` from source.
2. **Backend**: move default domain to tenant config; add `domain.backend_url` / `public_key` / `is_local`.
3. **Backend**: add well-known discovery route + keypair generation on first boot.
4. **Backend**: split `addMessageToSubscribers` into local/remote branches; add outbound `FederationClient` + `federation_delivery` table + retry worker.
5. **Backend**: add inbound `FederationApi` with signature verification.
6. **Spin up** a second tenant (e.g. `example.com`) in staging, verify round-trip: compose from 103mail → inbox on example.com → reply federates back.
7. **Harden**: rate-limit inbound per peer, cap attachment size, bounce semantics (`federation_delivery.status = permanent_failure` generates a local bounce message to sender).

---

## Open questions

- **Reply threading across domains**: `g_thread.id` is a local autoincrement. Need a federated stable ID — propose `g_thread.federation_id` (UUID, generated on origin, carried in envelope, matched on inbound replies).
- **Account portability**: if `user@example.com` wants to move to `acme.io`, do we keep the old address forwarding? Out of scope for v1 — document as "no".
- **Spam / abuse**: TOFU pubkey pinning trusts any domain that has DNS. Add a tenant-level knob (`federation.accept_from: any | allowlist`) for private deployments.
