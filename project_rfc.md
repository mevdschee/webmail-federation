# RFC: Webmail-Federation Protocol

- **Name:** Webmail-Federation
- **Version:** 0.1 (draft)
- **Status:** Draft — not yet implemented
- **Date:** 2026-04-18
- **Editor:** Maurits van der Schee

## Abstract

This document specifies Webmail-Federation, an HTTP-based protocol for exchanging messages between independently operated Webmail Domains. Each Domain is the authoritative host for exactly one internet domain and for the mailboxes local to it; a single physical Server may host one or more Domains, each appearing to its peers as an independent participant. Webmail-Federation replaces SMTP with a narrower, authenticated, JSON-over-HTTPS envelope exchange. The protocol covers peer discovery, envelope format, cryptographic authentication, idempotent delivery, attachment transfer, and error semantics. It does not cover user authentication, end-to-end encryption, or spam filtering; those concerns are addressed elsewhere.

## Status of this memo

This is a working draft. It is intended to become the reference protocol specification for the `webmail-federation` project. Until version 1.0 is published, the wire format and endpoint paths are subject to incompatible change.

## 1. Conventions and terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

The following terms are used throughout:

- **Domain** — The logical Webmail-Federation participant. A Domain is authoritative for exactly one DNS domain and for the mailboxes local to it. All protocol rules in this document apply per-Domain.
- **Server** — A physical deployment of the Webmail backend (process + database). A Server hosts one or more Domains; each Domain hosted on the same Server has its own DNS domain, peer descriptor, and signing keypair, and is indistinguishable from a single-Domain Server to its peers. "Server" is an operational term and does not appear on the wire.
- **Local recipient** — An account whose address ends in the Domain's own DNS domain.
- **Remote recipient** — An account whose address ends in any other DNS domain.
- **Origin Domain** — The Domain whose user composes a new message.
- **Destination Domain** — The Domain authoritative for a Remote recipient.
- **Envelope** — The JSON document transferred between Domains for one message.
- **Peer descriptor** — The JSON document published at a well-known URL describing a Domain to other Domains.
- **Registrar** — A Domain that additionally resolves external email addresses to Federation-enabled canonicals and accepts inbound envelopes whose recipients are external, per §8. A Domain is a Registrar if its peer descriptor advertises `registrar: true` (§4.2); otherwise it is a non-Registrar Domain. Non-Registrar is the default.
- **External address** — A mail address whose domain is neither the Destination's own DNS domain nor a Federation-enabled DNS domain (e.g. `alice@gmail.com`). External addresses appear only in envelopes sent to Registrars (§8.2).

## 2. Goals and non-goals

### 2.1 Goals

- Permit a user at domain `A` to send a message to a user at domain `B` when `A` and `B` are served by separate Webmail Domains.
- Preserve the no-SMTP property: no message body is delivered to a host that is not a Webmail Domain.
- Authenticate every inter-Domain request so that a Domain cannot spoof messages from a third domain.
- Be idempotent under retry; be deliverable offline by the sending Domain.
- Permit Domains that advertise the Registrar role (§8) to resolve external email addresses to Federation-enabled canonicals and route envelopes accordingly, without requiring a global registry.

### 2.2 Non-goals

- Transparent relay through intermediaries: messages flow Origin → Destination directly.
- Backwards compatibility with SMTP, IMAP, JMAP, or any existing internet mail protocol.
- End-to-end encryption or signing of message content for end-user verification. Signatures in this protocol authenticate Domain-to-Domain transport only.
- Address portability across domains.

## 3. Protocol overview

A Webmail-Federation exchange for a single message proceeds as follows:

```
Origin Domain                                 Destination Domain
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

Each Domain **MUST** publish a peer descriptor at:

```
https://{domain}/.well-known/webmail-federation
```

over HTTPS with a valid certificate chain for `{domain}`. The endpoint **MUST** respond with `Content-Type: application/json` and **MUST NOT** require authentication.

A Server hosting multiple Domains **MUST** serve a distinct peer descriptor at each Domain's DNS domain; descriptors **MUST NOT** be shared across DNS domains. In particular, each co-hosted Domain **MUST** carry its own `public_key`, because peers pin keys per DNS domain (§4.3) and a compromise of one Domain's key must not enable impersonation of another.

### 4.2 Peer descriptor

The descriptor is a JSON object with the following members:

| Field         | Type    | Required | Description                                                     |
|---------------|---------|----------|-----------------------------------------------------------------|
| `domain`      | string  | yes      | The DNS domain the Domain is authoritative for.                 |
| `backend`     | string  | yes      | Absolute base URL of the Domain's API.                          |
| `inbound`     | string  | yes      | Absolute URL of the inbound envelope endpoint (see §6).         |
| `public_key`  | string  | yes      | Ed25519 public key for transport signing, as `ed25519:{base64}`.|
| `versions`    | array   | yes      | List of protocol versions the Domain understands, e.g. `["0.1"]`. |
| `registrar`   | boolean | no       | `true` if the Domain is a Registrar. Defaults to `false`. Governs §8 behavior. |
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
  "registrar": false,
  "max_envelope_bytes": 10485760,
  "max_attachment_bytes": 26214400
}
```

### 4.3 Trust-on-first-use and caching

A Domain **MUST** cache the peer descriptor keyed by domain. A Domain **MUST** store the `public_key` value observed on the first successful fetch and use that value to verify all subsequent inbound signatures from that domain (trust on first use). The cache entry **SHOULD** be refreshed no more often than once per 24 hours, and **MUST** be refreshed when signature verification fails with the cached key.

A Domain **MAY** expose an operator-controlled allowlist of permitted peer domains; if configured, envelopes from domains not on the allowlist **MUST** be rejected with `403 Forbidden` (see §9).

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
- `recipients` **MUST** contain only addresses whose domain is the Destination's domain, except when the Destination is a Registrar (§8.2). Origin Domains **MUST** split fan-out by destination domain and send one envelope per Destination.
- Timestamps **MUST** be RFC 3339 / ISO 8601 UTC with a trailing `Z`.
- `thread.in_reply_to`, if present, **MUST** be a `thread.federation_id` previously seen by both Domains.
- Every attachment's payload **MUST** be carried inline in `data` as base64 (see §7). Out-of-band or deferred attachment transfer is not supported.

### 5.4 Idempotency

An envelope is identified uniquely by `envelope_id`. A Destination Domain **MUST** deduplicate envelopes by `envelope_id` per origin domain. Re-delivery of the same `envelope_id` **MUST** produce the same outcome (202 Accepted) without re-inserting messages into mailboxes.

## 6. Inbound transfer

### 6.1 Request

The Origin Domain issues:

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

On transient failure, the Origin Domain **MUST** retry using exponential backoff with full jitter, starting at 30 seconds and capped at 1 hour, for at least 48 hours. After the retry budget is exhausted, the Origin Domain **MUST** generate a local bounce into the sender's mailbox referencing the final error.

On `429 Too Many Requests` with a `retry_after_seconds` field, the Origin Domain **MUST** wait at least that long before the next attempt to that Destination.

### 6.5 Destination processing

On accepting an envelope the Destination **MUST**:

1. Record `(origin_domain, envelope_id)` for deduplication.
2. For each recipient in `recipients`, resolve or onboard a local account (same auto-onboarding path used today for local composition).
3. Insert a per-recipient `thread` / `message` row referencing the same `g_thread` / `g_message`. The federation identifiers **MUST** be stored alongside the local autoincrement ids to support reply correlation.
4. Emit the existing metadata-only notification (the `sent_email` row) to the recipient, with a link pointing at the Destination Domain.

## 7. Attachment transfer

