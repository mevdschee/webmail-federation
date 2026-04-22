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

### 14. Shadow mailboxes for unreachable domains

When Alice sends to `bob@unknown.com` and `unknown.com` has no federation endpoint, the message cannot be pushed. Three behaviors are possible: bounce immediately (SMTP-style), bounce after a retry budget, or **hold the message in a local shadow mailbox until the domain is reachable**. We take the third — it gives the sender a "delivered" experience, preserves history across the gap, and later produces a clean hand-off when the remote domain joins the federation.

The cost is identity management: a shadow is an unclaimed placeholder mailbox for a person who does not (yet) control any account on this server. That cost is paid by the **claim protocol** (§14.4), which transfers shadow history to the real mailbox over federation once the domain becomes reachable.

#### 14.1 The shadow_alias table

A new table maps external addresses to local placeholder users, separately from the regular `alias` table:

```
shadow_alias (
  id,
  external_email    VARCHAR  UNIQUE,   -- "bob@unknown.com"
  user_id           FK user,           -- the local shadow user
  created_at,
  claimed_at        NULLABLE,          -- set when §14.4 completes
  claimed_by_domain NULLABLE           -- the domain that federated the claim
)
```

The referenced `user` row is a normal row in a normal `customer` — the only thing unusual is that it has no password, no login history, and its single `alias` is the random `{7digit}@{hosting_domain}` handle (§13.5). Nobody can log in; the mailbox accumulates messages.

**Which hosting domain?** On a multi-domain deployment (§13), the shadow is created under the **sender's** active tenant. Alice on 103mail triggers `1234567@103mail.com`; an Example Mail user triggering the same address would get a separate `7654321@example.com` shadow. This keeps blast-radius scoped, and it mirrors the existing §13.5 rule that auto-onboarding uses the active tenant.

Yes, this means two shadows may exist for the same `bob@unknown.com` if two hosted domains each had a sender reach out. That is fine; both get claimed independently in §14.4.

#### 14.2 Onboarding trigger: fan-out fall-through

The §4 fan-out loop gains a third branch **after** local and remote:

```
for each subscriber email:
    domain = parse(email)
    if domain in local_domains:
        local branch (unchanged)
    else if domain.is_local == false and domain.backend_url is set:
        remote branch (unchanged — federation push)
    else:
        # domain is unknown OR discovery has failed
        shadow branch:
            row = shadow_alias.find_or_create(external_email = email,
                                              tenant         = active_tenant)
            deliver locally into row.user_id's Inbox
            (original To: header preserved — no rewriting)
```

"Discovery has failed" means: a `/.well-known/webmail-federation` probe has been attempted and returned 404 / connection refused / TLS failure / invalid JSON. Transient failures (timeouts, 5xx) do **not** trigger the shadow branch; they keep the envelope in `federation_delivery` with `status = pending` and retry. The shadow branch is only for the terminal "this domain does not participate".

Alice's Sent folder still shows `bob@unknown.com` as the recipient. The shadow alias is never surfaced in her UI.

#### 14.3 Discovery promotion

A background worker re-probes every domain with `domain.backend_url IS NULL` and at least one unclaimed `shadow_alias`. Frequency: low (hourly is fine). When a probe succeeds:

1. The `domain` row is promoted: `backend_url`, `public_key`, `last_seen_at` populated.
2. An **offer** envelope is sent to the newly-federated peer (§14.4), one per external email that has a shadow on our side.
3. Until the peer responds, new sends to that address use the normal remote branch (the domain is reachable now). Old messages stay in the shadow; they are only moved by a successful claim.

Promotion can also happen reactively: if we receive a well-formed inbound envelope from `unknown.com`, that is proof the domain is federated. We promote and then emit the offer envelopes.

#### 14.4 The claim protocol

Claim is a **two-party, federation-signed** handshake. There is no email verification, no out-of-band code — control of `bob@unknown.com` is proven by the fact that `unknown.com`'s authoritative federation backend signs the claim.

##### Offer (old-host → new-host)

When 103mail discovers that `unknown.com` is now federated and has shadow(s), it emits one envelope per shadow to the new peer:

