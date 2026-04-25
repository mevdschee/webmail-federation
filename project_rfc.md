# RFC: Webmail-Federation Protocol

- **Name:** Webmail-Federation
- **Version:** 0.2 (draft)
- **Status:** Draft — not yet implemented
- **Date:** 2026-04-23
- **Editor:** Maurits van der Schee

## Abstract

This document specifies Webmail-Federation, an HTTPS-based protocol for
exchanging messages between independently operated Webmail Domains. A Webmail
Domain is the authoritative host for mailboxes under exactly one DNS domain; a
single physical Server may host one or more Domains, each appearing to its peers
as an independent participant. Webmail-Federation is a narrower alternative to
SMTP between cooperating Webmail Domains — it does not replace SMTP on the open
internet. The protocol covers peer discovery, envelope format, cryptographic
authentication, idempotent delivery, attachment transfer, and error semantics.
It does not cover user authentication, end-to-end encryption of content, or spam
filtering; those concerns are addressed elsewhere.

## Status of this memo

This is a working draft. It is intended to become the reference protocol
specification for the `webmail-federation` project. Until version 1.0 is
published, the wire format and endpoint paths are subject to incompatible
change.

## 1. Conventions and terminology

### 1.1 Requirement levels

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in RFC 2119 and RFC 8174 when, and
only when, they appear in all capitals.

The name "Webmail-Federation" is used consistently throughout. Implementations
are free to refer to it as they like; the wire protocol does not carry the name.

### 1.2 Deployment hierarchy

Webmail-Federation participants form a four-level hierarchy. Only the Domain and
User levels are observable on the wire; Server and Customer are implementation
details of a given deployment and are not negotiated, advertised, or transmitted
by this protocol.

```
Server      (physical deployment; process + database)
  └── Customer    (tenant / billing / admin boundary; internal only)
        └── Domain      (DNS domain; federation participant on the wire)
              └── User        (mailbox holder; identified by sender_email)
```

### 1.3 Term definitions

- **DNS domain** — The bare internet domain a Domain is authoritative for (e.g.
  `example.com`).
- **Federation hostname** — The fixed hostname `webmail.{domain}` under which
  every endpoint defined by this document is served. All protocol URLs are
  rooted at `https://webmail.{domain}`. No other subdomain is permitted (see
  §4.1 and §9.2).
- **Domain** — The logical Webmail-Federation participant. A Domain is
  authoritative for exactly one DNS domain and for the mailboxes local to it.
  All wire-protocol rules apply per-Domain.
- **Server** — A physical deployment of the Webmail backend (process +
  database). A Server hosts one or more Customers; each Customer owns one or
  more Domains.
- **Customer** — A Server-internal tenant entity that owns one or more Domains
  and the Users within them. The Customer concept is not transmitted on the
  wire; peers **MUST NOT** be given any way to infer whether two Domains share a
  Customer.
- **User** — An individual account within a Customer. A User belongs to exactly
  one Customer and may hold addresses (aliases) in any Domain owned by that
  Customer. On the wire, a User is identified only by the `sender_email` on
  outbound envelopes (§5.2).
- **Local recipient** — An account whose address ends in the receiving Domain's
  own DNS domain.
- **Remote recipient** — An account whose address ends in any other DNS domain.
- **Origin Domain** — The Domain whose user composes a new message.
- **Destination Domain** — The Domain authoritative for a Remote recipient.
- **Envelope** — The JSON document transferred between Domains for one message
  (§5).
- **Peer descriptor** — The JSON document published at the Federation hostname
  describing a Domain to its peers (§4.2).
- **Federation-enabled DNS domain** (or "federated domain") — A DNS domain for
  which the observing Domain holds a cached, successfully validated peer
  descriptor (§4.3). Federation status is observer-relative: `example.com` may
  be federated from the point of view of `103mail.com` but not from `acme.io`'s
  if `acme.io` has never successfully discovered it.
- **Registrar** — A Domain whose peer descriptor advertises `"registrar": true`
  (§4.2). Registrars accept envelopes addressed to External addresses and expose
  the endpoints in §8.3–§8.5. Non-Registrar is the default.
- **External address** — A mail address whose domain is neither the
  Destination's own DNS domain nor a Federation-enabled DNS domain from the
  Destination's point of view (e.g. `alice@gmail.com` to any Destination that
  does not federate with `gmail.com`).

## 2. Goals and non-goals

### 2.1 Goals

- Permit a user at domain `A` to send a message to a user at domain `B` when `A`
  and `B` are served by separate Webmail Domains.
- Preserve the no-SMTP property: no message body is delivered to a host that is
  not a Webmail Domain.
- Authenticate every inter-Domain request so that a Domain cannot spoof messages
  purporting to originate at a third domain.
- Be idempotent under retry.
- Permit the Origin to accept and durably persist a user's compose action even
  when the Destination is unreachable, and to retry delivery asynchronously.
- Permit Domains that advertise the Registrar role (§8) to resolve External
  addresses to Federation-enabled canonicals and route envelopes accordingly,
  without requiring a global registry.

### 2.2 Non-goals

- Transparent relay through untrusted intermediaries: envelopes flow Origin →
  Destination directly, or Origin → Registrar → Destination for
  External-addressed envelopes. No opaque relay chains.
- Backwards compatibility with SMTP, IMAP, JMAP, or any existing internet mail
  protocol.
- End-to-end encryption or signing of message content for end-user verification.
  Signatures in this protocol authenticate Domain-to-Domain transport only.
- Cross-Domain address portability (a User at `alice@example.com` cannot retain
  that address when moving to `acme.io`). Note: Registrar "forwarder" aliases
  (§8.5) route External addresses to federated canonicals and preserve
  sender-visible labels; this is a routing indirection inside the Registrar, not
  address portability across Domains.

## 3. Protocol overview

A Webmail-Federation exchange for a single message proceeds as follows:

```
Origin Domain                                    Destination Domain
      |                                                      |
      |  1. GET /.well-known/webmail-federation              |
      |     (one-time; cached up to 24h; §4.3)               |
      |----------------------------------------------------->|
      |<-----------------------------------------------------|
      |         peer descriptor                              |
      |                                                      |
      |  2. POST /api/v1/federation/inbound (signed env; §6) |
      |----------------------------------------------------->|
      |<-----------------------------------------------------|
      |                 202 Accepted                         |
```

Step 1 is cacheable and idempotent. Step 2 is the normative transfer event and
carries the message body and all attachment bytes in a single request; there is
no separate fetch step.

### 3.1 Reliability model

Every inter-Domain request runs over HTTPS against a remote peer whose
availability the Origin does not control. Implementations **MUST** design for
unreliable transport.

- **User-facing actions MUST NOT block on remote peers.** When a User composes a
  message, accepts a claim, or performs any action whose scope includes a remote
  Domain, the Origin **MUST** durably persist the user's intent locally before
  returning success to the user. The remote HTTP call is performed by a
  background worker after the user has been acknowledged.
- **Mutating requests are retried with idempotency.** Every mutating outbound
  request — inbound envelope (§6), `claim-by-real-email` (§8.5) — carries a
  UUIDv7 identifier (`envelope_id`, `claim_id`) chosen before the first attempt
  and reused on every retry. The Destination **MUST** deduplicate on
  `(X-Federation-Sender, identifier)` and return the original terminal response
  on replay without re-applying side effects. See §6.4 for the retry schedule.
