---
title: Errors and status codes
deprecated: false
hidden: false
metadata:
  robots: index
---
The API uses conventional HTTP status codes and returns a consistent JSON error
object, so you can branch on both the status code and a stable machine-readable
`code`.

## How errors are returned

Any failed request returns a non-2xx status code and a body with a single
`error` object:

```json
{
  "error": {
    "type": "card_error",
    "code": "card_declined",
    "message": "The card was declined.",
    "param": "payment_method",
    "doc_url": "https://docs.example-pay.com/errors/card_declined"
  }
}
```

| Field     | Description                                                                   |
| --------- | ----------------------------------------------------------------------------- |
| `type`    | The broad category of error (see the table below).                            |
| `code`    | A stable, machine-readable code for the specific error. Branch on this.       |
| `message` | A human-readable explanation. Safe to log; don't show the raw text to buyers. |
| `param`   | The request field the error relates to, when applicable.                      |
| `doc_url` | A link to documentation for this specific error.                              |

Branch on `type` and `code`, not on `message` — message wording can change.

## HTTP status codes

| Status                     | Meaning                                                              |
| -------------------------- | -------------------------------------------------------------------- |
| `200 OK`                   | The request succeeded.                                               |
| `201 Created`              | A resource was created.                                              |
| `204 No Content`           | The request succeeded and there's nothing to return (e.g. a delete). |
| `400 Bad Request`          | The request was malformed or missing a required field.               |
| `401 Unauthorized`         | Authentication failed, or the token was missing or expired.          |
| `402 Payment Required`     | The charge was declined by the issuer.                               |
| `403 Forbidden`            | The token's scopes don't cover this action.                          |
| `404 Not Found`            | No resource exists with the given identifier.                        |
| `409 Conflict`             | The request conflicts with the resource's current state.             |
| `422 Unprocessable Entity` | The request was well-formed but failed validation.                   |
| `429 Too Many Requests`    | The rate limit was exceeded.                                         |
| `500 Service Error`        | Something went wrong on our side. Safe to retry.                     |

## Error types

| `type`                  | When you'll see it                               |
| ----------------------- | ------------------------------------------------ |
| `invalid_request_error` | Missing parameters, bad IDs, or state conflicts. |
| `authentication_error`  | Missing, invalid, or expired credentials.        |
| `card_error`            | The card was declined or is otherwise unusable.  |
| `validation_error`      | A field failed a validation rule.                |
| `rate_limit_error`      | Too many requests in a short window.             |
| `api_error`             | An unexpected error on our side.                 |

## Common error codes

| `code`                   | Status | What to do                                                        |
| ------------------------ | ------ | ----------------------------------------------------------------- |
| `parameter_missing`      | 400    | Add the required field named in `param`.                          |
| `token_expired`          | 401    | Fetch a new access token and retry.                               |
| `card_declined`          | 402    | Ask the buyer for another payment method.                         |
| `insufficient_funds`     | 402    | Ask the buyer for another card.                                   |
| `resource_missing`       | 404    | Check the ID; the object may be in the other mode (test vs live). |
| `payment_not_capturable` | 409    | The payment isn't in `requires_capture`; check its status first.  |
| `amount_too_small`       | 422    | Raise the amount above the minimum for the currency.              |
| `rate_limited`           | 429    | Back off and retry after `Retry-After` seconds.                   |

## Handling rate limits

A `429` response includes a `Retry-After` header with the number of seconds to
wait:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 2
```

Respect that delay and retry with exponential backoff. Don't retry tight in a
loop — that extends the limit.

## Safe retries with idempotency

Network errors and `500`s can leave you unsure whether a create succeeded. Send
an `Idempotency-Key` header on `POST` requests that create money movement
(payments and refunds). Reusing the same key replays the original result instead
of creating a duplicate.

```bash
curl https://sandbox.example-pay.com/v1/payments \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Idempotency-Key: order-1138-attempt-1" \
  -H "Content-Type: application/json" \
  -d '{ "amount": { "value": 2000, "currency": "USD" }, "customer_id": "cus_8sJk2Pq1mN", "payment_method_id": "pm_1Ld9xBke72" }'
```

Generate one key per logical operation (for example, per checkout attempt) and
reuse it across retries of that same operation.

## Next steps

- [Quickstart](/docs/quickstart) — make your first call.
- [Authentication](/docs/authentication) — tokens, scopes, and keys.