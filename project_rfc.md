# RFC: Webmail-Federation Protocol

- **Name:** Webmail-Federation
- **Version:** 0.1 (draft)
- **Status:** Draft — not yet implemented
- **Date:** 2026-04-18
- **Editor:** Maurits van der Schee

## Abstract

This document specifies Webmail-Federation, an HTTP-based protocol for exchanging messages between independently operated Webmail servers. Each participating server is the authoritative host for exactly one internet domain and for the mailboxes local to it. Webmail-Federation replaces SMTP with a narrower, authenticated, JSON-over-HTTPS envelope exchange. The protocol covers peer discovery, envelope format, cryptographic authentication, idempotent delivery, attachment transfer, and error semantics. It does not cover user authentication, end-to-end encryption, or spam filtering; those concerns are addressed elsewhere.

## Status of this memo

This is a working draft. It is intended to become the reference protocol specification for the `webmail-federation` project. Until version 1.0 is published, the wire format and endpoint paths are subject to incompatible change.

## 1. Conventions and terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

The following terms are used throughout:

- **Server** — A deployment of the Webmail backend authoritative for exactly one domain.
- **Domain** — A DNS domain served by exactly one Server.
- **Local recipient** — An account whose address ends in the Server's own domain.
- **Remote recipient** — An account whose address ends in any other domain.
- **Origin Server** — The Server whose user composes a new message.
- **Destination Server** — The Server authoritative for a Remote recipient.
- **Envelope** — The JSON document transferred between Servers for one message.
- **Peer descriptor** — The JSON document published at a well-known URL describing a Server to other Servers.

## 2. Goals and non-goals

### 2.1 Goals

- Permit a user at domain `A` to send a message to a user at domain `B` when `A` and `B` are served by separate Webmail Servers.
- Preserve the no-SMTP property: no message body is delivered to a host that is not a Webmail Server.
- Authenticate every inter-Server request so that a Server cannot spoof messages from a third domain.
- Be idempotent under retry; be deliverable offline by the sending Server.

### 2.2 Non-goals

- Transparent relay through intermediaries: messages flow Origin → Destination directly.
- Backwards compatibility with SMTP, IMAP, JMAP, or any existing internet mail protocol.
- End-to-end encryption or signing of message content for end-user verification. Signatures in this protocol authenticate Server-to-Server transport only.
- Address portability across domains.

## 3. Protocol overview

A Webmail-Federation exchange for a single message proceeds as follows:

```
Origin Server                                 Destination Server
      |                                                 |
      |  1. GET /.well-known/webmail-federation         |
      |------------------------------------------------>|
      |                                                 |
      |<------------------------------------------------|
      |         peer descriptor (cached by Origin)      |
      |                                                 |
      |  2. POST {inbound} (signed envelope)            |
      |------------------------------------------------>|
      |                                                 |
      |<------------------------------------------------|
      |                 202 Accepted                    |
```

Step 1 is cacheable. Step 2 is the normative transfer event. All message data — including attachment bytes — is carried inside the envelope in step 2; there is no separate fetch step.

## 4. Peer discovery

### 4.1 Well-known endpoint

Each Server **MUST** publish a peer descriptor at:

```
https://{domain}/.well-known/webmail-federation
```

over HTTPS with a valid certificate chain for `{domain}`. The endpoint **MUST** respond with `Content-Type: application/json` and **MUST NOT** require authentication.

### 4.2 Peer descriptor

The descriptor is a JSON object with the following members:

| Field         | Type    | Required | Description                                                     |
|---------------|---------|----------|-----------------------------------------------------------------|
| `domain`      | string  | yes      | The DNS domain the Server is authoritative for.                 |
| `backend`     | string  | yes      | Absolute base URL of the Server's API.                          |
| `inbound`     | string  | yes      | Absolute URL of the inbound envelope endpoint (see §6).         |
| `public_key`  | string  | yes      | Ed25519 public key for transport signing, as `ed25519:{base64}`.|
| `versions`    | array   | yes      | List of protocol versions the Server understands, e.g. `["0.1"]`. |
| `max_envelope_bytes` | integer | no | Advertised upper bound on inbound envelope body size.           |
| `max_attachment_bytes` | integer | no | Advertised upper bound on a single attachment payload.        |

Example:

```json
{
  "domain": "example.com",
  "backend": "https://api.example.com",
  "inbound": "https://api.example.com/api/v1/federation/inbound",
  "public_key": "ed25519:MCowBQYDK2VwAyEA...",
  "versions": ["0.1"],
  "max_envelope_bytes": 10485760,
  "max_attachment_bytes": 26214400
}
```

### 4.3 Trust-on-first-use and caching

A Server **MUST** cache the peer descriptor keyed by domain. A Server **MUST** store the `public_key` value observed on the first successful fetch and use that value to verify all subsequent inbound signatures from that domain (trust on first use). The cache entry **SHOULD** be refreshed no more often than once per 24 hours, and **MUST** be refreshed when signature verification fails with the cached key.

A Server **MAY** expose an operator-controlled allowlist of permitted peer domains; if configured, envelopes from domains not on the allowlist **MUST** be rejected with `403 Forbidden` (see §8).

## 5. Envelope format

### 5.1 Content type

Envelopes are JSON objects transferred with `Content-Type: application/json; charset=utf-8`.

### 5.2 Envelope schema

```json
{
  "version": "0.1",
  "envelope_id": "2c1e9b4a-...-uuidv7",
  "origin_domain": "103mail.com",
  "origin_sent_at": "2026-04-18T10:15:00Z",
  "thread": {
    "federation_id": "8e2f9c...-uuidv7",
    "subject": "Hello from 103mail",
    "in_reply_to": null
  },
  "message": {
    "federation_id": "7a31d0...-uuidv7",
    "sender_email": "1234567@103mail.com",
    "sender_name": "Alice",
    "body": "...plain text...",
    "body_html": "<p>...</p>",
    "sent_at": "2026-04-18T10:15:00Z"
  },
  "attachments": [
    {
      "federation_id": "11aa22bb-...",
      "file_name": "report.pdf",
      "mime_type": "application/pdf",
      "file_size": 138241,
      "hash_sha256": "base64(...)",
      "data": "base64(...)"
    }
  ],
  "recipients": [
    "bob@example.com",
    "carol@example.com"
  ]
}
```

### 5.3 Field rules

- `version` **MUST** be a version string the Destination advertises in `versions` of its peer descriptor. Otherwise the Destination **MUST** respond `400 Bad Request` with `error = "unsupported_version"`.
- `envelope_id`, `thread.federation_id`, `message.federation_id`, and `attachments[].federation_id` **MUST** be UUIDv7 (RFC 9562) values.
- `origin_domain` **MUST** match the domain portion of every `message.sender_email` in the envelope and **MUST** match the `X-Federation-Sender` header (§6).
- `recipients` **MUST** contain only addresses whose domain is the Destination's domain. Origin Servers **MUST** split fan-out by destination domain and send one envelope per Destination.
- Timestamps **MUST** be RFC 3339 / ISO 8601 UTC with a trailing `Z`.
- `thread.in_reply_to`, if present, **MUST** be a `thread.federation_id` previously seen by both Servers.
- Every attachment's payload **MUST** be carried inline in `data` as base64 (see §7). Out-of-band or deferred attachment transfer is not supported.

### 5.4 Idempotency

An envelope is identified uniquely by `envelope_id`. A Destination Server **MUST** deduplicate envelopes by `envelope_id` per origin domain. Re-delivery of the same `envelope_id` **MUST** produce the same outcome (202 Accepted) without re-inserting messages into mailboxes.

## 6. Inbound transfer

### 6.1 Request

The Origin Server issues:

```
POST {inbound}
Host: {destination host}
Content-Type: application/json; charset=utf-8
Content-Length: ...
X-Federation-Version:   0.1
X-Federation-Sender:    {origin_domain}
X-Federation-Signature: ed25519:{base64(signature)}
X-Federation-Nonce:     {unique per request}

{envelope JSON}
```

### 6.2 Signature

The signature is computed as:

```
signature = Ed25519-sign(
  private_key_of(origin_domain),
  SHA-256( request_body_bytes || "\n" || X-Federation-Nonce )
)
```

The Destination **MUST** reject with `401 Unauthorized` (`error = "bad_signature"`) if:

- The signature does not verify under the cached public key for `X-Federation-Sender`.
- `X-Federation-Sender` does not match `envelope.origin_domain`.
- `X-Federation-Nonce` has been seen within the last 10 minutes from the same sender.
- The `Date` header (if present) is more than 5 minutes from the Destination's clock.

### 6.3 Responses

| Status | When                                                                         | Body                                                                |
|--------|------------------------------------------------------------------------------|---------------------------------------------------------------------|
| 202    | Envelope accepted (new or idempotent replay).                                | `{"envelope_id": "...", "status": "accepted"}`                      |
| 400    | Malformed envelope, unsupported version, schema violation.                   | `{"error": "...", "detail": "..."}`                                 |
| 401    | Signature, nonce, or sender check failed.                                    | `{"error": "bad_signature"}`                                        |
| 403    | Origin domain is not on the Destination's allowlist.                         | `{"error": "peer_not_allowed"}`                                     |
| 404    | All `recipients` are unknown at the Destination and none can be onboarded.   | `{"error": "no_recipients"}`                                        |
| 413    | Envelope body exceeds `max_envelope_bytes`.                                  | `{"error": "too_large"}`                                            |
| 429    | Rate limit exceeded.                                                         | `{"error": "rate_limited", "retry_after_seconds": n}`               |
| 5xx    | Transient Destination failure.                                               | `{"error": "server_error"}`                                         |

Errors in the 4xx range (except 408, 425, 429) are **permanent**: the Origin **MUST NOT** retry the same envelope. Errors in the 5xx range and the retryable 4xx (408, 425, 429) are **transient** and **SHOULD** be retried (§6.4).

### 6.4 Retry policy

On transient failure, the Origin Server **MUST** retry using exponential backoff with full jitter, starting at 30 seconds and capped at 1 hour, for at least 48 hours. After the retry budget is exhausted, the Origin Server **MUST** generate a local bounce into the sender's mailbox referencing the final error.

On `429 Too Many Requests` with a `retry_after_seconds` field, the Origin Server **MUST** wait at least that long before the next attempt to that Destination.

### 6.5 Destination processing

On accepting an envelope the Destination **MUST**:

1. Record `(origin_domain, envelope_id)` for deduplication.
2. For each recipient in `recipients`, resolve or onboard a local account (same auto-onboarding path used today for local composition).
3. Insert a per-recipient `thread` / `message` row referencing the same `g_thread` / `g_message`. The federation identifiers **MUST** be stored alongside the local autoincrement ids to support reply correlation.
4. Emit the existing metadata-only notification (the `sent_email` row) to the recipient, with a link pointing at the Destination Server.

## 7. Attachment transfer

All attachment bytes **MUST** be carried inline in the envelope as base64 in `data`. After a Destination accepts an envelope, it holds a complete, self-contained copy of every attachment; no subsequent request to the Origin is ever required to read an attachment. This guarantees that a message, once accepted, remains readable even if the Origin Server is permanently decommissioned.

Requirements:

- The Origin **MUST** set `file_size` to the decoded byte length of `data` and `hash_sha256` to the SHA-256 of the decoded bytes.
- The Destination **MUST** recompute both values on receipt and reject the envelope with `400 Bad Request` (`error = "attachment_mismatch"`) if either differs.
- An Origin **MUST NOT** send an envelope whose serialized size exceeds the Destination's advertised `max_envelope_bytes`, nor an individual attachment exceeding `max_attachment_bytes`. A Destination receiving such an envelope **MUST** respond `413 Payload Too Large`.
- An Origin that needs to send an attachment larger than any Destination will accept **MUST** surface a permanent failure to the sender; this protocol does not define a fallback out-of-band transfer.

This design trades bandwidth and envelope size against simplicity, durability, and the "all data local to every server" invariant. That tradeoff is deliberate and **MUST NOT** be relaxed by extensions that introduce deferred, external, or pull-based attachment fetching.

## 8. Security considerations

### 8.1 Transport

All requests defined in this document **MUST** be served over TLS 1.2 or later with certificates valid for the domain being addressed. Plaintext HTTP **MUST NOT** be used, including for the well-known endpoint.

### 8.2 Authentication scope

Signatures in Webmail-Federation authenticate **the Server**, not the end user. A recipient **MUST NOT** infer from a valid envelope signature that the message content was authored by the claimed `sender_email`; they can only infer that the `origin_domain` Server asserts so. End-user authentication is out of scope.