- **Read requests MAY be synchronous but MUST be bounded.** Peer discovery
  (§4.1), registrar lookup (§8.3), and pending-shadows queries (§8.4) are GETs
  (or POSTs with no state mutation, in the case of §8.3 — see that section).
  Callers **MAY** block on the result. Every such request **MUST** carry a
  finite connect and read timeout. On timeout or transport failure,
  implementations **MUST** have a fallback behavior specified by the calling
  flow (e.g. re-queue, use cached value, proceed with fallback target).
- **No synchronous fan-out.** If a single user action would contact multiple
  remote Domains, the Origin **MUST** dispatch each Destination independently.
  One peer's slowness or failure **MUST NOT** block delivery to the others.
- **Permanent failure has a user-visible consequence.** When the retry budget is
  exhausted or a terminal `4xx` (other than `408`, `425`, `429`) is received,
  the Origin **MUST** produce a local bounce per §6.5. An exhausted retry that
  silently disappears is not conformant.

## 4. Peer discovery

### 4.1 Well-known endpoint

Each Domain **MUST** publish its peer descriptor at:

```
https://webmail.{domain}/.well-known/webmail-federation
```

Publication at the apex (`https://{domain}/...`) and at any subdomain other than
`webmail.` is not defined by this protocol. Peers **MUST NOT** probe, accept, or
cache peer descriptors served from any hostname other than `webmail.{domain}`.
This is consistent with RFC 8615 §3, which delegates the hostname-scope choice
to the application specification.

The endpoint:

- **MUST** be served exclusively over HTTPS (see §9.2 for TLS requirements).
- **MUST** respond with `Content-Type: application/json; charset=utf-8`.
- **MUST NOT** require client authentication (the descriptor itself is public).
- **MUST NOT** redirect. A peer that receives a `3xx` response from the
  well-known URL **MUST** abort the fetch and treat discovery as failed.
- **MUST NOT** return a response body larger than 65536 bytes. A peer **MUST**
  cap its read at that size and abort if exceeded.

A Server hosting multiple Domains **MUST** serve a distinct peer descriptor at
each Domain's Federation hostname; descriptors **MUST NOT** be shared across DNS
domains. Each co-hosted Domain **MUST** carry its own `public_key`.

### 4.1.1 Subdomain Domains

A Domain's DNS domain **MAY** be a subdomain (e.g. `mail.example.com`,
`eu.acme.io`). The `webmail.` label is always prepended to the Domain's full DNS
domain — it is not hoisted to the registrable parent. The Federation hostname is
therefore exactly one DNS label deeper than the Domain's DNS domain, regardless
of depth:

| Domain DNS domain        | Federation hostname (required)   | Not acceptable                            |
| ------------------------ | -------------------------------- | ----------------------------------------- |
| `example.com`            | `webmail.example.com`            | —                                         |
| `mail.example.com`       | `webmail.mail.example.com`       | `webmail.example.com`                     |
| `eu.tenant.corp.example` | `webmail.eu.tenant.corp.example` | `webmail.corp.example`, `webmail.example` |

Consequences:

- The peer descriptor for a subdomain Domain **MUST** be served at
  `https://webmail.{subdomain}/.well-known/webmail-federation` where
  `{subdomain}` is the Domain's full DNS domain. Peers **MUST NOT** probe,
  accept, or cache a descriptor fetched from the registrable parent's Federation
  hostname as authoritative for a subdomain.
- The TLS certificate presented at the Federation hostname **MUST** be valid for
  that exact hostname (§9.2). A certificate valid only for `webmail.{parent}` is
  insufficient; a wildcard covering `*.{parent}` does not cover
  `webmail.sub.{parent}` (RFC 6125 §6.4.3 wildcards match a single label).
- A parent Domain and a subdomain Domain are independent Webmail-Federation
  participants. `webmail.example.com` asserts nothing about
  `webmail.mail.example.com`, and vice versa; their descriptors, public keys,
  Registrar status, and address spaces **MUST NOT** be conflated. A peer that
  federates with `example.com` does not thereby federate with
  `mail.example.com`; a separate discovery (§4.3) is required.
- `envelope.origin_domain` and the domain part of `message.sender_email`
  **MUST** be byte-equal to the Domain's DNS domain. Parent/child DNS
  relationships confer no substitution (this reiterates §8.5 step 2 for the
  general case).

### 4.2 Peer descriptor

The descriptor is a JSON object (RFC 8259) with the following members:

| Field                  | Type    | Required | Description                                                                                                                                                                 |
| ---------------------- | ------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `domain`               | string  | yes      | The DNS domain the Domain is authoritative for. **MUST** match the host suffix `webmail.{domain}` from which the descriptor was fetched.                                    |
| `public_key`           | string  | yes      | Current Ed25519 signing public key, encoded as `ed25519:{b64}` where `{b64}` is RFC 4648 §4 base64 (with padding) of the 32 raw public-key bytes.                           |
| `previous_public_keys` | array   | no       | Ed25519 public keys in the same encoding, accepted by this Domain's peers during a rotation window (§9.5). Absent or `[]` if no rotation is in progress. Maximum 4 entries. |
| `versions`             | array   | yes      | Non-empty array of protocol version strings this Domain understands, e.g. `["0.2"]`.                                                                                        |
| `registrar`            | boolean | no       | `true` if this Domain is a Registrar (§8). Defaults to `false`.                                                                                                             |
| `max_envelope_bytes`   | integer | no       | Advertised inbound envelope-body size cap in bytes. Default when absent: `10485760` (10 MiB).                                                                               |
| `max_attachment_bytes` | integer | no       | Advertised per-attachment decoded size cap in bytes. Default when absent: `26214400` (25 MiB).                                                                              |

Endpoint paths are fixed by this specification — they are not advertised in the
descriptor. For version `0.2`, every Webmail-Federation Domain serves its
endpoints at:

| Endpoint              | Fixed URL                                                                          |
| --------------------- | ---------------------------------------------------------------------------------- |
| Peer descriptor       | `https://webmail.{domain}/.well-known/webmail-federation`                          |
| Inbound envelope      | `https://webmail.{domain}/api/v1/federation/inbound`                               |
| Claim by real email   | `https://webmail.{domain}/api/v1/federation/claim-by-real-email` (Registrars only) |
| Registrar lookup      | `https://webmail.{domain}/api/v1/registrar/lookup` (Registrars only)               |
| Pending-shadows query | `https://webmail.{domain}/api/v1/registrar/pending-shadows` (Registrars only)      |

A peer **MUST** derive these URLs from the fetched descriptor's `domain` field;
a peer **MUST NOT** accept a descriptor that attempts to override any endpoint
path (descriptor fields such as `inbound_path`, `backend`, or `inbound` are not
defined in this version; if present, they **MUST** be ignored per the
unknown-field rule below).

Unknown fields in the descriptor **MUST** be ignored by the fetching peer.

Example:

```json
{
  "domain": "example.com",
  "public_key": "ed25519:MTIzNDU2Nzg5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTI=",
  "previous_public_keys": [],
  "versions": ["0.2"],
  "registrar": false,
  "max_envelope_bytes": 10485760,
  "max_attachment_bytes": 26214400
}
```

A fetching peer **MUST** reject a descriptor that:

- fails JSON parsing,
- is missing any required field,
- carries a `domain` not matching the Federation hostname from which it was
  fetched,
- carries a `public_key` that does not decode to exactly 32 bytes,
- carries `previous_public_keys` with more than 4 entries or any entry that does
  not decode to exactly 32 bytes,
- carries an empty `versions` array.

### 4.3 Trust-on-first-use and caching

