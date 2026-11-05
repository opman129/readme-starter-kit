---
title: Quickstart
deprecated: false
hidden: false
metadata:
  robots: index
---
Make your first successful API call in about five minutes. By the end you'll
have fetched an access token, created a customer, and charged a test card — all
against the sandbox, so no real money moves.

## Before you start

You need:

- A sandbox account and your test credentials (`client_id` and a
  `client_secret` that starts with `sk_test_`). Both are on the **Developers →
  API keys** page of the dashboard.
- A tool to make HTTP requests — `curl`, the examples below, or the "Try it"
  playground on this page.

All sandbox requests go to:

```
https://sandbox.example-pay.com/v1
```

Amounts are always an integer in the currency's **minor unit**. `2000` with
currency `USD` means $20.00.

## 1. Get an access token

Exchange your credentials for a short-lived bearer token.

```bash
curl https://sandbox.example-pay.com/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "YOUR_CLIENT_ID",
    "client_secret": "sk_test_YOUR_SECRET",
    "scope": "payments.write customers.write"
  }'
```

The response contains the token and how long it lasts (one hour):

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.payload.sig",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "payments.write customers.write"
}
```

Copy the `access_token`. You'll send it on every other request. Full details,
including scopes and token lifetime, are in the [Authentication](/docs/authentication)
guide.

## 2. Make your first call: create a customer

Send the token as a bearer token in the `Authorization` header.

```bash
curl https://sandbox.example-pay.com/v1/customers \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "ada@example.com",
    "name": "Ada Lovelace",
    "address": { "line1": "12 Marina Road", "city": "Lagos", "country": "NG" }
  }'
```

A `201 Created` response returns the new customer and its `id`:

```json
{
  "id": "cus_8sJk2Pq1mN",
  "object": "customer",
  "email": "ada@example.com",
  "name": "Ada Lovelace",
  "livemode": false,
  "created": "2026-11-05T09:30:00Z"
}
```

## 3. Charge a test card

Create a payment for the customer using the test card `4242 4242 4242 4242`,
which always succeeds in the sandbox.

```bash
curl https://sandbox.example-pay.com/v1/payments \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": { "value": 2000, "currency": "USD" },
    "customer_id": "cus_8sJk2Pq1mN",
    "payment_method": {
      "type": "card",
      "card": { "number": "4242424242424242", "exp_month": 12, "exp_year": 2030, "cvc": "123" }
    },
    "description": "Order #1138"
  }'
```

The payment comes back `succeeded`:

```json
{
  "id": "pay_3Nf2kL9aZ",
  "object": "payment",
  "amount": { "value": 2000, "currency": "USD" },
  "status": "succeeded",
  "customer_id": "cus_8sJk2Pq1mN",
  "description": "Order #1138",
  "capture_method": "automatic",
  "livemode": false,
  "created": "2026-11-05T09:31:12Z"
}
```

That's a full charge, start to finish.

## Test cards

| Card number         | Behaviour                       |
| ------------------- | ------------------------------- |
| 4242 4242 4242 4242 | Succeeds                        |
| 4000 0000 0000 0002 | Declined (`card_declined`)      |
| 4000 0000 0000 9995 | Declined (`insufficient_funds`) |

## Next steps

- Authorize now and capture later with `capture_method: manual` — see the
  payments reference.
- Return funds with the refunds endpoints.
- Read the [Authentication](/docs/authentication) guide for scopes, API keys,
  and token lifetime.
- Read [Errors and status codes](/docs/errors) before you ship, so your
  integration handles declines and rate limits cleanly.
