---
name: webmail-federation project overview
description: What the webmail-federation repo is for and its two sibling repos (webmail backend, webmail-client frontend)
type: project
originSessionId: 5d5787c3-6720-4400-8326-1aced9a9e8fb
---
webmail-federation is a greenfield project (only .git as of 2026-04-18) meant to design and coordinate federation between multiple deployments of the `webmail` backend, so domains like 103mail.com, example.com, etc. can share one client bundle while still exchanging mail.

**Why:** The existing `webmail` PHP backend at /home/maurits/projects/webmail is hardcoded to a single default domain (103mail.com) and `webmail-client` at /home/maurits/projects/webmail-client has 103mail branding baked in. Goal is multi-tenant: one client, per-domain backends, federated delivery.

**How to apply:** When the user asks about federation, multi-domain, or "103mail and other domains", think in terms of the trio: webmail (PHP backend, per-domain instance), webmail-client (React/MUI, one bundle), webmail-federation (design + glue). Don't treat the three repos in isolation.

**Key architectural facts to remember:**
- webmail does NOT send SMTP. It writes g_thread/g_message once and fans out per-subscriber thread/message rows in its own DB. Only a metadata-only notification (sent_email table) goes outside.
- Schema already has a `domain` table but code treats it as singleton via `EmailClient::getDefaultDomain()`.
- Unknown recipients get auto-onboarded via `addCustomer()` with a random 7-digit @103mail.com alias — this shortcut is wrong under federation and must split into local-vs-remote branches.
- webmail-client is currently mock-only; no real API client yet, so federation-aware config can be introduced cleanly.