A Domain **MUST** cache the peer descriptor keyed by DNS domain. On first
successful fetch, it pins the `public_key` (trust on first use) and uses it to
verify subsequent signatures from that DNS domain.

Refresh policy:

- A Domain **MUST NOT** refresh a cached descriptor more often than once every
  24 hours, except as required below.
- A Domain **MUST** refresh if an inbound signature fails verification against
  the currently cached keyset (current `public_key` plus any non-expired
  `previous_public_keys` entries).
- A Domain **MAY** refresh when it is notified by operational means (log, admin
  action) that a peer has rotated its key.
- A Domain that has failed discovery (404, TLS failure, JSON error) **MUST NOT**
  re-probe that peer more often than once per hour per failure (negative-cache
  floor).

A Domain **MAY** expose an operator-controlled allowlist of peer domains;
envelopes from peers not on the allowlist **MUST** be rejected with
`403 Forbidden` (§6.3, §9.7).

## 5. Envelope format

### 5.1 Content type and encoding

Envelopes are JSON objects per RFC 8259, transferred with:

```
Content-Type: application/json; charset=utf-8
```

Requests **MUST NOT** carry `Content-Encoding: gzip` or any other compression on
the request body, because the signature (§6.2) is computed over the transmitted
body bytes and compression middleware would invalidate it. Responses **MAY** be
compressed.

`Transfer-Encoding: chunked` is permitted; the signed bytes are the reassembled
body, not the wire chunks.

### 5.2 Envelope schema

```json
{
  "version": "0.2",
  "envelope_id": "018f4d2b-9f21-7890-abcd-ef0123456789",
  "origin_domain": "103mail.com",
  "origin_sent_at": "2026-04-23T10:15:00.000Z",
  "thread": {
    "federation_id": "018f4d2b-9f21-7890-abcd-ef0123456790",
    "subject": "Hello from 103mail",
    "in_reply_to": null
  },
  "message": {
    "federation_id": "018f4d2b-9f21-7890-abcd-ef0123456791",
    "sender_email": "1234567@103mail.com",
    "sender_name": "Alice",
    "body": "plain text body",
    "body_html": "<p>optional html body</p>",
    "sent_at": "2026-04-23T10:15:00.000Z"
  },
  "attachments": [
    {
      "federation_id": "018f4d2b-9f21-7890-abcd-ef0123456792",
      "file_name": "report.pdf",
      "mime_type": "application/pdf",
      "file_size": 138241,
      "hash_sha256_hex": "3a7bd3e2360a3d29eea436fcfb7e44c735d117c42d1c1835420b6b9942dd4f1b",
      "data": "JVBERi0xLjQK..."
    }
  ],
  "recipients": [
    "bob@example.com",
    "carol@example.com"
  ]
}
```

### 5.3 Field rules

Lengths below are maximum decoded byte lengths of the UTF-8 string. Receivers
**MUST** reject envelopes that violate these bounds with `400 Bad Request` and
`error = "schema_violation"`.

| Field                   | Type   | Required | Constraint                                                                                                                                                                                                                                                          |
| ----------------------- | ------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `version`               | string | yes      | **MUST** appear in the Destination's advertised `versions`. Otherwise `400 unsupported_version`.                                                                                                                                                                    |
| `envelope_id`           | string | yes      | UUIDv7 per RFC 9562, canonical hyphenated lowercase form.                                                                                                                                                                                                           |
| `origin_domain`         | string | yes      | Lowercased DNS name. **MUST** equal `X-Federation-Sender` (§6.1) and **MUST** equal the domain part of every `message.sender_email`.                                                                                                                                |
| `origin_sent_at`        | string | yes      | RFC 3339 UTC timestamp with trailing `Z`. Millisecond precision permitted; higher precision **MUST NOT** be used.                                                                                                                                                   |
| `thread.federation_id`  | string | yes      | UUIDv7.                                                                                                                                                                                                                                                             |
| `thread.subject`        | string | yes      | May be empty string. Max 998 bytes.                                                                                                                                                                                                                                 |
| `thread.in_reply_to`    | string | no       | If non-null, a UUIDv7 that is the `federation_id` of a thread the Origin has previously sent to or received from the Destination. The Destination **MAY** accept an envelope whose `in_reply_to` it has not seen and **SHOULD** create a fresh thread in that case. |
| `message.federation_id` | string | yes      | UUIDv7.                                                                                                                                                                                                                                                             |
| `message.sender_email`  | string | yes      | RFC 5321 / 5322 addr-spec. Lowercased domain part. Max 320 bytes.                                                                                                                                                                                                   |
| `message.sender_name`   | string | no       | If present, max 255 bytes.                                                                                                                                                                                                                                          |
| `message.body`          | string | yes      | Plain-text body. May be empty. Max 1048576 bytes (1 MiB).                                                                                                                                                                                                           |
| `message.body_html`     | string | no       | HTML body. Max 4194304 bytes (4 MiB).                                                                                                                                                                                                                               |
| `message.sent_at`       | string | yes      | Same rules as `origin_sent_at`.                                                                                                                                                                                                                                     |
| `attachments`           | array  | yes      | 0–64 entries. See §7 for attachment field rules.                                                                                                                                                                                                                    |
| `recipients`            | array  | yes      | 1–100 entries. Each entry is an addr-spec. **MUST** either all share the Destination's DNS domain, or the Destination **MUST** be a Registrar (§8.2).                                                                                                               |

Unknown fields in the envelope **MUST** be ignored by the Destination. No
unknown field may carry side effects.

### 5.4 Idempotency and retention

A Destination **MUST** deduplicate envelopes by
`(X-Federation-Sender, envelope_id)`. Re-delivery of the same key **MUST**
produce the same terminal response (typically `202 Accepted`) without
re-inserting messages into mailboxes.

Destinations **MUST** retain the `(sender, envelope_id)` dedup record for at
least 96 hours from first acceptance. This exceeds the §6.4 retry budget by 24
hours to cover clock skew and re-queue edge cases. Origins **MUST NOT** issue
retries after the 72-hour budget has elapsed.

## 6. Inbound transfer

### 6.1 Request

The Origin Domain issues a `POST` to the fixed inbound URL derived from the
Destination's `domain`:

```
POST https://webmail.{destination_domain}/api/v1/federation/inbound
Host: webmail.{destination_domain}
Content-Type: application/json; charset=utf-8
Content-Length: {decimal length of body}
Date: {RFC 7231 IMF-fixdate, UTC}
X-Federation-Version:   0.2
X-Federation-Sender:    {origin_domain}
X-Federation-Nonce:     {unique per sender per 10-minute window}
X-Federation-Signature: ed25519:{base64(signature)}

{envelope JSON}
```

The path `/api/v1/federation/inbound` is fixed by this specification (§4.2) and
**MUST NOT** be configured by either peer. An Origin that receives a `3xx`
redirect on this URL **MUST** abort; redirects are not honored.

All listed headers are **REQUIRED**. The `Date` header **MUST** be present (the
§6.2 clock-skew check depends on it).

`X-Federation-Nonce` **MUST** be a 128-bit value encoded as RFC 4648 §5
base64url without padding (22 chars). A ULID or a UUIDv7 is acceptable as long
as the encoded form is base64url-unpadded. The value must be unique per sender
within a 10-minute window.

HTTP version: `HTTP/1.1` or `HTTP/2` **MUST** be supported by every conformant
Destination. `HTTP/3` **MAY** be supported. `Transfer-Encoding: chunked` is
permitted; `Content-Encoding` on the request body is not (§5.1).

### 6.2 Signature