All attachment bytes **MUST** be carried inline in the envelope as base64 in `data`. After a Destination accepts an envelope, it holds a complete, self-contained copy of every attachment; no subsequent request to the Origin is ever required to read an attachment. This guarantees that a message, once accepted, remains readable even if the Origin Domain is permanently decommissioned.

Requirements:

- The Origin **MUST** set `file_size` to the decoded byte length of `data` and `hash_sha256` to the SHA-256 of the decoded bytes.
- The Destination **MUST** recompute both values on receipt and reject the envelope with `400 Bad Request` (`error = "attachment_mismatch"`) if either differs.
- An Origin **MUST NOT** send an envelope whose serialized size exceeds the Destination's advertised `max_envelope_bytes`, nor an individual attachment exceeding `max_attachment_bytes`. A Destination receiving such an envelope **MUST** respond `413 Payload Too Large`.
- An Origin that needs to send an attachment larger than any Destination will accept **MUST** surface a permanent failure to the sender; this protocol does not define a fallback out-of-band transfer.

This design trades bandwidth and envelope size against simplicity, durability, and the "all data local to every Domain" invariant. That tradeoff is deliberate and **MUST NOT** be relaxed by extensions that introduce deferred, external, or pull-based attachment fetching.

## 8. Registrar extension

This section defines additional wire protocol that applies only to Domains advertising `registrar: true` in their peer descriptor (§4.2). Non-Registrar Domains are unaffected.

### 8.1 Registrar advertisement

A Domain **MAY** advertise `registrar: true` in its peer descriptor. A Domain whose descriptor omits `registrar`, or whose `registrar` is `false`, **MUST NOT** be treated as a Registrar by its peers: its peers **MUST NOT** send such a Domain envelopes containing external addresses (§8.2) and **MUST NOT** query it on the endpoints defined in §8.3 and §8.4.

Registrar capability is declared at descriptor-refresh time only; a peer that observes a change in `registrar` **MUST** honor the new value from the next cache refresh onward (§4.3).

### 8.2 Recipient rule relaxation

Notwithstanding §5.3, when an envelope is delivered to a Domain whose current peer descriptor advertises `registrar: true`:

- `recipients` **MAY** contain external addresses, as defined in §1.
- The Destination **MUST NOT** reject such an envelope on grounds of `recipients` domain mismatch.
- The Destination is responsible for resolving or onboarding each external recipient. The internal model used (mapping table, onboarding state machine, claim lifecycle) is not specified by this document.

An Origin **MUST NOT** include external addresses in `recipients` when sending to a Domain that does not advertise `registrar: true`. Such an envelope, if received, **MUST** be rejected with `400 Bad Request` and `error = "not_registrar"`.

An envelope whose `recipients` contain a mix of Destination-domain addresses and external addresses is valid only when the Destination is a Registrar. An Origin **MAY** always choose to split such an envelope by recipient class and send multiple envelopes.

For any given external-address recipient in an outbound send, an Origin **MUST** select exactly one Registrar as the Destination for that recipient and **MUST** keep that selection stable across retries of the same envelope. An Origin **MUST NOT** fan out the same external-address recipient to multiple Registrars in parallel, nor re-select a different Registrar on a subsequent attempt for a recipient that has not yet been acknowledged, because doing so may cause two Registrars to independently onboard the same external address and produce divergent routing state. How an Origin chooses the Registrar (static configuration, ordered list, policy) is out of scope; the stability and single-destination requirements are not.

An Origin that is itself a Registrar **MAY** resolve external-address recipients locally under §8.3 rather than selecting a peer Registrar as Destination. A non-Registrar Origin with no Registrar available to it cannot send external-addressed envelopes at all; such a configuration is permitted but precludes the sends in question.

### 8.3 Registrar lookup

A Registrar **MUST** expose:

```
GET {backend}/api/v1/registrar/lookup?hash={sha256_hex(lowercase(address))}
```

