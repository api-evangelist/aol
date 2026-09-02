---
name: revoke-aol-access
description: >-
  Reverse an AOL token grant — revoke an issued access or refresh token via
  RFC 7009, and understand what revocation does and does not undo.
api: AOL OAuth2 API
operations:
  - getToken
generated: '2026-09-02'
method: generated
source: https://api.login.aol.com/.well-known/openid-configuration
---

# Revoke AOL access

This is the reversal skill for `sign-in-with-aol`. Read it BEFORE you issue a
token, not after — it tells you what can be taken back.

## The reversal path

AOL's own discovery document declares a token revocation endpoint:

`token_revocation_endpoint: https://api.login.aol.com/oauth2/revoke`

That is RFC 7009. It is the reversal for the one consequential write this API
has — token issuance at `getToken`.

Call it with your client credentials (`client_secret_basic` or
`client_secret_post`, the only two methods AOL supports) and the token to
revoke, form-encoded. Revoking an already-revoked token is a no-op by RFC 7009,
so this operation is safe to retry.

## What is NOT stated, and why that matters

AOL publishes **no window** for revocation: no statement of propagation delay,
no grace period, no guarantee about how long an already-issued access token
remains accepted by resource endpoints after you revoke it. Do not assume the
revocation is instantaneous across every AOL surface, and do not assume it is
delayed either — AOL has not said. If your use case depends on an immediate
hard cut-off, verify it yourself against your own client.

This is why `conventions/aol-conventions.yml` grades AOL's reversibility as
`documented` rather than `verified`: the path exists and is machine-discoverable,
but the window is not published.

## What revocation does NOT undo

- **The user's consent grant.** Revoking a token does not un-grant the
  application's authorization. Only the end user can do that, by hand, at
  https://myaccount.aol.com/. There is no API operation for it — an agent cannot
  withdraw consent programmatically.
- **Itself.** Revocation is terminal. To regain access you must run the full
  Authorization Code flow again from `requestAuth`.
- **Anything you already read.** Profile claims your client already fetched from
  `getUserInfo` are yours to delete under your own retention policy.

## Checking a token's state first

`introspection_endpoint: https://api.login.aol.com/oauth2/introspect` (RFC 7662)
is declared in the discovery document. It refuses anonymous callers with a bare
HTML `403` — client authentication is required, and the response shape is not
verifiable from public probing. Treat introspection as declared-but-unverified.

## Note on this repository's specification

The revocation and introspection endpoints are present in AOL's discovery
document but are **not** in `openapi/aol-oauth2-api-openapi.yml`. That is a gap
in our specification, not in AOL's surface. Trust the discovery document.