Ed25519 (RFC 8032) is the only signature scheme in this version. The signature
is computed over the following bytes, concatenated in order with no additional
separators beyond the explicit `0x0A` bytes shown:

```
signing_input =
    <X-Federation-Version, UTF-8 bytes>        0x0A
    <X-Federation-Sender,  UTF-8 bytes>        0x0A
    <X-Federation-Nonce,   UTF-8 bytes>        0x0A
    <Date,                 UTF-8 bytes>        0x0A
    <request body bytes as transmitted>

signature = Ed25519.sign(private_key_of(origin_domain), signing_input)
```

- No pre-hashing. Ed25519 handles hashing internally.
- The request body bytes are the exact bytes the Origin transmitted (after
  reassembling any HTTP chunking, before any hypothetical decompression — none
  is allowed on the request).
- `ed25519:{base64}` is RFC 4648 §4 standard base64 with padding. The encoded
  signature is 64 raw bytes → 88 base64 chars (including trailing `=`).

Verification on the Destination:

```
signing_input = reconstruct from the same four headers + received body
Ed25519.verify(resolved_public_key, signature, signing_input)
```

Where `resolved_public_key` is the current `public_key` of the sender's cached
descriptor (§4.3), or any entry in `previous_public_keys` — the verifier
**MUST** accept any match.

The Destination **MUST** reject with `401 Unauthorized` if any of the following
is true:

- `X-Federation-Signature` does not verify against any of the sender's currently
  valid keys (→ `bad_signature`).
- `X-Federation-Sender` does not match `envelope.origin_domain` (→
  `sender_mismatch`).
- `X-Federation-Sender` does not match the domain part of every
  `message.sender_email` (→ `sender_mismatch`).
- `X-Federation-Nonce` has been seen within the last 10 minutes from this sender
  (→ `replay`).
- `Date` is absent, unparseable, or more than 5 minutes from the Destination's
  clock (→ `stale_request`).

### 6.3 Responses

| Status | Condition                                                                                                   | Body                                                                             |
| ------ | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| 202    | Envelope accepted (new or idempotent replay).                                                               | `{"envelope_id": "...", "status": "accepted"}`                                   |
| 400    | Malformed envelope, unsupported version, schema violation, attachment mismatch.                             | `{"error": "<code>", "detail": "..."}`                                           |
| 401    | Signature, nonce, sender, or date check failed.                                                             | `{"error": "bad_signature" \| "sender_mismatch" \| "replay" \| "stale_request"}` |
| 403    | Origin domain is not on the Destination's allowlist.                                                        | `{"error": "peer_not_allowed"}`                                                  |
| 404    | All `recipients` are unknown at the Destination and none were onboarded.                                    | `{"error": "no_recipients"}`                                                     |
| 413    | Envelope body exceeds `max_envelope_bytes`.                                                                 | `{"error": "too_large"}`                                                         |
| 415    | `Content-Type` is not `application/json; charset=utf-8`, or `Content-Encoding` was set on the request body. | `{"error": "unsupported_media_type"}`                                            |
| 429    | Rate limit exceeded.                                                                                        | `{"error": "rate_limited", "retry_after_seconds": n}`                            |
| 5xx    | Transient Destination failure.                                                                              | `{"error": "server_error"}`                                                      |

`400`, `401`, `403`, `404`, `413`, and `415` are **permanent**; the Origin
**MUST NOT** retry the same envelope. `408`, `425`, `429`, and `5xx` are
**transient** and **SHOULD** be retried per §6.4.

Every error response **SHOULD** carry a `Retry-After` HTTP header on `429`;
`retry_after_seconds` in the JSON body is a convenience mirror.

### 6.4 Retry policy

Retry schedule for transient failures:

- **Base delay:** 30 seconds.
- **Backoff:** Exponential with base 2 and full jitter per attempt. Cap delay at
  1 hour.
- **Budget:** exactly 72 hours from the first attempt. **MUST NOT** retry after
  the budget has elapsed.
- **Rate-limit pushback:** On `429` with `Retry-After` or `retry_after_seconds`,
  the Origin **MUST** delay at least that long for the next attempt to that
  Destination and **SHOULD** pause other outbound sends to the same Destination.

Network-level errors (connection refused, TLS failure, connect timeout, read
timeout) are treated as transient.

### 6.5 Destination processing and bounce semantics

On accepting an envelope the Destination **MUST**:

1. Record `(X-Federation-Sender, envelope_id)` in the dedup table (retention per
   §5.4).
2. For each recipient in `recipients`, resolve or create the target User account
   within the Destination's tenancy. Onboarding policy for
   unknown-but-acceptable recipients is implementation-defined and out of scope
   for this document; Destinations that do not onboard **MUST** respond
   `404 no_recipients`.
3. Persist per-recipient delivery state (the specific table layout is out of
   scope; the federation identifiers **MUST** be retrievable so that reply
   correlation by `thread.federation_id` is possible).
4. Emit the Destination's notification to each recipient. The notification body
   is out of scope.

On permanent failure (retry budget exhausted, or a permanent 4xx received), the
Origin **MUST** generate a local bounce back to the sender:

- The bounce is a local `message` row inserted into the sender's inbox, under
  the same `customer` tenancy as the original sender.
- Subject: `"Delivery failed: {original subject}"`.
- Body: a plain-text summary identifying (a) the failed recipient addresses and
  their Destination Domain, (b) the terminal HTTP status and `error` code
  received (or `"timed_out"` if the budget was exhausted), (c) the `envelope_id`
  for correlation.
- `sender_email`: a reserved local-part `mailer-daemon@{origin_domain}`.
- The bounce is local — it is **NOT** a federated envelope and **MUST NOT**
  leave the Origin Server.

## 7. Attachment transfer

All attachment bytes **MUST** be carried inline in the envelope. The `data`
field is the attachment bytes encoded as RFC 4648 §4 standard base64 (with
padding). After a Destination accepts an envelope it holds a self-contained
copy; no subsequent request to the Origin is ever required to read an
attachment. This preserves readability if the Origin is decommissioned.

Per-attachment field rules:

| Field             | Type    | Required | Constraint                                                                |
| ----------------- | ------- | -------- | ------------------------------------------------------------------------- |
| `federation_id`   | string  | yes      | UUIDv7.                                                                   |
| `file_name`       | string  | yes      | UTF-8, max 255 bytes. Receivers **MUST** sanitize for storage/display.    |
| `mime_type`       | string  | yes      | RFC 6838 media type. Max 255 bytes.                                       |
| `file_size`       | integer | yes      | Decoded byte length of `data`. **MUST** match the decoded length exactly. |
| `hash_sha256_hex` | string  | yes      | SHA-256 of the decoded attachment bytes, lowercase hex, exactly 64 chars. |
| `data`            | string  | yes      | Base64-encoded attachment bytes.                                          |

The Destination **MUST** decode `data`, recompute `file_size` and
`hash_sha256_hex`, and reject the envelope with `400 attachment_mismatch` if
either differs.

Size limits:

- An Origin **MUST NOT** send an envelope exceeding the Destination's advertised
  `max_envelope_bytes` (counted as the body size in bytes after JSON
  serialization).
- An Origin **MUST NOT** send an attachment whose decoded size exceeds the
  Destination's advertised `max_attachment_bytes`.
- A Destination receiving either kind of violation **MUST** respond
  `413 too_large`.
- An Origin that needs to send an attachment larger than any Destination will
  accept **MUST** generate a permanent local bounce to the sender per §6.5. This
  protocol does not define an out-of-band transfer.