```
POST https://api.unknown.com/api/v1/federation/shadow-offer
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
POST https://api.103mail.com/api/v1/federation/claim
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

The signature over this body by `unknown.com`'s pinned key is the sole proof of control. 103mail verifies, looks up the `shadow_alias`, and proceeds to backfill.

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

1. 103mail sets `shadow_alias.claimed_at = NOW()`, `claimed_by_domain = "unknown.com"`.
2. The shadow user / customer / alias / folder rows are **soft-deleted** (tombstone, not dropped — audit trail).
3. `shadow_alias` itself is retained as a historical pointer; future lookups for `bob@unknown.com` miss it (the domain is now federated, normal remote branch applies).

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

- **During shadow hold:** Alice sees her message in Sent as usual, recipient `bob@unknown.com`. No bounce, no "pending" indicator. If she sends again, the shadow_alias lookup short-circuits — the new message joins the same shadow mailbox.
- **At claim time:** Alice sees nothing. The backfill is entirely a 103mail → unknown.com internal matter.
- **After claim, on reply:** Bob replies from `bob@unknown.com`. The reply arrives at Alice via the normal §11 path; the `thread.federation_id` matches Alice's existing `g_thread`, so the reply appears threaded under her original message. From her perspective, Bob "finally got back to her" — the claim is invisible.

#### 14.7 Edge cases

- **Two operators host shadows for the same external email.** Each emits its own offer; Bob can accept both, and his backend receives two backfills that may or may not overlap. Thread correlation by `federation_id` handles any overlap gracefully (dedupe on `(origin_domain, envelope_id)` from §6).
- **Sender spoofing of offers.** The offer is signed by the purported old-host's domain key, pinned via the same TOFU rules as regular federation (§5). A third party cannot inject fake offers.
- **Bob does not exist on unknown.com yet.** Offers queue on unknown.com keyed by `external_email`; when the local account is first created (via normal onboarding), pending offers are surfaced at login.
- **Key rotation of the old host between shadow creation and claim.** Not a problem — shadow creation does not cryptographically bind to the old-host's key at creation time; the offer is signed fresh at offer time with whatever key is current, and the claim signature from the new host is what 103mail verifies against the new host's pinned key.
- **Replay of a claim.** 103mail stores `(offer_id, claim_id)` and refuses duplicates.

#### 14.8 What this adds to the schema

- `shadow_alias` table (§14.1).
- One new outbound endpoint on every peer: `POST /api/v1/federation/shadow-offer`.
- One new outbound endpoint on every peer: `POST /api/v1/federation/claim`.
- No changes to `g_thread` / `g_message` — backfill uses the existing inbound envelope with two optional fields.

The federation protocol remains additive: a peer that does not implement §14 still federates normally for reachable domains; shadows simply never claim out to it, which is the same behavior as "the domain never federates". Claim is a pure capability upgrade.

### 15. Real-email addressing

Every user account — on any tenant — has a verified external `real_email` (a regular mailbox like `alice@gmail.com`). The field serves two operational jobs on every tenant — receiving notifications and logging in — and one addressing job — being the lookup key senders can use instead of the canonical — that is handled only by a specific tenant role called a **registrar**. §15 covers the single-registrar mechanics; §16 covers multiple registrars.

#### 15.1 Two tenant roles

A federated tenant is **plain** or **registrar**. The role is declared in `config/tenant.php`:

```
'role' => 'registrar',   // or 'plain' (default)
```

A **plain** tenant (example.com, acme.io): users have canonical addresses like `bob@example.com`. Their `real_email` is operational only — they log in with it and receive notifications at it, but senders cannot address them by it.

A **registrar** (103mail.com): all of the above, plus it accepts real_emails as recipients. A sender addressing `bob_real@gmail.com` hits a lookup (§15.5) and is either resolved to an existing canonical, forwarded to a federated home (§15.8), or auto-onboarded into a new local mailbox (§15.5 step 3).

A registrar is a full federated tenant; the role is an added capability, not a replacement. The role does not change at runtime; promoting a plain tenant later requires a real_email backfill.

#### 15.2 Schema

All tenants extend `user`:

```
user.real_email              VARCHAR NULLABLE
user.real_email_verified_at  DATETIME NULLABLE
UNIQUE(domain_id, real_email)
```

Uniqueness is per-tenant. Two tenants may each hold a `real_email` record for the same external address, for the same person or different people — cross-tenant reconciliation happens at claim time (§15.7, §16.4).

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
- **Forwarder** — a real_email that a federated-tenant user has claimed; the registrar routes sends to their canonical (§15.8).
- **Tombstone** — a historical pointer (no mailbox, no routing).

#### 15.3 Registration (all tenants)

Signup is gated on real_email verification (double opt-in):

1. User enters local-part + real_email at `mail.<tenant>/signup`.
2. Backend creates `user` with `real_email_verified_at = NULL`. On a registrar the canonical is the random 7-digit handle (§13.5); on a plain tenant it's whatever local-part the user chose.
3. SMTP verification link to real_email.
4. On click: `real_email_verified_at = NOW()`. Account becomes usable.

Unverified accounts cannot send or receive. Users may log in with either their canonical or their real_email as username; both are tenant-scoped.

#### 15.4 Compose — plain tenant

Plain tenants do no real-email lookup themselves. The §4 fan-out gains one branch at the end:

```
for each subscriber "X@Y":
    if Y ∈ local_domains:      local branch (unchanged)
    else if Y is federated:    federation branch (§4)
    else:                      registrar branch — federate to configured registrar
                               with recipient = "X@Y" unchanged.