### 8.3 Replay and clock skew

The `X-Federation-Nonce` and timestamp constraints in §6.2 exist to limit replay windows. Destinations **MUST** keep nonce state for at least 10 minutes per origin domain.

### 8.4 Key rotation

A Server **MAY** rotate its Ed25519 keypair by publishing a new `public_key` in its peer descriptor. Destinations observing a signature failure **MUST** re-fetch the descriptor before giving up. During a rotation window a Server **MAY** sign with the old key while publishing the new one for up to 48 hours; after that window the old key **MUST NOT** be used.

### 8.5 Peer identity and spoofing

A Destination **MUST** reject any envelope where `envelope.origin_domain`, `message.sender_email`'s domain, and `X-Federation-Sender` do not all agree. This prevents one authenticated peer from impersonating another.

### 8.6 Abuse, rate limiting, and content

This document does not specify anti-abuse rules. A Destination Server **SHOULD** apply per-peer rate limits, per-peer daily quotas, and content-based filters before accepting envelopes for delivery, and **MAY** refuse traffic from peers at its discretion using `403 Forbidden`.

### 8.7 Privacy

The peer descriptor (§4.2) is public. Servers **MUST NOT** include user data, account counts, or internal topology in it.

## 9. Versioning and evolution

A Server **MUST** reject envelopes whose `version` is not listed in its advertised `versions`. Future versions of this protocol **MAY** add optional fields; unknown fields in an otherwise valid envelope **MUST** be ignored by the Destination. Breaking changes **MUST** be expressed as a new `version` string, and both peers **MUST** negotiate the highest version they both support.

## 10. IANA considerations

This document requests no IANA actions. The well-known path `/.well-known/webmail-federation` is reserved for the purposes described herein and is not at present registered with IANA.

## 11. References

- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
- RFC 8174 — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.
- RFC 3339 — Date and Time on the Internet: Timestamps.
- RFC 8032 — Edwards-Curve Digital Signature Algorithm (EdDSA).
- RFC 8615 — Well-Known Uniform Resource Identifiers.
- RFC 9562 — Universally Unique IDentifiers (UUIDs), including UUIDv7.

## Appendix A. Example exchange

### A.1 Origin-side send

Alice at `103mail.com` composes a message to `bob@example.com` and `carol@example.com`.

The Origin Server splits fan-out by domain and, for `example.com`, fetches (or reuses cached) `https://example.com/.well-known/webmail-federation`, then issues:

```
POST https://api.example.com/api/v1/federation/inbound
Content-Type: application/json; charset=utf-8
X-Federation-Version:   0.1
X-Federation-Sender:    103mail.com
X-Federation-Nonce:     01HN2K4Z9X-...
X-Federation-Signature: ed25519:MEUCIQD...

{
  "version": "0.1",
  "envelope_id": "018f4d2b-...-7ac1",
  "origin_domain": "103mail.com",
  "origin_sent_at": "2026-04-18T10:15:00Z",
  "thread": {
    "federation_id": "018f4d2b-...-thread",
    "subject": "Hello from 103mail",
    "in_reply_to": null
  },
  "message": {
    "federation_id": "018f4d2b-...-msg",
    "sender_email": "1234567@103mail.com",
    "sender_name": "Alice",
    "body": "Hi Bob and Carol — ...",
    "body_html": "<p>Hi Bob and Carol — ...</p>",
    "sent_at": "2026-04-18T10:15:00Z"
  },
  "attachments": [],
  "recipients": ["bob@example.com", "carol@example.com"]
}
```

### A.2 Destination-side accept

```
HTTP/1.1 202 Accepted
Content-Type: application/json

{"envelope_id": "018f4d2b-...-7ac1", "status": "accepted"}
```

The Destination then writes `g_thread`, `g_message`, and two per-recipient `thread` / `message` rows (onboarding Bob and/or Carol if needed) and emits a `sent_email` notification to each, linking back to `https://mail.example.com`.

### A.3 Reply

Carol replies. Her Origin Server (example.com) issues a new envelope back to `103mail.com` with `thread.federation_id` set to the value Carol received in A.1 and `thread.in_reply_to` equal to the message's `federation_id`. 103mail matches on `federation_id` and appends to the existing `g_thread`.