This design trades bandwidth and envelope size against simplicity, durability,
and the "all data local to every Domain" invariant. The tradeoff is deliberate
and is normative.

## 8. Registrar extension

This section defines additional wire protocol that applies only to Domains
advertising `"registrar": true` in their peer descriptor (§4.2). Non-Registrar
Domains are unaffected by §8.

### 8.1 Registrar advertisement

A Domain **MAY** advertise `"registrar": true`. A Domain whose descriptor omits
`registrar`, or whose `registrar` is `false`, is a non-Registrar.
Non-Registrars:

- **MUST NOT** accept envelopes containing External addresses in `recipients`
  (§8.2).
- **MUST NOT** serve `/api/v1/registrar/lookup` or
  `/api/v1/registrar/pending-shadows` (or MAY return `404 Not Found` at those
  paths).
- **MUST NOT** accept `POST /api/v1/federation/claim-by-real-email` (§8.5).

A change in `registrar` status takes effect from a peer's next descriptor
refresh. Because refresh is rate-limited to once per 24 hours (§4.3), a flip has
up to 24h of lag at any given peer.

### 8.2 Recipient rule relaxation

When an envelope is delivered to a Domain whose current peer descriptor
advertises `"registrar": true`:

- `recipients` **MAY** contain External addresses.
- The Destination **MUST NOT** reject the envelope on grounds of `recipients`
  domain mismatch.
- The Destination is responsible for resolving or onboarding each External
  recipient. Internal resolution semantics are out of scope.

An Origin **MUST NOT** include External addresses in `recipients` when sending
to a non-Registrar Destination. Such an envelope, if received, **MUST** be
rejected with `400 not_registrar`.

An envelope mixing Destination-domain recipients and External recipients is
valid only when the Destination is a Registrar. An Origin **MAY** split by
recipient class and send separate envelopes.

**Stable selection of a Registrar for a given External address.** For any given
External recipient in an outbound send, an Origin **MUST** select exactly one
Registrar Domain as the Destination for that recipient and **MUST** keep that
selection stable across retries of the same envelope, across Origin restarts,
and across queue replays. Specifically:

- The chosen Registrar's DNS domain **MUST** be persisted in the Origin's
  outbound queue alongside the `envelope_id`, and **MUST** be re-read from the
  queue on each retry attempt. Fresh re-selection on retry is forbidden.
- If the chosen Registrar's descriptor no longer advertises `registrar: true` at
  retry time, the Origin **MUST** treat the send as permanently failed
  (`registrar_revoked`) and generate a bounce per §6.5.

How an Origin initially chooses the Registrar (static configuration, ordered
list of fallbacks) is out of scope; the stability and single-destination
requirements are not.

An Origin that is itself a Registrar **MAY** resolve External recipients locally
rather than selecting a peer Registrar. A non-Registrar Origin with no Registrar
available to it **MUST** refuse the send at compose time and **MUST NOT**
silently drop or rewrite recipients.

### 8.3 Registrar lookup

A Registrar **MUST** expose, at its Federation hostname:

```
POST https://webmail.{domain}/api/v1/registrar/lookup
Content-Type: application/json; charset=utf-8

{
  "version": "0.2",
  "hash":    "{sha256_hex of lowercased external address}"
}
```

This endpoint is `POST` rather than `GET` to keep the hash out of HTTP access
logs and proxy caches (§9.8). It is idempotent despite being `POST` and **MAY**
be retried freely. The request carries no authentication.

| Status | Condition                    | Body                                                                                                |
| ------ | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| 200    | Authoritative record exists. | `{"canonical": "...", "forward_canonical": "..." \| null, "issued_at": "...", "expires_at": "..."}` |
| 404    | No authoritative record.     | `{"error": "not_found"}`                                                                            |
| 415    | Wrong content type.          | `{"error": "unsupported_media_type"}`                                                               |
| 429    | Rate limit exceeded.         | `{"error": "rate_limited", "retry_after_seconds": n}`                                               |

Rules:

- A Registrar **MUST** answer only from its own authoritative state. It **MUST
  NOT** recursively proxy through peer Registrars and serve the result as its
  own.
- A Registrar **MUST NOT** return records whose `expires_at` is in the past.
- At most one `{canonical, forward_canonical}` pair may be authoritative per
  `hash`; collisions are impossible by construction because
  `(domain_id, real_email)` is unique (the hash is per-address).
- A caller **MAY** cache `200` responses keyed by `(hash, responder_domain)`
  until `expires_at`. A caller **MUST NOT** serve cached peer responses via its
  own `/lookup` to third parties.
- Routing: if `forward_canonical` is non-null, deliver to the Domain
  authoritative for the domain of `forward_canonical` using `forward_canonical`
  as the recipient. Otherwise deliver to the Domain authoritative for the domain
  of `canonical` using `canonical` as the recipient.

### 8.4 Pending-shadow query

A Registrar **MUST** expose, at its Federation hostname:

```
POST https://webmail.{domain}/api/v1/registrar/pending-shadows
Content-Type: application/json; charset=utf-8
Date: {IMF-fixdate}
X-Federation-Version:   0.2
X-Federation-Sender:    {peer_registrar_domain}
X-Federation-Nonce:     {unique per sender per 10-minute window}
X-Federation-Signature: ed25519:{base64(signature)}

{
  "version": "0.2",
  "hash":    "{sha256_hex of lowercased external address}"
}
```

Authentication is identical to §6.2 with the request body in place of the
envelope. The Destination **MUST** reject with `401` if:

- The signature does not verify (→ `bad_signature`),
- `X-Federation-Sender`'s current descriptor does not advertise
  `registrar: true` (→ `not_registrar`),
- The nonce or `Date` checks fail (→ `replay` or `stale_request`).

| Status | Condition                                       | Body                                                                                                                                                                                                     |
| ------ | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 200    | One or more pending records exist for the hash. | `[ { "record_id": "...", "message_count": n, "oldest_sent_at": "...", "newest_sent_at": "...", "summary": [ { "federation_id": "...", "subject": "...", "sender_email": "...", "sent_at": "..." } ] } ]` |
| 404    | No pending records.                             | `{"error": "not_found"}`                                                                                                                                                                                 |
| 401    | Auth check failed.                              | `{"error": "<code>"}`                                                                                                                                                                                    |
| 429    | Rate limit exceeded.                            | `{"error": "rate_limited", "retry_after_seconds": n}`                                                                                                                                                    |

A "pending record" is a Registrar-internal state holding messages for an
External address not yet bound to a canonical. Internal life cycle is out of
scope.

### 8.5 Claim by real email

A Registrar **MUST** accept, at its Federation hostname:

```
POST https://webmail.{domain}/api/v1/federation/claim-by-real-email
Content-Type: application/json; charset=utf-8
Date: {IMF-fixdate}
X-Federation-Version:   0.2
X-Federation-Sender:    {origin_domain}
X-Federation-Nonce:     {unique per sender per 10-minute window}
X-Federation-Signature: ed25519:{base64(signature)}

{
  "version":    "0.2",
  "claim_id":   "{uuidv7}",
  "token":      "{opaque bearer token previously issued by the Registrar}",
  "real_email": "{external address}",
  "canonical":  "{federated address of the claiming user}"
}
```

Signing rules: identical to §6.2 with the request body in place of the envelope.