The `hash` query parameter is the hexadecimal SHA-256 digest of the lowercased external address being resolved. The request carries no body and requires no authentication.

| Status | When                                 | Body                                                                                                 |
|--------|--------------------------------------|------------------------------------------------------------------------------------------------------|
| 200    | Authoritative record exists.         | `{"canonical": "...", "forward_canonical": "..." \| null, "issued_at": "...", "expires_at": "..."}`  |
| 404    | No authoritative record.             | `{"error": "not_found"}`                                                                             |
| 429    | Rate limit exceeded.                 | `{"error": "rate_limited", "retry_after_seconds": n}`                                                |

Rules:

- A Registrar **MUST** answer only from its own authoritative state. It **MUST NOT** recursively resolve through peer Registrars and serve the result as its own answer.
- A Registrar **MUST NOT** return records whose `expires_at` is in the past.
- A Registrar **MAY** cache lookup responses obtained from peer Registrars keyed by `(hash, responder_domain)` for use in its own routing decisions; such cached records **MUST NOT** be returned to third parties via this endpoint.
- The caller's interpretation of a `200` response is: if `forward_canonical` is non-null, route to the Domain authoritative for the domain of `forward_canonical` using the canonical as the recipient; otherwise route to the Domain authoritative for the domain of `canonical`.

### 8.4 Pending-shadow query

A Registrar **MUST** expose:

```
GET {backend}/api/v1/registrar/pending-shadows?hash={sha256_hex(lowercase(address))}
X-Federation-Sender:    {peer_registrar_domain}
X-Federation-Nonce:     {unique per request}
X-Federation-Signature: ed25519:{base64(signature)}
```

The signature is:

```
signature = Ed25519-sign(
  private_key_of(X-Federation-Sender),
  SHA-256( METHOD || "\n" || PATH_AND_QUERY || "\n" || X-Federation-Nonce )
)
```

This endpoint is authenticated. The Destination **MUST** reject with `401 Unauthorized` (`error = "bad_signature"`) if the signature does not verify under the cached public key for `X-Federation-Sender`, if the sender's peer descriptor does not currently advertise `registrar: true`, or if the nonce has been seen within the last 10 minutes for that sender.

| Status | When                                               | Body                                                                                                                                                             |
|--------|----------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 200    | One or more pending records exist for the hash.    | `[ { "record_id": "...", "message_count": n, "oldest_sent_at": "...", "newest_sent_at": "...", "summary": [ { "federation_id": "...", "subject": "...", "sender_email": "...", "sent_at": "..." } ] } ]` |
| 404    | No pending records.                                | `{"error": "not_found"}`                                                                                                                                         |
| 401    | Signature or sender check failed.                  | `{"error": "bad_signature"}`                                                                                                                                     |
| 429    | Rate limit exceeded.                               | `{"error": "rate_limited", "retry_after_seconds": n}`                                                                                                            |

A "pending record" in this document denotes a Registrar-internal state holding messages for an external address that has not yet been bound to a canonical. The semantics of how such records come into existence or are subsequently consumed are out of scope; this endpoint only exposes their existence to authorized peer Registrars.

### 8.5 Claim by real email

A Registrar **MUST** accept:

```
POST {backend}/api/v1/federation/claim-by-real-email
Content-Type: application/json; charset=utf-8
X-Federation-Version:   0.1
X-Federation-Sender:    {origin_domain}
X-Federation-Nonce:     {unique per request}
X-Federation-Signature: ed25519:{base64(signature)}

{
  "version":    "0.1",
  "claim_id":   "{uuidv7}",
  "token":      "{opaque token previously issued by the Registrar}",
  "real_email": "{external address the claim is for}",
  "canonical":  "{federated address of the claiming user}"
}
```

The signature is computed over the body and nonce per §6.2. The Registrar **MUST**:

1. Verify the signature under the cached public key for `X-Federation-Sender`; reject with `401` (`error = "bad_signature"`) on failure.
2. Verify that the domain of `canonical` equals `X-Federation-Sender`; reject with `401` (`error = "bad_sender"`) on mismatch.
3. Resolve `token` against its internal record store; reject with `400` (`error = "invalid_token"`) if the token is unknown, expired, already consumed, or bound to a different `real_email`.
4. Deduplicate on `(X-Federation-Sender, claim_id)`; a duplicate **MUST** return the original outcome (202 or the prior error) without further side effects.

On success, the Registrar binds `real_email` to `canonical`. Subsequent inbound envelopes containing `real_email` in `recipients` (§8.2) **MUST** be re-dispatched by the Registrar via ordinary inbound transfer (§6) to the Domain authoritative for `canonical`'s domain, with `recipients` rewritten to `[canonical]`. The Registrar, acting as a routing layer only:

- **MUST** preserve the original envelope's `origin_domain`, `message.sender_email`, `message.federation_id`, and `thread.federation_id`.
- **MUST NOT** substitute its own identity for the Origin's in the signature path: a re-dispatched envelope is signed by the original Origin if the Registrar was the first hop, or by the Registrar's own key only when the Registrar is itself the Origin of the forwarded message.
- **MUST** be idempotent on the re-dispatch: the recipient Destination deduplicates on `envelope_id` as usual (§5.4).

| Status | When                                              | Body                                                                 |
|--------|---------------------------------------------------|----------------------------------------------------------------------|
| 200    | Claim accepted (new or idempotent replay).        | `{"claim_id": "...", "status": "accepted"}`                          |
| 400    | Invalid token, malformed body, version mismatch.  | `{"error": "...", "detail": "..."}`                                  |
| 401    | Signature or sender check failed.                 | `{"error": "bad_signature"}` or `{"error": "bad_sender"}`            |
| 429    | Rate limit exceeded.                              | `{"error": "rate_limited", "retry_after_seconds": n}`                |

### 8.6 Trust model for Registrars

This protocol does not define a global Registrar directory. Each Registrar operator curates a local set of peer Registrars whose `/registrar/lookup` responses it consults and whose `/registrar/pending-shadows` queries it will answer. Peer selection is an operator configuration concern and is out of scope for this document.

A Registrar that is not on a Domain's peer list is, for the purposes of §8.3 – §8.5, identical to a non-Registrar Domain: its lookup responses **MUST NOT** be trusted and its pending-shadow queries **MUST** be rejected on signature grounds (its key will not be pinned in §4.3).

Removal of a peer from the configured set is effective immediately. A Domain that has cached lookup responses originating from a removed peer **SHOULD** purge those cache entries.

### 8.7 Interaction with §5.4 and §6

The idempotency rules of §5.4 apply unchanged to envelopes carrying external addresses: dedupe is keyed on `(origin_domain, envelope_id)`, independent of recipient class.

Re-dispatch by a Registrar (§8.5) is a new inbound transfer in its own right; the recipient Destination treats it as it would any other envelope and applies its own §5.4 deduplication. Because `envelope_id` is preserved by the Registrar, multiple re-dispatches produced by retry are collapsed at the final Destination.

## 9. Security considerations

### 9.1 Transport

All requests defined in this document **MUST** be served over TLS 1.2 or later with certificates valid for the domain being addressed. Plaintext HTTP **MUST NOT** be used, including for the well-known endpoint.

### 9.2 Authentication scope

Signatures in Webmail-Federation authenticate **the Domain**, not the end user. A recipient **MUST NOT** infer from a valid envelope signature that the message content was authored by the claimed `sender_email`; they can only infer that the `origin_domain` Domain asserts so. End-user authentication is out of scope.

### 9.3 Replay and clock skew

The `X-Federation-Nonce` and timestamp constraints in §6.2 exist to limit replay windows. Destinations **MUST** keep nonce state for at least 10 minutes per origin domain. The same nonce and replay defenses apply to §8.4 and §8.5 requests.