```

Plain tenants point at one or more registrars in `config/tenant.php`:

```
'registrars' => [
    ['domain' => '103mail.com',     'pinned_key' => 'ed25519:...'],
    ['domain' => 'mailhub.org',     'pinned_key' => 'ed25519:...'],
],
```

**Invariant:** every federated server must be able to resolve an unknown real_email. A tenant with `role = 'registrar'` satisfies this on its own (it runs §15.5 locally). A tenant with `role = 'plain'` satisfies it by configuring at least one entry in `registrars` — boot fails fast if the list is empty. Enforcing this at config load keeps the §15.4 fan-out total: there is no "no registrar available" branch to reason about.

Outbound federation to a registrar reuses the §6 inbound envelope unmodified; the registrar is responsible for resolving or onboarding the non-federated recipient (§15.5). The sending tenant never fans out to more than one registrar. The **first entry** in `registrars` is the tenant's designated auto-registration home: all unknown-recipient sends go there, so any tentative shadow created for this tenant's users lands on a single, predictable registrar. Additional entries exist only as federation-retry fallbacks if the first is unreachable after the normal retry window (§10); they never see traffic otherwise, which prevents split-brain shadows across registrars for the same sender tenant.

#### 15.5 Compose — registrar

When a registrar is asked to deliver to `X@Y` where `Y` is not a federated domain:

```
1. Local match:
     SELECT * FROM alias
      WHERE domain_id = active_tenant AND real_email = 'X@Y'
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
     deliver the message locally; SMTP invite to X@Y
```

Tentative shadows do not answer `/lookup` as authoritative and do not announce to peers. They are registrar-local until claim.

#### 15.6 Claim

The SMTP invite links to `https://mail.<registrar>/claim?token=...`. The token is single-use, signed, 30-day, bound to a specific tentative shadow.

The claim page offers two options:

**(a) Sign up here.** User sets a password. The tentative shadow is promoted in place: `user.real_email_verified_at = NOW()`, `alias.state = 'active'`. Canonical remains the 7-digit handle. §16.4 consolidation follows.

**(b) I have a federated account.** User enters a canonical, e.g. `bob@example.com`. Registrar redirects to `mail.example.com/federated-claim?token=...&registrar=103mail.com`. example.com authenticates Bob, confirms his `user.real_email` matches the shadow's real_email, and posts a claim back to the registrar (§15.7). §16.4 consolidation follows.

#### 15.7 claim-by-real-email endpoint

Plain tenant → registrar, closing path (b):