**Token threat model.** The `token` is a bearer credential the Registrar issued
to the External address via out-of-band email (typically an "onboarding
invite"). Anyone with access to the External address's inbox at issue time
through the validity window can consume the token. Protection against hijack
relies on (a) the External inbox being under the rightful recipient's control
and (b) the claiming Origin's Ed25519 signature proving that Origin vouches for
the `canonical`. Compromise of the External inbox at a specific moment can
permanently bind the real_email to an attacker's canonical; the Registrar
**MUST** bound the window by issuing tokens with an explicit expiry
(recommended: 30 days) and by making tokens single-use.

The Registrar **MUST**:

1. Verify the signature (§6.2). On failure: `401 bad_signature`.
2. Verify that the DNS domain of `canonical` equals `X-Federation-Sender`. On
   mismatch: `401 bad_sender`. Subdomain relationships (e.g. `mail.example.com`
   vs `example.com`) are not considered matches; the two strings must be
   byte-identical.
3. Resolve `token` against its internal issue log. On any of unknown / expired /
   already-consumed / bound-to-different-real-email: `400 invalid_token`.
4. Deduplicate on `(X-Federation-Sender, claim_id)`. A duplicate **MUST** return
   the original outcome (the original `2xx` or the original `4xx`) without
   further side effects.

On success, the Registrar:

- Binds `real_email` → `canonical` with `issued_at = NOW`,
  `expires_at = NOW + 90 days`. The binding is served via §8.3.
- **MUST** re-dispatch any pending-shadow messages for `real_email` to the
  Domain authoritative for `canonical`, via ordinary §6 inbound envelopes, with
  `recipients` rewritten to `[canonical]` and with `origin_domain`,
  `message.sender_email`, `message.federation_id`, and `thread.federation_id`
  preserved from the original envelopes. Because `envelope_id` is preserved, the
  receiving Destination deduplicates naturally on retry.
- **MUST** sign the re-dispatched envelopes with the original Origin's key
  **only if the Registrar was itself the Origin** of those messages; otherwise
  the Registrar **MUST** drop the re-dispatch and queue the messages for the
  canonical to pull once the canonical's Domain federates with the Registrar.
  The Registrar **MUST NOT** forge signatures on behalf of other Origins. (An
  alternative architectural choice — Registrar-as-relay with its own signature —
  is explicitly not permitted; re-dispatch authenticity is upstream.)

| Status | Condition                                        | Body                                                                        |
| ------ | ------------------------------------------------ | --------------------------------------------------------------------------- |
| 202    | Claim accepted (new or idempotent replay).       | `{"claim_id": "...", "status": "accepted"}`                                 |
| 400    | Invalid token, malformed body, version mismatch. | `{"error": "<code>", "detail": "..."}`                                      |
| 401    | Auth check failed.                               | `{"error": "bad_signature" \| "bad_sender" \| "replay" \| "stale_request"}` |
| 429    | Rate limit exceeded.                             | `{"error": "rate_limited", "retry_after_seconds": n}`                       |

### 8.6 Trust model for Registrars

This protocol does not define a global Registrar directory. Each operator
curates a local set of peer Registrars whose `/lookup` responses it will consult
and whose `/pending-shadows` queries it will answer. Peer selection is operator
policy and is out of scope.

A Registrar that is not on a Domain's peer list is, for the purposes of
§8.3–§8.5, identical to a non-Registrar: its lookup responses **MUST NOT** be
trusted and its pending-shadow queries **MUST** be rejected on signature grounds
(its key will not be pinned in §4.3).

Removal of a peer from the configured set is effective immediately. A Domain
that has cached lookup responses originating from a removed peer **MUST** purge
those cache entries within 60 seconds of the configuration change. This is a
hard requirement, not a best-effort hint.

### 8.7 Interaction with §5.4

`(origin_domain, envelope_id)` dedup (§5.4) applies unchanged to envelopes
carrying External addresses and to Registrar re-dispatches (§8.5). A re-dispatch
preserving `envelope_id` is collapsed at the final Destination naturally.

## 9. Security considerations

### 9.1 Threat model

This section enumerates attackers and the protection this protocol does or does
not provide.

**In scope.** The protocol defends against the following adversaries:

- **Network MITM on the path between two Servers.** Defense: HTTPS with Web PKI
  certificate validation (§9.2), TLS 1.2+ minimum.
- **Another federated peer attempting to impersonate a third peer.** Defense:
  per-Domain Ed25519 keys pinned via TOFU (§4.3), header-and-body authentication
  of every request (§6.2), `origin_domain` ↔ `sender_email` domain ↔
  `X-Federation-Sender` triple check.
- **Replay of a previously-captured signed request.** Defense: 10-minute nonce
  window and 5-minute clock-skew window (§6.2), mandatory `Date` header.
- **A peer's past key being reused after rotation.** Defense: explicit
  `previous_public_keys` with bounded rotation window (§9.5).
- **Traffic-log mining of query parameters.** Defense: sensitive lookups use
  POST with JSON bodies (§8.3), not GET with query strings.
- **An attacker attempting to cause double-delivery by replaying an envelope.**
  Defense: `envelope_id` dedup per-sender, retention ≥ 96h (§5.4).

**Out of scope.** This protocol does not defend against:

- **Compromise of a sender's private key at the Server.** Once the Origin's
  signing key is exfiltrated, the attacker can sign arbitrary envelopes as that
  Origin. Key protection is an operator concern.
- **Compromise of a certificate authority in the Web PKI.** Defense is
  Certificate Transparency monitoring, which is out of scope for this document.
- **A rogue authoritative Registrar in a peer's trusted set.** A peer who trusts
  a misbehaving Registrar may receive misrouted outbound traffic. Defense is
  curation (§8.6).
- **Content-level confidentiality from the Destination.** The Destination holds
  the plaintext body; there is no end-to-end encryption between end users.
- **Denial-of-service attacks by legitimate peers.** Rate-limiting is advisory
  (§9.10).
- **Spam and content-based abuse.** Out of scope.
- **Compromise of a User's External inbox during a §8.5 claim window.** See §8.5
  threat model.

### 9.2 Transport

All endpoints defined in this document **MUST** be served exclusively over HTTPS
at `webmail.{domain}`. Specifically:

- TLS 1.2 or TLS 1.3 **MUST** be negotiated. TLS versions below 1.2 **MUST** be
  rejected.
- The presented X.509 certificate **MUST** be valid for the hostname
  `webmail.{domain}`, **MUST** chain to a CA in the fetching peer's Web PKI
  trust store, and **MUST** have a valid not-before / not-after window. Subject
  Alternative Names covering `webmail.{domain}` are required; subject Common
  Name matches **MUST NOT** be relied on.
- Domain-validated (DV) certificates are the minimum acceptable validation
  level. Self-signed certificates, private-CA certificates not in the Web PKI,
  and IP-address certificates **MUST** be rejected at the TLS layer before any
  protocol bytes are read.
- HSTS (`Strict-Transport-Security`) **SHOULD** be served with
  `max-age ≥ 15552000` (180 days) and `includeSubDomains` at operator
  discretion.
- Plaintext HTTP **MUST NOT** be used, including as a redirect source. Peers
  encountering any `http://` URL in a descriptor, configuration, or redirect
  **MUST** abort the request.

A peer **MUST NOT** accept or generate protocol URLs rooted at any hostname
other than `webmail.{domain}`. Configuration that specifies alternative
hostnames (`mail.`, `api.`, the apex, IP addresses, `.onion`, `.local`) **MUST**
be rejected at configuration load time.

### 9.3 Authentication scope

Signatures in Webmail-Federation authenticate **the Domain**, not the end user.
A recipient **MUST NOT** infer from a valid envelope signature that the message
content was authored by the claimed `sender_email`; the only assertion is that
the `origin_domain` Domain stands behind the content. End-user authentication is
out of scope.

The Customer layer (§1.2) is not authenticated and is not visible on the wire. A
peer **MUST NOT** act on any claim about Customer membership because no such
claim is carried by this protocol. Implementations **MUST NOT** add headers,
envelope fields, or descriptor fields that disclose Customer identity, Customer
membership across Domains, or the number of Domains a Customer owns.

### 9.4 Replay and clock skew

The `X-Federation-Nonce` and `Date` header requirements in §6.2 exist to limit
replay and stale-request windows. Destinations **MUST** retain nonce state for
at least 10 minutes per sender per endpoint-kind (inbound, claim,
pending-shadows). Nonce memory **MAY** be partitioned per endpoint-kind to bound
the state size.

### 9.5 Key rotation

The protocol supports non-disruptive key rotation via the `previous_public_keys`
array in the peer descriptor (§4.2):

1. To rotate, a Domain generates a new Ed25519 keypair.
2. It publishes a new descriptor with `public_key` = new key and
   `previous_public_keys` = `[old key]`.
3. All outgoing signatures are now made with the new key.
4. Peers refresh descriptors on their own schedule (up to 24 hours). During that
   refresh gap, peers verify incoming signatures against whatever keyset is
   currently cached: the incoming signature with the new key will fail
   verification, which triggers a forced refresh per §4.3.
5. After 48 hours, the rotating Domain removes the old key from
   `previous_public_keys`. Peers that have not observed this by then will have
   refreshed descriptors (because subsequent incoming signatures using the new
   key force a refresh within 24h of first such signature).
6. A key **MUST NOT** appear in `public_key` or `previous_public_keys` more than
   48 hours after it has been superseded.

Rationale: TOFU pinning without a rotation mechanism is operationally
catastrophic; explicit `previous_public_keys` avoids relying on out-of-band key
distribution and keeps the wire format self-describing.

### 9.6 Peer identity and spoofing

A Destination **MUST** reject any envelope where `envelope.origin_domain`,
`message.sender_email`'s domain part, and `X-Federation-Sender` do not all
agree.

When a Registrar re-dispatches an envelope (§8.5), the original Origin's domain
remains the authoritative `origin_domain`. The Registrar **MUST NOT** forge
signatures as another Origin.

### 9.7 Allowlisting and refusal

An operator **MAY** configure an allowlist of permitted peer domains; envelopes
from peers not on the allowlist **MUST** be rejected with
`403 peer_not_allowed`. An operator **MAY** additionally refuse traffic from any
peer for operational reasons using `403`. These mechanisms are not authenticated
at the protocol level; they are policy.

### 9.8 Federation hostname and DNS trust

Locating the peer descriptor at `webmail.{domain}` rather than at the apex is
consistent with RFC 8615 §3, which delegates the hostname-scope choice to the
application specification. The choice carries an implicit trust assumption: a
descriptor at `webmail.example.com` claims authority over mailboxes at
`example.com`, a parent DNS label. RFC 8615 §4.3 warns against policy published
at one host applying to another; here, that cross-host assumption is deliberate
and load-bearing.

The trust root is DNS: whoever controls the `example.com` zone controls both the
`webmail` label and the authoritative nameservers, and therefore can publish
descriptors and direct address-level routing. This is the same trust root SMTP
and MX records rely on. No stronger anchor is assumed.

Consequences:

- A peer **MUST NOT** accept a descriptor for DNS domain `D` unless it was
  fetched from `https://webmail.D/.well-known/webmail-federation` over a TLS
  connection whose certificate is valid for `webmail.D` under the Web PKI.
  Fetching from any other host does not establish authority over `D`.
- An operator who publishes `webmail.{domain}` is asserting authority for
  `{domain}`. There is no scoped or non-authoritative mode.

### 9.9 Privacy

The peer descriptor is public. Domains **MUST NOT** include user data, account
counts, or internal topology in it.

`/registrar/lookup` (§8.3) reveals, for a given External-address hash, whether a
binding exists and what federated canonical it resolves to. Because the endpoint
uses POST with the hash in the body (not in the URL), the hash is not written to
HTTP access logs by standard servers. Operators **SHOULD** ensure their logging
configuration does not opt into body logging. The endpoint is still an oracle
for anyone who possesses the plaintext External address; knowing the address is
prerequisite to meaningful querying.

`/registrar/pending-shadows` (§8.4) reveals the existence of unclaimed messages
for a given External-address hash. This is a stronger disclosure than `/lookup`
because it signals activity, not just binding. Access is peer-authenticated, and
operators **SHOULD** treat it as sensitive peer metadata: a rogue entry in
`registrar_peers` can enumerate queried addresses. The trust model (§8.6)
assumes operators monitor peer behavior.

Timing side-channels on authenticated endpoints are not mitigated by this
document.

### 9.10 Abuse and rate limiting

A Destination **SHOULD** apply per-sender rate limits and per-sender daily
quotas, with explicit `429` responses carrying `retry_after_seconds`. A
Destination **MAY** refuse traffic from any peer using `403 peer_not_allowed`.
Registrars **SHOULD** additionally rate-limit `/registrar/lookup` per source IP
or per signing sender to mitigate enumeration attempts.

## 10. Versioning

The `versions` field in the peer descriptor is the authoritative statement of
what versions a Domain supports. Origins select exactly one version per outbound
envelope:

- The Origin **MUST** pick a `version` string that appears in the Destination's
  advertised `versions`.
- The Origin **SHOULD** pick the highest version string both peers advertise,
  ordered by string-ordered dotted-decimal comparison (e.g. `0.3` > `0.2` >
  `0.1`). This is not strictly a negotiation — it's Origin-side selection with
  Destination-side validation.
- The Destination **MUST** accept every version it advertises in its current
  descriptor.
- The Destination **MUST** reject with `400 unsupported_version` any envelope
  whose `version` does not appear in its current advertised `versions`.

Future versions **MAY** add optional envelope or descriptor fields. Unknown
fields in an otherwise valid envelope or descriptor **MUST** be ignored by the
receiver (with no side effects). Breaking changes **MUST** be expressed as a new
`version` string; an implementation that cannot speak the new version continues
to speak the old.

## 11. Conformance

A **Basic Webmail-Federation Domain** **MUST** implement:

- §4 (peer discovery, TOFU, rotation via `previous_public_keys`).
- §5 (envelope format, field rules, idempotency and retention).
- §6 (inbound transfer, signature, responses, retry, bounces).
- §7 (inline attachments with size caps and hash verification).
- §9.1–§9.5, §9.8 (transport, auth scope, replay, rotation, DNS trust).
- §10 (versioning and unknown-field policy).

A **Registrar Domain** **MUST** implement all of the above, plus:

- §8.1–§8.7 (every Registrar endpoint).
- §9.6 peer identity (strict, with attention to Registrar re-dispatch rules).
- §9.9 privacy considerations specific to `/lookup` and `/pending-shadows`.

An implementation that cannot satisfy the Basic tier is not a conformant
Webmail-Federation Domain. An implementation that implements some but not all of
the Registrar endpoints **MUST NOT** advertise `"registrar": true`.

## 12. IANA considerations

This document requests registration of the following well-known URI suffix under
RFC 8615:

- **URI Suffix:** `webmail-federation`
- **Change Controller:** The Webmail-Federation project.
- **Specification:** This document.
- **Related Information:** The suffix identifies the peer descriptor (§4.2).

## 13. References

### 13.1 Normative references

- **RFC 2119** — Key words for use in RFCs to Indicate Requirement Levels.
- **RFC 3339** — Date and Time on the Internet: Timestamps.
- **RFC 4648** — The Base16, Base32, and Base64 Data Encodings.
- **RFC 5321** — Simple Mail Transfer Protocol (for addr-spec syntax).
- **RFC 5322** — Internet Message Format (for addr-spec syntax and line-length
  guidance).
- **RFC 6838** — Media Type Specifications and Registration Procedures.
- **RFC 7231** — HTTP/1.1: Semantics and Content (including `Date` header and
  `IMF-fixdate`).
- **RFC 8032** — Edwards-Curve Digital Signature Algorithm (EdDSA), specifically
  Ed25519.
- **RFC 8174** — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.
- **RFC 8259** — The JavaScript Object Notation (JSON) Data Interchange Format.
- **RFC 8446** — The Transport Layer Security (TLS) Protocol Version 1.3.
- **RFC 8615** — Well-Known Uniform Resource Identifiers.
- **RFC 9562** — Universally Unique IDentifiers (UUIDs), including UUIDv7.

### 13.2 Informative references

- RFC 8615 §3 (delegated hostname scope) and §4.3 (cross-host policy caveat),
  both cited in §9.8.

## Appendix A. Example exchange

### A.1 Origin-side send

Alice at `103mail.com` composes a message to `bob@example.com` and
`carol@example.com`.

The Origin splits fan-out by domain and, for `example.com`, fetches (or reuses
cached) `https://webmail.example.com/.well-known/webmail-federation`, then
issues:

```
POST https://webmail.example.com/api/v1/federation/inbound
Host: webmail.example.com
Content-Type: application/json; charset=utf-8
Content-Length: 612
Date: Thu, 23 Apr 2026 10:15:00 GMT
X-Federation-Version:   0.2
X-Federation-Sender:    103mail.com
X-Federation-Nonce:     AbCdEfGhIjKlMnOpQrStUw
X-Federation-Signature: ed25519:YTa8u3VX4uTGz...ZqJ0OQ== (88 chars)

{
  "version": "0.2",
  "envelope_id": "018f4d2b-9f21-7890-abcd-ef0123456789",
  "origin_domain": "103mail.com",
  "origin_sent_at": "2026-04-23T10:15:00.000Z",
  "thread": {
    "federation_id": "018f4d2b-9f21-7890-abcd-ef012345678a",
    "subject": "Hello from 103mail",
    "in_reply_to": null
  },
  "message": {
    "federation_id": "018f4d2b-9f21-7890-abcd-ef012345678b",
    "sender_email": "1234567@103mail.com",
    "sender_name": "Alice",
    "body": "Hi Bob and Carol — ...",
    "body_html": "<p>Hi Bob and Carol — ...</p>",
    "sent_at": "2026-04-23T10:15:00.000Z"
  },
  "attachments": [],
  "recipients": ["bob@example.com", "carol@example.com"]
}
```

### A.2 Destination-side accept

```
HTTP/1.1 202 Accepted
Content-Type: application/json

{"envelope_id": "018f4d2b-9f21-7890-abcd-ef0123456789", "status": "accepted"}
```

The Destination writes per-sender state (the specific schema is out of scope),
onboards or matches Bob and/or Carol as required by its internal policy, and
emits its notifications linking back to `https://webmail.example.com`.

### A.3 Reply

Carol replies. Her Origin Domain (`example.com`) issues a new envelope back to
`103mail.com` with `thread.federation_id` set to the value from A.1 and
`thread.in_reply_to` equal to the original message's `federation_id`.
`103mail.com` matches by `thread.federation_id` and appends to the existing
thread.

## Appendix B. Test vector

The following end-to-end vector enables an implementation to verify its
signature and envelope handling.

**Origin private key** (Ed25519 seed, 32 bytes, hex):

```
9d 61 b1 9d ef fd 5a 60 ba 84 4a f4 92 ec 2c c4
44 49 c5 69 7b 32 69 19 70 3b ac 03 1c ae 7f 60
```

**Derived public key** (32 bytes, hex):

```
d7 5a 98 01 82 b1 0a b7 d5 4b fe d3 c9 64 07 3a
0e e1 72 f3 da a6 23 25 af 02 1a 68 f7 07 51 1a
```

Which in descriptor form is:

```
"public_key": "ed25519:11qYAYKxCrfVS/7TyWQHOg7hcvPapiMlrwIaaPcHURo="
```

**Request headers** (exact bytes, CRLF line endings):

```
POST /api/v1/federation/inbound HTTP/1.1
Host: webmail.example.com
Content-Type: application/json; charset=utf-8
Content-Length: 37
Date: Thu, 23 Apr 2026 10:15:00 GMT
X-Federation-Version: 0.2
X-Federation-Sender: 103mail.com
X-Federation-Nonce: AbCdEfGhIjKlMnOpQrStUw
```

**Request body** (exact bytes, 37 bytes, no trailing newline):

```
{"version":"0.2","envelope_id":"x"}
```

(This body is intentionally minimal for test-vector readability; in a real
request the body is the full envelope of §5.2.)

**Signing input** (concatenation per §6.2; UTF-8 with explicit 0x0A separators,
total length = sum of part lengths):

```
"0.2" 0x0A "103mail.com" 0x0A "AbCdEfGhIjKlMnOpQrStUw" 0x0A
"Thu, 23 Apr 2026 10:15:00 GMT" 0x0A
body_bytes
```

An implementation is conformant if, given the private key above and the request
above, it produces an `X-Federation-Signature` header that a verifier using the
listed public key successfully validates. Implementations **SHOULD** publish
their computed signature bytes in test logs during development; the exact base64
encoding is deterministic given the inputs.

## Appendix C. Change log

- **0.2 (2026-04-23):**
  - Signature: dropped pre-SHA-256, added header coverage, specified 0x0A
    separators and exact byte encodings. Ed25519 signs directly.
  - Key rotation: added `previous_public_keys` descriptor field and 48-hour
    rotation window (§9.5).
  - Descriptor: added size cap (65536 bytes), removed `backend` and `inbound` /
    `inbound_path` fields — every endpoint path is fixed by §4.2 and derived
    from `domain`; specified base64 and key byte-length validation.
  - Envelope: added required/optional/length rules for every field (§5.3),
    capped `recipients` (100) and `attachments` (64), specified millisecond
    timestamp resolution.
  - Retention: dedup window bounded at 96 hours; retry budget fixed at 72 hours
    (was "at least 48").
  - Attachments: unified on hex SHA-256 (`hash_sha256_hex`), removed base64 hash
    variant.
  - §6.3: added `415 unsupported_media_type`; mandatory `Date` header.
  - §8.3: `/registrar/lookup` moved to POST with JSON body to keep hashes out of
    access logs.
  - §8.5: explicit token threat model, signature-forgery prohibition for
    Registrar re-dispatch.
  - §8.6: peer-removal cache purge hardened from SHOULD to MUST within 60s.
  - §9: new §9.1 threat model, new §9.5 key rotation, tightened §9.2 transport,
    §9.9 privacy expansion.
  - §10: recast "negotiation" as Origin-side selection.
  - §11 new: conformance tiers (Basic vs Registrar).
  - §12: IANA registration requested for `webmail-federation` well-known suffix.
  - §13: expanded normative references.
  - Appendix A signature example replaced (was ECDSA prefix `MEU...`); test
    vectors added as Appendix B.
- **0.1 (2026-04-18):** initial draft.
