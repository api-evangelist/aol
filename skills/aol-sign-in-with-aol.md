---
name: sign-in-with-aol
description: >-
  Sign a user in with their AOL account using the OAuth 2.0 Authorization Code
  flow against AOL's OpenID Connect provider, and read their profile claims.
api: AOL OAuth2 API + AOL OpenID Connect API
operations:
  - requestAuth
  - getToken
  - getUserInfo
generated: '2026-09-02'
method: generated
source: >-
  openapi/aol-oauth2-api-openapi.yml, openapi/aol-openid-connect-api-openapi.yml,
  https://api.login.aol.com/.well-known/openid-configuration
---

# Sign in with AOL

AOL runs its own OpenID Connect provider. The issuer is
`https://api.login.aol.com` — confirmed by its discovery document at
`https://api.login.aol.com/.well-known/openid-configuration`. Only the
Authorization Code grant is supported; there is no client-credentials or device
flow.

Before you start you need a Consumer Key and Consumer Secret from an app
registration, and a registered `redirect_uri`. AOL runs no developer portal of
its own; registration for this identity stack lives on the legacy Yahoo
developer site (https://developer.yahoo.com/oauth2/guide/openid_connect/), which
documents the sibling deployment of the same platform.

## Step 1 — send the user to the authorization endpoint (`requestAuth`)

`GET https://api.login.aol.com/oauth2/request_auth`

Required query parameters: `client_id`, `redirect_uri`, `response_type=code`.
Send `scope=openid profile email` to get an ID token plus profile and email
claims. Always send a random `state` and compare it on the way back — it is the
only CSRF control this flow gives you, and AOL round-trips it unchanged.

Optional: `language` (defaults to `en-us`).

If you need proof of multi-factor authentication, request `AAL2` — AOL
advertises `acr_values_supported: [AAL1, AAL2]` (NIST SP 800-63B). Note that the
OIDC `claims` request parameter is NOT supported
(`claims_parameter_supported: false`), and neither are request objects
(`request_parameter_supported: false`), so you cannot ask for individual claims —
you widen the claim set with scopes instead.

AOL responds with a 302 back to your `redirect_uri` carrying `code` and `state`,
or `error` on failure.

## Step 2 — exchange the code for tokens (`getToken`)

`POST https://api.login.aol.com/oauth2/get_token`
with `Content-Type: application/x-www-form-urlencoded`.

Body: `client_id`, `client_secret`, `redirect_uri` (the same one), `grant_type=authorization_code`, `code`.

Client credentials may go in the body (`client_secret_post`) or in an
`Authorization: Basic` header holding base64 `client_id:client_secret`
(`client_secret_basic`). Those two are the only methods AOL supports.

The 200 response carries `access_token`, `token_type: bearer`, `expires_in`,
`refresh_token`, `id_token` (when `openid` was requested) and
`xoauth_yahoo_guid` — a vendor-specific stable user id carried over from the
Oath-era platform. It is not an OIDC standard claim; prefer `sub`.

There is no idempotency key on this endpoint and none is needed: an authorization
code is single-use by RFC 6749. Do not retry a code exchange after a 400.

Failures: `400` for a malformed request, `401` for bad client authentication.
Both return the RFC 6749 envelope `{"error", "error_description"}` — not
problem+json.

## Step 3 — verify the ID token against the live JWKS (`getJwks`)

Never trust an ID token you have not verified. Read the `kid` from the JWT
header, fetch `https://api.login.aol.com/openid/v1/certs` (anonymous, no token
needed), select the matching key, and verify the signature. AOL signs with
`ES256` or `RS256`. Then check `iss == "https://api.login.aol.com"`,
`aud == your client_id`, and `exp`.

Cache the key set and re-fetch on an unknown `kid` rather than on every request.

## Step 4 — read profile claims (`getUserInfo`)

`GET https://api.login.aol.com/openid/v1/userinfo`
with `Authorization: Bearer <access_token>`.

Returns `sub`, `name`, `given_name`, `family_name`, `email`, `email_verified`,
`locale`, `profile_images`, and — per AOL's `claims_supported` — `birthdate` and
`auth_time`. A `401` means the token is missing, expired, or was not issued with
the `openid` scope.

## Refreshing

`POST https://api.login.aol.com/oauth2/get_token` with
`grant_type=refresh_token` and `refresh_token`, plus your client credentials.

## What to expect when things go wrong

The identity host answers non-OAuth paths with a proprietary envelope —
`{"error":{"localizedMessage","errorId","message"}}` — and answers a refused
anonymous introspection call with a bare HTML `403`. Do not assume every error
on this host is JSON. There is no request-id or correlation header to cite, and
no published rate limits or `RateLimit-*` headers. See
`errors/aol-problem-types.yml` and `conventions/aol-conventions.yml`.