```
POST https://api.103mail.com/api/v1/federation/claim-by-real-email
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
3. Accepts example.com's assertion that `canonical` is a verified-real_email user on its tenant. The cryptographic proof is the federation signature; SMTP ownership is the claiming tenant's job to verify on its side, not the registrar's.
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

`sent_email` sets SMTP `To:` to `user.real_email`. Under §15, every user has this field populated (gated on verification). No template change beyond the field reference; the notification remains metadata-only with a tenant-hosted link (§7).

### 16. Multi-registrar coordination

When more than one registrar exists, users must see a consistent world — a lookup for `bob@gmail.com` reaches the same canonical regardless of which registrar answers, and a claim on one registrar does not strand orphan shadows on others. The protocol is minimal: registrars are ordinary federated peers that expose two extra endpoints and know each other via per-registrar config. No manifest, no quorum, no gossip.

#### 16.1 Registrars as federated peers

A registrar's `/.well-known/webmail-federation` adds one field:

```
{ ...§5 fields..., "role": "registrar" }
```

All other trust mechanics — key pinning, TOFU, signature verification — reuse §5/§6 unchanged. A registrar is discoverable to anyone; it becomes relevant to another registrar's `/lookup` logic only if it appears in that registrar's `registrar_peers` config:

```
'registrar_peers' => [
    ['domain' => 'mailhub.org',     'pinned_key' => 'ed25519:...'],
    ['domain' => 'example-mail.net','pinned_key' => 'ed25519:...'],
],
```

Adding or removing a peer is an ops change on each running registrar. Trust is bilateral and local, matching how federated tenants already treat each other. A registrar not in your `registrar_peers` is not consulted on lookups and has no standing to drive shadow consolidation on your side.

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
    spawn: GET https://{peer.domain}/api/v1/registrar/lookup?hash=...
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
    GET /api/v1/registrar/pending-shadows?hash=... → peer_shadows
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

**Rogue registrar.** A peer in your `registrar_peers` fabricates a `/lookup` response. Sends for the targeted real_email may be routed to a canonical the rogue controls, for tenants that trust it. Defenses:

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

1. **Client**: introduce `/config.json` + `TenantContext`, strip literal `103mail.com` from source.
2. **Backend**: move default domain to tenant config; add `domain.backend_url` / `public_key` / `is_local`.
3. **Backend**: add well-known discovery route + keypair generation on first boot.
4. **Backend**: split `addMessageToSubscribers` into local/remote branches; add outbound `FederationClient` + `federation_delivery` table + retry worker.
5. **Backend**: add inbound `FederationApi` with signature verification.
6. **Spin up** a second tenant (e.g. `example.com`) in staging, verify round-trip: compose from 103mail → inbox on example.com → reply federates back.
7. **Harden**: rate-limit inbound per peer, cap attachment size, bounce semantics (`federation_delivery.status = permanent_failure` generates a local bounce message to sender).
8. **Schema (all tenants)**: `user.real_email` + `real_email_verified_at` + unique `(domain_id, real_email)`.
9. **Registration (all tenants)**: require real_email at signup; SMTP double-opt-in; gate account activation on verification.
10. **Notifications (all tenants)**: wire `sent_email` `To:` to `user.real_email`.
11. **Schema (registrars)**: `alias.real_email`, `alias.forward_canonical`, `alias.state`, `alias.issued_at`, `alias.expires_at`.
12. **Compose (plain tenants)**: add `registrars` config; extend §4 fan-out with the registrar branch — federate non-federated recipients to the configured registrar.
13. **Compose (registrars)**: implement §15.5 — local match, peer match, auto-onboarding with tentative shadow + SMTP invite + claim page (both signup and federated-claim paths).
14. **Federation (registrars)**: `POST /api/v1/federation/claim-by-real-email`; reuse §14.4 backfill for messages moving out of a shadow.
15. **Multi-registrar (registrars)**: expose `/api/v1/registrar/lookup` and `/api/v1/registrar/pending-shadows`; add `registrar_peers` config; cross-registrar query on lookup miss; consolidation on claim.
16. **Staging test**: two-registrar + one-plain-tenant deployment. Verify (a) plain tenant's send to unseen real_email resolves via registrar and auto-onboards; (b) race between two registrars converges on a single mailbox after claim via consolidation; (c) federated-account claim produces a stable forwarder, and subsequent sends route directly without creating new shadows.

---

## Open questions

- **Reply threading across domains**: `g_thread.id` is a local autoincrement. Need a federated stable ID — propose `g_thread.federation_id` (UUID, generated on origin, carried in envelope, matched on inbound replies).
- **Account portability**: if `user@example.com` wants to move to `acme.io`, do we keep the old address forwarding? Out of scope for v1 — document as "no".
- **Spam / abuse**: TOFU pubkey pinning trusts any domain that has DNS. Add a tenant-level knob (`federation.accept_from: any | allowlist`) for private deployments.
- **Real-email rotation latency**: §16.6 relies on re-issue-or-expire (TTL 90d). A user changing `real_email` on their plain-tenant home is reflected at registrars only after the next claim; old forwarders may route stale for up to the TTL. v2 could add a proactive `POST /federation/real-email-change` notification to shorten the window. Out of scope v1.
- **Record-level signatures on `/registrar/lookup`**: v1 trusts the HTTPS response from a `registrar_peers`-listed peer. v2 hardening would sign each record so cached entries survive peer removal with attributable provenance. Not required for correctness given the curated-peer model.
- **Registrar admission**: `registrar_peers` is curated per-operator; there is no global registrar directory. Adding a new registrar requires each existing registrar's operator to add its domain + key to their own config. This is intentionally a governance-free, config-level choice — the tradeoff is that a new registrar is not immediately visible network-wide. Acceptable while N stays small.