### 9.4 Key rotation

A Domain **MAY** rotate its Ed25519 keypair by publishing a new `public_key` in its peer descriptor. Destinations observing a signature failure **MUST** re-fetch the descriptor before giving up. During a rotation window a Domain **MAY** sign with the old key while publishing the new one for up to 48 hours; after that window the old key **MUST NOT** be used.

### 9.5 Peer identity and spoofing

A Destination **MUST** reject any envelope where `envelope.origin_domain`, `message.sender_email`'s domain, and `X-Federation-Sender` do not all agree. This prevents one authenticated peer from impersonating another. When a Registrar re-dispatches an envelope on behalf of a non-Registrar Origin (§8.5), the original Origin's domain remains the authoritative `origin_domain` of record; the Registrar's identity, if any, is conveyed transport-layer only.

### 9.6 Registrar trust and misbehavior

A Registrar configured as a peer can influence the routing of external-address envelopes sent through Domains that trust it, by returning crafted `/registrar/lookup` responses. Operators configuring peer Registrars **SHOULD** limit this list to entities whose operational integrity they have independent grounds to trust, and **SHOULD** monitor for evidence of misrouting (e.g. sustained mismatch between re-dispatch targets and recipients' expected domains). There is no protocol-level quorum or revocation mechanism; defense is curation and removal.

### 9.7 Abuse, rate limiting, and content

This document does not specify anti-abuse rules. A Destination Domain **SHOULD** apply per-peer rate limits, per-peer daily quotas, and content-based filters before accepting envelopes for delivery, and **MAY** refuse traffic from peers at its discretion using `403 Forbidden`. Registrars **SHOULD** additionally rate-limit `/registrar/lookup` per source IP or per sender domain to mitigate enumeration of their internal mapping.

### 9.8 Privacy

The peer descriptor (§4.2) is public. Domains **MUST NOT** include user data, account counts, or internal topology in it.

Registrar lookup responses (§8.3) reveal, for a given external-address hash, that a binding exists and to which Federation-enabled canonical. Because the lookup is keyed by a hash of the address itself, querying requires prior knowledge of the address; the endpoint does not enable enumeration of bindings. However, the mapping from an external address to a canonical may reveal that the holder of the external address is a user on a specific federated domain. Operators who consider this observable a privacy concern **MAY** decline to operate a Registrar role.

## 10. Versioning and evolution

A Domain **MUST** reject envelopes whose `version` is not listed in its advertised `versions`. Future versions of this protocol **MAY** add optional fields; unknown fields in an otherwise valid envelope **MUST** be ignored by the Destination. Breaking changes **MUST** be expressed as a new `version` string, and both peers **MUST** negotiate the highest version they both support.

## 11. IANA considerations

This document requests no IANA actions. The well-known path `/.well-known/webmail-federation` is reserved for the purposes described herein and is not at present registered with IANA.

## 12. References

- RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
- RFC 8174 — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.
- RFC 3339 — Date and Time on the Internet: Timestamps.
- RFC 8032 — Edwards-Curve Digital Signature Algorithm (EdDSA).
- RFC 8615 — Well-Known Uniform Resource Identifiers.
- RFC 9562 — Universally Unique IDentifiers (UUIDs), including UUIDv7.

## Appendix A. Example exchange

### A.1 Origin-side send

Alice at `103mail.com` composes a message to `bob@example.com` and `carol@example.com`.

The Origin Domain splits fan-out by domain and, for `example.com`, fetches (or reuses cached) `https://example.com/.well-known/webmail-federation`, then issues:

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

Carol replies. Her Origin Domain (example.com) issues a new envelope back to `103mail.com` with `thread.federation_id` set to the value Carol received in A.1 and `thread.in_reply_to` equal to the message's `federation_id`. 103mail matches on `federation_id` and appends to the existing `g_thread`.
