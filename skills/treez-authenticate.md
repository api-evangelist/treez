---
name: treez-authenticate
description: Mint the self-signed RSA JWT every Treez v3 API call requires, and verify which organizations, dispensaries and endpoints your certificate may reach.
api: openapi/treez-jwt-check-openapi.json
generated: '2026-08-30'
method: generated
source: https://code.treez.io/reference/authentication
operations:
  - post_jwt-validation
---

# Authenticate to the Treez v3 APIs

Treez does **not** issue bearer tokens on v3. Every single request carries a freshly minted,
self-signed JWT that lives for 30 seconds. There is no token endpoint to call and nothing to
cache — the signing happens in your process, on every call.

**Read this before you write the signer:** the token is *two* parts, not three. Treez expects
`base64url(claims) + "." + base64(RSA-SHA256 signature)`. A standard JWT library will emit the
three-part compact JWS (`header.payload.signature`) and Treez will reject it. Build the token by
hand, exactly as below.

## Prerequisites

1. An RSA-4096 key pair:
   `openssl req -new -newkey rsa:4096 -x509 -sha256 -days 1825 -noenc -out public.crt -keyout private.key`
2. The public `.crt` emailed to `api-support@treez.io`.
3. The **Certificate ID** Treez returns. It is bound to a record listing the organizations,
   dispensaries and endpoints you are entitled to call.
4. The **Organization ID** (GUID) of the org you are calling into.

Keep `private.key` out of source control and out of any prompt or tool argument.

## Steps

1. **Build the claims object.**

   | claim | value |
   |---|---|
   | `aud` | the exact endpoint URL you are about to call — not the host, the full URL |
   | `iss` | your Certificate ID |
   | `oid` | the target Organization ID |
   | `iat` | now, in **milliseconds** since epoch |
   | `exp` | `iat + 30000` — exactly 30,000 ms |
   | `jti` | a fresh UUID (recommended, prevents replay) |

   `iat`/`exp` are milliseconds, not the seconds RFC 7519 specifies. A TTL outside 30,000 ms
   returns **400**, not 401.

2. **base64url-encode the claims JSON** (`+`→`-`, `/`→`_`, strip `=` padding).

3. **Sign that encoded string** with your private key using `SHA256withRSA`, then base64 the
   signature.

4. **Send** `Authorization: <encodedClaims>.<signature>` with `Content-Type: application/json`.

5. **Verify your entitlements** — call `post_jwt-validation`
   (`POST https://api-prod.treez.io/service/jwt-validation`) with the token you just built. It
   confirms the signature is accepted and returns the resources the certificate can reach. Do this
   first, once, when onboarding a new store; do not call it before every request.

## Conventions and failure modes

- `401` with `{"resultCode":"FAIL","resultReason":"Full authentication is required to access this resource"}`
  means the token was missing, malformed, or signed with the wrong key. Check `aud` matches the
  URL character for character.
- `400` on an otherwise valid call usually means the TTL is wrong, or `iat`/`exp` were sent in
  seconds.
- **Sign per request.** A token minted for one URL will not authenticate another — `aud` is the
  endpoint.
- There is no rate limit published on signing or on the APIs; see
  `rate-limits/treez-rate-limits.yml`. On v3 there is no auth endpoint to over-call.
- The legacy v2 surface (`api.treez.io/v2.0/...`) uses a completely different scheme — a client ID
  plus per-location API key exchanged for a 2-hour token. Do not mix them. Treez requires certified
  partners to reuse that v2 token for its full life.

Full detail: `authentication/treez-authentication.yml`.
