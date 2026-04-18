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
- **Attachments**: inline base64 is fine for small; for large, consider a pull model (`GET /api/v1/federation/attachment/{fedId}` on origin, signed) to avoid buffering both sides.
- **Account portability**: if `user@example.com` wants to move to `acme.io`, do we keep the old address forwarding? Out of scope for v1 — document as "no".
- **Spam / abuse**: TOFU pubkey pinning trusts any domain that has DNS. Add a tenant-level knob (`federation.accept_from: any | allowlist`) for private deployments.
