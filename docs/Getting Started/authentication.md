---
title: Authentication
deprecated: false
hidden: false
metadata:
  robots: index
---
Every request to the Acme Payments API — except the token endpoint itself —
must be authenticated, either with a short-lived bearer token or, for trusted
server-to-server jobs, an API key. This guide covers how to get a token, how to
use it, how scopes work, and how to keep your credentials safe.

## How it works

Authentication uses the OAuth 2.0 **client-credentials** grant:

1. You hold a `client_id` and a `client_secret` (your API keys).
2. You exchange them at `POST /auth/token` for an `access_token`.
3. You send that token as a bearer token on every other request.
4. The token expires after one hour; when it does, you request a new one.

Your `client_secret` is the sensitive half. Treat it like a password: keep it
server-side, never commit it, and never put it in client-side code or a URL.

## Get a token

Send your credentials to the token endpoint. Request only the scopes you need.

```bash
curl https://sandbox.example-pay.com/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "sk_test_YOUR_SECRET",
    "scope": "payments.write customers.read"
  }'
```

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.payload.sig",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "payments.write customers.read"
}
```

## Use the token

Put the token in the `Authorization` header, prefixed with `Bearer`:

```bash
curl https://sandbox.example-pay.com/v1/payments \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

A missing, malformed, or expired token returns `401 Unauthorized`:

```json
{
  "error": {
    "type": "authentication_error",
    "code": "token_expired",
    "message": "The access token has expired.",
    "doc_url": "https://docs.example-pay.com/errors/token_expired"
  }
}
```

## Token lifetime

Tokens last **3600 seconds (one hour)**, as given by `expires_in`. There is no
refresh token — when a token expires, request a new one from `/auth/token` with
the same credentials. In practice, cache a token and reuse it until shortly
before it expires, rather than fetching one per request.

## Scopes

Request the narrowest set of scopes your integration needs. A token can only
perform actions covered by its scopes; anything else returns `403`.

| Scope             | Grants                                          |
| ----------------- | ----------------------------------------------- |
| `payments.read`   | Read payments and refunds                       |
| `payments.write`  | Create and capture payments, create refunds     |
| `customers.read`  | Read customers and payment methods              |
| `customers.write` | Create and update customers and payment methods |

## Test vs. live keys

Your account has two key pairs:

- **Test keys** (`sk_test_…`) run against the sandbox. No real money moves, and
  responses have `"livemode": false`.
- **Live keys** (`sk_live_…`) run against production.

Keep them separate per environment. A common setup is to read the key from an
environment variable so the same code runs in both.

## API keys for server-to-server calls

For simple back-office scripts you can authenticate with a secret key directly,
sent in the `X-API-Key` header, instead of fetching a token:

```bash
curl https://sandbox.example-pay.com/v1/customers \
  -H "X-API-Key: sk_test_YOUR_SECRET"
```

Bearer tokens are preferred for application traffic because they're short-lived
and scoped; reserve raw API-key auth for trusted internal jobs.

## Keep credentials safe

- Store secrets in a secrets manager or environment variables, never in source
  control.
- Rotate keys if one may have leaked; you can roll keys from the dashboard.
- Scope tokens tightly and set short lifetimes.
- Never send a secret in a query string or expose it in browser code.

## Next steps

- [Quickstart](/docs/quickstart) — your first authenticated call.
- [Errors and status codes](/docs/errors) — how auth failures and other errors
  are reported.
