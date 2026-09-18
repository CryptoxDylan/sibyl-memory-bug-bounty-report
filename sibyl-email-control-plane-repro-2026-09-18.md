# Sibyl Memory — Email Control-Plane Repro

**Date:** 2026-09-18  
**Tester:** Dylan (`@CryptoxDylan`) via Grok  
**Target:** Sibyl Memory Plugin control plane (`api.sibyllabs.org/api/plugin/*`)  
**Email used:** `cryptoxdylan@gmail.com` (owned). No other addresses were bound.  
**Verdict:** **REPRODUCED**, then test devices revoked.

---

## What we did

Confirm the email login ticket, owned address only:

1. Bind the owned email from **session A** → save `account_id` + token.
2. Bind the **same** email from **session B** → expect the same `account_id` and a second live token.
3. Prove the token works on `/access`, `/devices`, `/check-write`. Wrong `account_id` → 403.
4. Stop. Do not bind anyone else’s address.
5. Revoke the two test devices/sessions.

Email bind is **not** inbox OTP. The CLI prints a 6-digit pairing code; the browser posts that code with the email string. No mailbox access is required.

### Protocol

```
POST /api/plugin/session-init
  { session, pairing_code_hash: sha256("{code}:{session}"), env }

POST /api/plugin/email-bind
  { session, email, pairing_code }

GET  /api/plugin/check?session=<uuid>
  → credentials { account_id, session_token, bearer_token, email, tier, … }

POST /api/plugin/access
  { account_id, session_token }   (+ Authorization: Bearer <token>)

GET  /api/plugin/devices?account_id=<uuid>
  Authorization: Bearer <token>

POST /api/plugin/check-write
  { account_id, session_token, current_size_bytes, proposed_delta_bytes }
  Authorization: Bearer <token>

POST /api/plugin/devices
  { bearer_id, account_id }       # account_id is required; CLI 0.4.1 omits it
  Authorization: Bearer <token>
```

Observed on this run: `session_token` == `bearer_token` == the pairing **session UUID**. Device rows use a **different** `bearer_id` UUID.

---

## What we found

### 1. Ticket match — same email, two live sessions

| Step | Result |
|---|---|
| Session A email-bind | HTTP 200. `account_id=b6d6010a-fbcd-4989-89a8-2eee3f08ab0d`, tier `free` |
| Session B email-bind (same email) | HTTP 200. **Same** `account_id`. **Second** live token |
| Both tokens → `POST /access` | **200** (`ok`, `session_verified: true`) |
| Both tokens → `GET /devices` | **200**, **2** devices, both `browser-email-pairing` / `email-pairing`. Each token sees itself as `is_this_device` |
| Both tokens → `POST /check-write` | **200** (`ok`, `tier: free`) |
| Wrong `account_id` (`00000000-0000-4000-8000-000000000000`) | **403** on `/access`, `/devices`, `/check-write` — `session does not match the requested account_id` |

This is the ticket. **Reproduced. Stopped. No other addresses.**

### 2. Email is an account key, not a proof of ownership

The email path is:

1. Client invents a pairing session + 6-digit code locally.
2. Browser `POST /email-bind` with `{session, email, pairing_code}`.
3. Server keys (or joins) the account on the **email string** and issues a bearer.

There is no magic link, no OTP to the inbox, no domain verification. The pairing code authenticates the **terminal**, not the mailbox.

Consequence: same email → same `account_id` + a new live token. That is multi-device login **without proving the caller owns the mailbox**. Wrong-account checks work (403). Email identity does not.

We did **not** bind any address we do not own. Cross-account takeover via a third-party email was not tested.

### 3. Device revoke ≠ session kill

Published `sibyl-memory-cli` **0.4.1** (`sibyl devices revoke N`) posts only `{bearer_id}`. Live server requires `account_id` in the body:

```
revoke failed: 400 {'error': 'account_id=<uuid> required'}
```

`sibyl logout` reuses that same POST, so it also failed to revoke remotely (`remote session may still be active`) and only deleted local `credentials.json`.

Working call (same endpoint the CLI intends):

```
POST /api/plugin/devices
Authorization: Bearer <session_token>
{ "bearer_id": "<device bearer_id>", "account_id": "<uuid>" }
```

With that body:

| Action | Server |
|---|---|
| Revoke session B device | `{ok: true, revoked: true}` |
| Self-revoke session A device | `{ok: true, revoked: true}` |
| `GET /devices` afterward | **0** active devices |
| Re-revoke either `bearer_id` | **404** `bearer not found, already revoked, or not yours` |

Device rows are gone. **Session tokens are not:**

| Endpoint after device revoke | Result |
|---|---|
| `POST /access` (A and B) | **200**, `session_verified: true` |
| `POST /check-write` (A and B) | **200**, `ok: true`, `tier: free` |
| `GET /devices` (A and B) | **200**, empty list (auth still accepted) |

No public session-kill route showed up (`/logout`, `/session-revoke`, `/revoke`, `DELETE /session` → 404). The pairing-session UUID remains a live credential for `/access` and `/check-write` after the device row is deleted.

Local cleanup: `~/.sibyl-memory/credentials.json` removed. Test tokens were not written back.

### 4. Related CLI notes (not the ticket)

- `sibyl` 0.4.1 does not import on Python 3.10/3.11 (nested quotes in f-strings; needs 3.12+).
- Logout caveat is accurate: local file gone, remote bearer/session may still be live — and in this case `/access` still verified.

---

## Control-plane summary

```
email string  ──(pairing code from attacker-controlled terminal)──►  account_id
account_id + session UUID  ──►  /access, /devices, /check-write
device bearer_id revoke    ──►  device row gone; session UUID still valid
wrong account_id           ──►  403 (this check holds)
mailbox ownership          ──►  never checked
```

**In scope / confirmed**

- Same owned email, two sessions → one `account_id`, two live tokens.
- Tokens authorize `/access`, `/devices`, `/check-write`.
- Mismatched `account_id` → 403.
- `sibyl devices revoke` as shipped does not revoke (missing `account_id`).
- Successful device revoke does not invalidate the session token used by `/access` and `/check-write`.

**Out of scope (not tested, per ticket)**

- Binding any email we do not own.
- Wallet / SIWE path.
- Whether a second binder can attach to an existing **paid** account.

---

## Suggested hardening (not implemented here)

1. Prove mailbox ownership (OTP or magic link) before treating email as an account key — or stop using email as a join key without that proof.
2. Issue a bearer distinct from the pairing session id (SEC-1). Invalidate it on `devices` revoke **and** on logout.
3. Make `/access` and `/check-write` fail closed if the device row is revoked or missing.
4. Fix CLI revoke/logout body: send `account_id` with `bearer_id`.
5. Add an explicit session-revoke API and wire `sibyl logout` to it.

---

## Artifacts

- Account under test: `` (free, email-bound).
- Test device labels: `browser-email-pairing` × 2, both revoked from the device list.
- Tokens redacted; not persisted after logout.
